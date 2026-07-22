# 练习 9: 可预测的 Select —— 让 Select 语句变成确定性的

> 📖 **想深入了解？** 阅读 Internals for Interns 上的 [The Scheduler](https://internals-for-interns.com/posts/go-runtime-scheduler/)，深入了解 Go 的 runtime 与 goroutine 调度。

在本练习中，你会把 Go 的 `select` 语句从随机选择改成确定性选择。默认情况下，当多个 channel 同时就绪时，Go 会随机挑一个 case。我们要改成：始终按固定顺序选择。

## 学习目标

完成本练习后，你将能够：

- 理解 Go 的 `select` 语句是如何实现的
- 明白 Go 为什么要用随机化（公平性 vs. 饥饿）
- 修改 runtime 的 channel 选择算法
- 对比并测试确定性选择与随机选择的行为差异

## 引言：Select 在内部是怎么工作的？

`select` 语句实现在 runtime 的 `selectgo()` 函数里，文件是 `runtime/select.go`。当代码执行到带多个 case 的 `select` 时，runtime 需要决定执行哪个 case。它靠两个数组来完成这件事：

- **`pollorder`**：决定 case 被**检查就绪**的顺序。默认会用 `cheaprandn()` 打乱顺序，保证公平——不会让某个 channel 永远优先。
- **`lockorder`**：决定 channel 锁的**获取**顺序（按地址排序，避免死锁）。

runtime 会先把 case 打乱填进 `pollorder`，再按这个顺序遍历。如果某个 case 的 channel 已经就绪（有数据可收，或有空间可发），就选中它。如果所有 case 都没就绪且有 `default`，就走 default。否则，当前 goroutine 会把自己挂到这些 channel 的等待队列上，直到有一个就绪。

`pollorder` 上的随机化，正是 `select` 不确定的来源——同一个 `select`、同样就绪的 channel，每次可能选出不同 case。这是刻意的设计，防止程序无意中依赖 case 书写顺序。

## 背景：Go 会随机化 Select

默认情况下，多个 channel 同时就绪时，Go 会随机决定执行哪个 case：

```go
select {
case v := <-ch1:  // Sometimes chosen
case v := <-ch2:  // Sometimes chosen
case v := <-ch3:  // Sometimes chosen
}
// Random selection prevents starvation
```

我们要让它变成确定性的：

```go
select {
case v := <-ch1:  // ALWAYS chosen first when ready
case v := <-ch2:  // Only if ch1 not ready
case v := <-ch3:  // Only if ch1 and ch2 not ready
}
// Predictable, source-order selection
```

## 第 1 步：写一个测试，观察当前的随机行为

创建 `random_select_demo.go` 文件：

```go
package main

func main() {
    ch1 := make(chan int, 1)
    ch2 := make(chan int, 1)
    ch3 := make(chan int, 1)

    // Fill all channels so they're all ready
    ch1 <- 1
    ch2 <- 2
    ch3 <- 3

    // Run select 10 times to see randomization
    for i := 0; i < 10; i++ {
        select {
        case v := <-ch1:
            println("Round", i, ": Selected ch1 (value", v, ")")
            ch1 <- 1 // Refill
        case v := <-ch2:
            println("Round", i, ": Selected ch2 (value", v, ")")
            ch2 <- 2 // Refill
        case v := <-ch3:
            println("Round", i, ": Selected ch3 (value", v, ")")
            ch3 <- 3 // Refill
        }
    }
}
```

用当前的 Go 运行，观察随机选择：

```bash
go run random_select_demo.go
```

输出会呈现随机选择：

```
Round 0: Selected ch3 (value 3)
Round 1: Selected ch1 (value 1)
Round 2: Selected ch2 (value 2)
...
```

## 第 2 步：定位 Select 的实现

```bash
cd go/src/runtime
```

`select.go` 包含了整套 select 语句实现。核心函数是 `selectgo()`，负责 case 选择。

## 第 3 步：读懂随机化代码

看 `select.go` 大约第 191 行附近：

```go
// go/src/runtime/select.go:191
j := cheaprandn(uint32(norder + 1))  // Random index!
pollorder[norder] = pollorder[j]
pollorder[j] = uint16(i)
norder++
```

这段代码实现了打乱 case 顺序的算法：

- `cheaprandn()` 生成伪随机数
- case 被放进 `pollorder` 数组的随机位置
- Select 再按这个随机顺序检查 case

## 第 4 步：让 Select 变成确定性的

**编辑 `select.go`：**

找到第 191 行，把随机化改成确定性逻辑：

```go
// go/src/runtime/select.go:191
// Original:
j := cheaprandn(uint32(norder + 1))
pollorder[norder] = pollorder[j]
pollorder[j] = uint16(i)

// Change to:
pollorder[norder] = uint16(len(scases)-1-i)
```

### 理解这次代码改动


- **`uint16(len(scases)-1-i)`**：这里用反向索引
- **结果**：`pollorder` 会始终按源码顺序排列
- **效果**：case 在 `pollorder` 中保持源码书写顺序

## 第 5 步：重新编译 Go Runtime

```bash
cd ../  # back to go/src
./make.bash
```

## 第 6 步：验证确定性行为

```bash
../go/bin/go run random_select_demo.go
```

现在你应该看到**确定性输出**：

```
Round 0: Selected ch1 (value 1)
Round 1: Selected ch1 (value 1)
Round 2: Selected ch1 (value 1)
Round 3: Selected ch1 (value 1)
...
```

完美！`ch1` **总是**被选中，因为它在源码里排第一，不再有随机顺序。

## 理解我们做了什么

1. **去掉随机化**：用确定性索引替换了 `cheaprandn()`
2. **保持源码顺序**：case 现在按出现顺序检查
3. **轻微性能提升**：稍快一点（不再生成随机数）
4. **改变语义**：语法不变，runtime 行为变了

## 我们学到了什么

- **Runtime 修改**：如何改动语言的基础行为
- **设计权衡**：并发系统中公平性与确定性的取舍
- **Select 内部机制**：`selectgo` 与 `pollorder` 如何工作
- **行为测试**：用测试程序验证语义变化

## 扩展想法

可以尝试这些额外改造：

1. 增加逆序模式（从后往前检查 case）
2. 按 case 位置增加优先级
3. 记录选择统计信息，方便调试
4. 通过环境变量让随机化可配置

## 清理

要恢复 Go 原始的随机行为：

```bash
cd go/src/runtime
git checkout select.go
cd ../
./make.bash
```

## 总结

你已经把 Go 的 `select` 从一个公平、随机的选择器，改成了可预测、确定性的优先级系统：

```go
// Before: Random selection (fair but unpredictable)
select {
case <-ch1: // 33% chance
case <-ch2: // 33% chance
case <-ch3: // 33% chance
}

// After: Deterministic selection (predictable but may starve)
select {
case <-ch1: // Always chosen when ready
case <-ch2: // Only if ch1 not ready
case <-ch3: // Only if ch1 and ch2 not ready
}
```

本练习展示了：runtime 层面的改动可以从根本上改变语言行为，同时也揭示了并发系统设计中的重要权衡。

---

*继续学习 [练习 10](10-java-style-stack-traces.zh.md)，或返回 [工作坊主页](../README.md)*
