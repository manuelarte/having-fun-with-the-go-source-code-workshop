# 练习 8: Goroutine Sleep Detective - Runtime 状态监控

> 📖 **想深入了解？** 阅读 Internals for Interns 上的 [The Scheduler](https://internals-for-interns.com/posts/go-runtime-scheduler/)，深入理解 Go 的 goroutine 调度与状态转换。

本练习中，你将修改 Go runtime scheduler，让它在 goroutine 状态切换时打日志。每当一个 goroutine 因等待某事而进入睡眠，它都会自报家门：「Hello, I'm goroutine 42, going to sleep waiting for channel receive」。

## 学习目标

完成本练习后，你将能够：

- 理解 Go 的 goroutine scheduler 状态转换
- 知道 goroutine 在 runtime 里会在哪里阻塞
- 改造 scheduler，获得调试洞察

## 引言：Scheduler 是怎么工作的？

Go 的 scheduler 使用 **GMP 模型**（Goroutine、Machine、Processor），把可能成千上万的 goroutine 映射到少量 OS 线程上。关键洞察是：当某个 OS 线程阻塞时（例如卡在 syscall），调度资源（P）可以拆下来挂到别的线程上，让工作继续流动。

Goroutine 并没有一个专门的 scheduler 线程在管它们。它们采用 **self-service（自助）模式** 管理自己的状态转换：当 goroutine 需要等待（channel、mutex、sleep 等）时，它会调用 `gopark()` 把自己 park 起来，加入对应的等待队列，再调用 `schedule()` 去找下一个可运行的 goroutine。等待条件满足后，`goready()` 再把它重新标为可运行。

Scheduler 按优先级选下一个要跑的 goroutine：先 GC 工作，再本地 `runnext` 槽，然后本地 run queue，接着全局队列（大约每 61 次检查一次，防止饿死），再看 network poller 的结果，最后从其他 P 做 work stealing。

理解这条调度路径很重要，因为本练习要在 goroutine **切到 waiting 状态** 的精确位置加日志。

## 背景：Goroutine 状态

Go 用不同状态管理 goroutine：

- **`_Grunnable`** - 就绪，但还没在执行
- **`_Grunning`** - 正在执行
- **`_Gwaiting`** - 阻塞，在等某事（我们的目标！）
- **`_Gsyscall`** - 正在执行系统调用
- ...

当 goroutine 需要等待（channel、mutex、sleep 等）时，它会「park」并切换到 `_Gwaiting`。

## 第 1 步：理解 Park 机制

所有同步原语在需要让 goroutine 等待时，都会调用 `gopark`。

```bash
cd go/src/runtime
grep -n "func gopark" proc.go
```

关键函数：

- **`gopark()`** - 发起 park 一个 goroutine
- **`park_m()`** - 真正把状态改成 `_Gwaiting`

## 第 2 步：找到状态转换代码

```bash
# Look at where the state actually changes
grep -n -A 5 "func park_m" proc.go
```

大约在第 4275 行，你会看到：

```go
casgstatus(gp, _Grunning, _Gwaiting)
```

这正是 goroutine 从 running 切到 waiting 的那一行。非常适合加日志！

## 第 3 步：加入 Goroutine 睡眠日志

**编辑 `proc.go`：**

你需要在三个会切到 waiting 状态的位置加日志：

### 位置 1：`casGToWaiting` 函数（大约第 1388 行）

找到 `casGToWaiting`，在设置 wait reason 之后加日志：

```go
func casGToWaiting(gp *g, old uint32, reason waitReason) {
	// Set the wait reason before calling casgstatus, because casgstatus will use it.
	gp.waitreason = reason
	if gp.goid > 1 { // Skip system goroutines 0 and 1
		print("Hello, I'm goroutine ", gp.goid, ", going to sleep waiting for ", gp.waitreason.String(), "\n")
	}
	casgstatus(gp, old, _Gwaiting)
}
```

### 位置 2：`casGFromPreempted` 函数（大约第 1430 行）

找到被抢占的 goroutine 切到 waiting 的地方。在设置 `waitreason` 之后、`CompareAndSwap` 之前加日志：

```go
func casGFromPreempted(gp *g, old, new uint32) bool {
	if old != _Gpreempted || new != _Gwaiting {
		throw("bad g transition")
	}
	gp.waitreason = waitReasonPreempted
	if gp.goid > 1 { // Skip system goroutines 0 and 1
		print("Hello, I'm goroutine ", gp.goid, ", going to sleep waiting for ", gp.waitreason.String(), "\n")
	}
	if !gp.atomicstatus.CompareAndSwap(_Gpreempted, _Gwaiting) {
		return false
	}
	if bubble := gp.bubble; bubble != nil {
		bubble.changegstatus(gp, _Gpreempted, _Gwaiting)
	}
	return true
}
```

### 位置 3：`park_m` 函数（大约第 4275 行）

找到 `park_m`，在直接调用 `casgstatus` 之前加日志：

```go
// Add this before: casgstatus(gp, _Grunning, _Gwaiting)
if gp.goid > 1 { // Skip system goroutines 0 and 1
    print("Hello, I'm goroutine ", gp.goid, ", going to sleep waiting for ", gp.waitreason.String(), "\n")
}
casgstatus(gp, _Grunning, _Gwaiting)
```

### 理解这段代码

- **`gp.goid`** - 唯一的 goroutine ID
- **`gp.waitreason.String()`** - 人类可读的等待原因（channel、mutex、sleep 等）
- **`print()`** - runtime 的打印函数（输出到 stderr）
- **`gp.goid > 1`** - 跳过系统 goroutine，减少噪声

## 第 4 步：重新构建 Go Runtime

```bash
cd ../  # back to go/src
./make.bash
```

## 第 5 步：测试 Channel 阻塞

创建 `channel_demo.go`：

```go
package main

import "time"

func main() {
    ch := make(chan string)

    // Start goroutine that will block on receive
    go func() {
        msg := <-ch  // Should trigger our logging!
        println("Received:", msg)
    }()

    // Let the goroutine start and block
    time.Sleep(100 * time.Millisecond)

    // Send something
    ch <- "Hello!"
    time.Sleep(10 * time.Millisecond)
}
```

用我们改过的 Go 先构建再运行：

```bash
../go/bin/go build channel_demo.go
./channel_demo
```

**注意：** 我们先构建二进制，再直接运行。这样可以避免把编译/构建过程里的 goroutine 和我们程序自己的 goroutine 混在一起，输出会更干净！

期望输出：

```
Hello, I'm goroutine 4, going to sleep waiting for GC scavenge wait
Hello, I'm goroutine 3, going to sleep waiting for GC sweep wait
Hello, I'm goroutine 2, going to sleep waiting for force gc (idle)
Hello, I'm goroutine 6, going to sleep waiting for chan receive
Hello, I'm goroutine 5, going to sleep waiting for GOMAXPROCS updater (idle)
Received: Hello!
```

你现在能看到哪些 goroutine 在阻塞了。

## 我们做了什么

1. **找到 Park 函数**：定位 goroutine 切到 waiting 的位置
2. **加入日志**：在状态切换前插入 print
3. **捕获等待原因**：用 `gp.waitreason.String()` 输出人类可读信息
4. **多场景验证**：用 channel、mutex、sleep、select 验证

你可能会看到的常见 wait reason：

- `chan receive` / `chan send`
- `sync mutex lock`
- `sleep`
- `GC`

## 我们学到了什么

- **Goroutine 生命周期**：goroutine 如何在状态之间切换
- **Park 机制**：`gopark` 与 `park_m`
- **同步原语内部**：channel、mutex、select 在哪里触发阻塞
- **Runtime 调试**：如何给 Go runtime 加可观测性
- **并发可见性**：实时观察阻塞操作

## 扩展想法

可以继续尝试：

1. 加 goroutine 唤醒日志（重新开始跑时）
2. 给不同 wait reason 加不同图标（channel、mutex、sleep）
3. 加时间戳，测量阻塞时长
4. 按特定 wait reason 过滤日志

## 清理

去掉日志：

```bash
cd go/src/runtime
git checkout proc.go
cd ../
./make.bash
```

## 总结

你已经给 Go 的并发模型装上了 X 光眼！改过的 runtime 现在会在每次 goroutine 阻塞时自报家门：

```
Hello, I'm goroutine 18, going to sleep waiting for chan receive
Hello, I'm goroutine 19, going to sleep waiting for sync mutex lock
Hello, I'm goroutine 20, going to sleep waiting for sleep
```

本练习揭开了 Go scheduler 的内部机制，以及同步原语如何与 runtime 交互。

---

*继续 [练习 9](09-predictable-select.zh.md)，或返回 [workshop 主页](../README.md)*
