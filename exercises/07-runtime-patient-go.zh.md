# 练习 7: Patient Go - 让 Go 等待所有 Goroutine

> 📖 **想深入了解？** 阅读 Internals for Interns 上的 [The Bootstrap](https://internals-for-interns.com/posts/understanding-go-runtime/) 和 [The Scheduler](https://internals-for-interns.com/posts/go-runtime-scheduler/)，深入理解 Go runtime 启动与 goroutine 调度。

本练习中，你将修改 Go runtime，让程序在退出前等待所有 goroutine 结束。目前当 `main()` 返回时，即使还有 goroutine 在跑，Go 也会立刻终止。我们要让 Go 更「有耐心」，等所有 goroutine 完成再退出。

## 学习目标

完成本练习后，你将能够：

- 理解 Go 程序的终止流程
- 知道如何统计活跃 goroutine 数量
- 修改 runtime 的 main 函数以改变程序行为
- 理解「自动等待 goroutine」带来的取舍

## 引言：Go 程序是如何启动的？

Go 二进制并不是从你的 `main()` 开始跑的。操作系统会先执行架构相关的汇编入口（如 `_rt0_amd64_linux`），搭建最基础的 runtime 基础设施。这个 bootstrap 过程会初始化 Go runtime 管理执行所用的三个核心抽象：

- **G (Goroutine)**：轻量级执行单元，自带栈（初始约 2KB）。每条 `go` 语句都会创建一个新的 G。
- **M (Machine)**：真正执行 goroutine 的 OS 线程。每个 M 都有一个特殊的 `g0` goroutine，用于 runtime 管理任务。
- **P (Processor)**：把 goroutine 接到线程上的逻辑调度上下文。每个 P 有自己的可运行 goroutine 队列和内存缓存。P 的数量由 `GOMAXPROCS` 控制。

Bootstrap 大致顺序是：汇编初始化 → scheduler 初始化（栈池、内存分配器、垃圾回收器）→ `runtime.main()` → 启动系统监控线程 → 启用 GC → 运行各 package 的 `init()` → 最后调用你的 `main.main()`。

当你的 `main()` 返回后，控制权回到 `runtime.main()`，继续做 tear-down。这正是我们要动手改的地方。

## 背景：Go 当前的终止行为

目前如果你这样写：

```go
package main

import "time"

func main() {
    go func() {
        time.Sleep(2 * time.Second)
        println("Goroutine finished!")
    }()
    println("Main finished!")
    // Program exits immediately, goroutine never completes
}
```

**输出：**
```
Main finished!
```

因为 `main()` 一返回程序就退出了，goroutine 根本来不及打印。

我们要改成：Go 会耐心等所有 goroutine 跑完：

**新的输出：**
```
Main finished!
Goroutine finished!
```

## 第 1 步：理解 Runtime 的 Main 函数

Go runtime 在 `runtime/proc.go` 里的 `main()` 负责调用你程序的 `main()`。先看看它是怎么串起来的：

```bash
cd go/src/runtime
```

打开 `proc.go`，找到 `main()` 函数。靠近开头（大约第 136–137 行），可以看到 runtime 如何链接到你程序的 main：

```go
//go:linkname main_main main.main
func main_main()
```

这个 `//go:linkname` 指令告诉链接器：把 runtime 的 `main_main` 连到你程序的 `main.main`。这样 runtime 才能调用 main package 里的代码。

再往下看同一个 `main()` 函数（大约第 289 行），会看到真正调用的地方：

```go
fn := main_main // make an indirect call, as the linker doesn't know the address of the main package when laying down the runtime
fn()

... // tear-down process continues
```

**执行流程：**

1. Go runtime 完成 bootstrap
2. runtime 的 `main()` 先跑起来
3. 再做一些 bootstrap 收尾
4. 调用 `main_main`（通过 linkname 映射到你程序的 `main()`）
5. 你的 `main()` 执行——**责任交到你的代码手里**
6. 你的 `main()` 返回后，控制权回到 runtime 的 `main()`
7. runtime 继续做程序的 **tear-down**（清理并退出）

目前 tear-down 会在你的 `main()` 返回后立刻开始，不会等其他 goroutine。

## 第 2 步：加入等待 Goroutine 的逻辑

我们要加一段逻辑：一直等到只剩 1 个 goroutine（也就是 main goroutine 自己）。

**编辑 `runtime/proc.go`：**

找到大约第 289–290 行调用 `main_main` 的位置：

```go
fn := main_main // make an indirect call, as the linker doesn't know the address of the main package when laying down the runtime
fn()
```

在 `fn()` 调用之后立刻加上等待逻辑：

```go
fn := main_main // make an indirect call, as the linker doesn't know the address of the main package when laying down the runtime
fn()

// Wait until only 1 goroutine is running (the main goroutine)
for gcount(false) > 1 {
	Gosched()
}
```

### 理解这段代码

- **`gcount(false)`** - runtime 函数，返回活跃 goroutine 数量（`false` 表示不把系统 goroutine 算进去）
- **`gcount(false) > 1`** - 只要还有比 main 更多的 goroutine 在跑
- **`Gosched()`** - 让出处理器，让其他 goroutine 有机会运行
- **循环结束条件** - 只剩 main goroutine 时（count = 1）

## 第 3 步：重新构建 Go Toolchain

```bash
cd go/src
./make.bash
```

这会用你「耐心等待」的逻辑重新构建 runtime。

## 第 4 步：测试基础的 Goroutine 等待

创建一个测试文件验证行为：

创建 `patient_demo.go`：

```go
package main

import "time"

func main() {
	println("Main starting...")

	go func() {
		time.Sleep(1 * time.Second)
		println("Goroutine 1 finished!")
	}()

	go func() {
		time.Sleep(2 * time.Second)
		println("Goroutine 2 finished!")
	}()

	println("Main finished, but Go will wait...")
}
```

用你改过的 Go 运行：

```bash
./bin/go run patient_demo.go
```

**期望输出：**

```
Main starting...
Main finished, but Go will wait...
Goroutine 1 finished!
Goroutine 2 finished!
```

成功！Go 现在会等所有 goroutine 完成。

## 我们学到了什么

- **程序终止**：Go 程序如何退出与清理
- **Goroutine 追踪**：`gcount()` 如何统计活跃 goroutine
- **协作式调度**：`Gosched()` 如何让出执行权给其他 goroutine
- **Runtime 修改**：一个小改动如何影响所有 Go 程序
- **设计取舍**：自动等待的好处与坏处

## 扩展想法

可以继续尝试：

1. 加超时：最多等 goroutine 10 秒
2. 加日志：开始等待时打印，以及还剩哪些 goroutine
3. 做成可配置：用环境变量开关
4. 加告警：检测 goroutine 里是否卡死/死循环

## 清理

恢复标准 Go 行为：

```bash
cd go/src/runtime
git checkout proc.go
cd ..
./make.bash
```

## 总结

你已经成功修改了 Go runtime，让它「有耐心」，会等待所有 goroutine！

```
Before:  main() returns → immediate exit → goroutines abandoned
After:   main() returns → wait for goroutines → all complete → exit

Changes: runtime/proc.go main() function
Result:  No goroutine left behind!
```

这次修改展示了：

- 对 Go runtime 的深入理解
- 程序终止如何工作
- `main()` 与 goroutine 的关系
- 语言设计里真实的取舍

现在你的 Go 变得很有耐心。

---

*继续 [练习 8](08-goroutine-sleep-detective.zh.md)，或返回 [workshop 主页](../README.md)*
