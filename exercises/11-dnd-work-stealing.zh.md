# 练习 11: D&D Work Stealing —— 掷骰子抢 Goroutine

> **想深入了解？** 阅读 Internals for Interns 上的 [The Scheduler](https://internals-for-interns.com/posts/go-runtime-scheduler/)，深入了解 Go 的 runtime 与 goroutine 调度。

在本练习中，你会在 Go scheduler 的 work stealing 算法里加上一次 d20 掷骰。当某个 processor（P）想从另一个 P 的 run queue 里偷 goroutine 时，必须先在二十面骰上掷出大于 10 的点数。掷失败就偷不成——这样既能看见 work 是怎么被分走的，也挺好玩。

## 学习目标

完成本练习后，你将能够：

- 理解 Go 的 work stealing scheduler 如何在各个 processor 之间分发 goroutine
- 知道 `stealWork` 函数在哪里，以及它如何遍历其他 P
- 改造 steal 逻辑，加上一层随机门槛
- 实时观察 work stealing 的尝试过程

## 背景：Work Stealing

Go 的 scheduler 用 work stealing 在各个 processor 之间平衡负载。当某个 P 没有可执行的 goroutine 时，它会查看其他 P 的队列，并偷走对方大约一半的工作：

```
Before (current behavior):
  P0: [g1, g2, g3, g4]    P1: []  (idle)
  P1 tries to steal from P0 → always succeeds
  P0: [g1, g2]             P1: [g3, g4]

After (our modification):
  P0: [g1, g2, g3, g4]    P1: []  (idle)
  P1 rolls d20 to steal from P0 → rolled 7, failed!
  P1 rolls d20 to steal from P0 → rolled 16, stole!
  P0: [g1, g2]             P1: [g3, g4]
```

## 第 1 步：理解 Steal 机制

work stealing 逻辑在 `proc.go` 的 `stealWork` 函数里：

```bash
cd go/src/runtime
grep -n "func stealWork" proc.go
```

你会在大约第 3828 行找到 `stealWork`。当某个 P 本地没有工作可做时，`findRunnable` 会调用它。它会按随机顺序遍历其他 P，尝试从它们的 run queue 里偷 goroutine。

## 第 2 步：找到真正发起 Steal 的位置

在 `stealWork` 内部，真正尝试 steal 的代码大约在第 3883 行：

```bash
grep -n "runqsteal" proc.go | head -5
```

你会看到类似这样的代码块（大约第 3883–3887 行）：

```go
// go/src/runtime/proc.go:3883-3887
// Don't bother to attempt to steal if p2 is idle.
if !idlepMask.read(enum.position()) {
    if gp := runqsteal(pp, p2, stealTimersOrRunNextG); gp != nil {
        return gp, false, now, pollUntil, ranTimer
    }
}
```

此处的关键变量：
- **`pp`** —— 当前 P（小偷），类型 `*p`，有字段 `pp.id`（int32）
- **`p2`** —— 目标 P（受害者），类型 `*p`，有字段 `p2.id`
- **`runqsteal(pp, p2, ...)`** —— 把 goroutine 从 p2 的队列挪到 pp 的队列

## 第 3 步：加上 D&D 掷骰

把第 3883–3887 行替换成我们带骰子门槛的版本：

```go
// go/src/runtime/proc.go:3883-3887
// Don't bother to attempt to steal if p2 is idle.
if !idlepMask.read(enum.position()) {
    if mainStarted && gogetenv("GODND") != "" {
        // D&D Work Stealing: Roll a d20 to attempt stealing!
        roll := cheaprandn(20) + 1 // Roll 1-20
        if roll > 10 {
            if gp := runqsteal(pp, p2, stealTimersOrRunNextG); gp != nil {
                println("🎲 [P", pp.id, "] Rolling to steal from P", p2.id, "... rolled", roll, ". Stole!")
                return gp, false, now, pollUntil, ranTimer
            }
        } else {
            println("🎲 [P", pp.id, "] Rolling to steal from P", p2.id, "... rolled", roll, ". Failed!")
        }
    } else {
        if gp := runqsteal(pp, p2, stealTimersOrRunNextG); gp != nil {
            return gp, false, now, pollUntil, ranTimer
        }
    }
}
```

### 理解这段代码

- **`mainStarted`** —— `proc.go` 里已有的布尔值，会在 main goroutine 启动早期（`sysmon`、GC 和 `init()` 运行之前）置为 `true`。它能滤掉最早期的一部分 scheduler 噪音，但 main 之前仍可能有一些打印（见下方说明）
- **`gogetenv("GODND")`** —— runtime 内部的环境变量读取函数（相当于 `os.Getenv`——runtime 不能 import `os`）。所有改动都挂在 `GODND=1` 后面，不主动开启时 scheduler 行为保持正常
- **`cheaprandn(20) + 1`** —— 掷出 1–20。`cheaprandn(20)` 返回 0–19，`+ 1` 把它调成标准 d20 范围
- **`roll > 10`** —— 约 50% 成功率（11–20 成功，1–10 失败）
- **`println(...)`** —— runtime 内置打印，经原始 syscall 写到 stderr，无需任何 import
- **`pp.id` / `p2.id`** —— processor 的 ID 字段（int32），定义在 `runtime2.go` 的 `p` 结构体里
- 只有当 `runqsteal` 返回非 nil 时我们才打印 "Stole!"（目标队列可能在 idle 检查与真正 steal 之间已经空了）

> **🐉 冷知识：scheduler 其实已经在“带优势掷骰”——而且还是四次！**
>
> 看看 `stealWork` 的外层循环：`const stealTries = 4`。scheduler 不是只偷一次——它会对所有 P **循环 4 遍**，每一遍都用 `cheaprand()` 重新打乱顺序。所以你的 d20 门槛不是每个 steal 尝试只掷一次——一个死磕的 P 对每个目标最多能有 4 次机会。用 D&D 的话说，这相当于优势……再平方。
>
> 而且第 4 遍很特别：它会把 `stealTimersOrRunNextG = true`，从而允许偷走受害者的 `runnext` goroutine——也就是对方马上要跑的那一个。源码注释里原话是 *"stealing from the other P's runnext should be the last resort."* 所以最后一轮是撕破脸的全力回合，什么都可以抢。
>
> 其实 Go scheduler 在你动手之前就已经在玩 D&D 了。你只是把骰子结果打出来而已。

## 第 4 步：重新编译 Go Toolchain

```bash
cd ../  # back to go/src
./make.bash
```

## 第 5 步：测试 D&D Scheduler

创建 `dnd_steal_demo.go` 文件：

```bash
# Create the file
touch /tmp/dnd_steal_demo.go
```

```go
package main

import (
    "runtime"
    "sync"
)

func busyWork(id int, wg *sync.WaitGroup) {
    defer wg.Done()
    sum := 0
    for i := 0; i < 1_000_000; i++ {
        sum += i
    }
    println("Goroutine", id, "finished")
}

func main() {
    runtime.GOMAXPROCS(4) // 4 P's for visible stealing
    println("=== D&D Work Stealing Demo ===")
    println()

    var wg sync.WaitGroup
    for i := 1; i <= 20; i++ {
        wg.Add(1)
        go busyWork(i, &wg)
    }
    wg.Wait()
    println()
    println("=== All goroutines completed! ===")
}
```

先编译再带 D&D 模式运行：

```bash
../go/bin/go build -o dnd_steal_demo /tmp/dnd_steal_demo.go
GODND=1 ./dnd_steal_demo
```

**为什么要 `GODND=1`？** 不加的话，scheduler 按正常路径跑——不掷骰、不打印。这样 `./make.bash` 和 `go build` 才会干净安静。注意：即使设了 `GODND=1`，main 前后仍可能有一些打印——原因见下方说明。

**为什么先 build 再 run？** `go build` 本身也会用你改过的 Go。分开编译（不带 `GODND=1`）可以让编译器自己的 work stealing 保持安静。

预期输出（每次运行都会不同）：

```
🎲 [P 3 ] Rolling to steal from P 0 ... rolled 13 . Stole!
=== D&D Work Stealing Demo ===

🎲 [P 2 ] Rolling to steal from P 0 ... rolled 15 . Stole!
🎲 [P 3 ] Rolling to steal from P 0 ... rolled 12 . Stole!
🎲 [P 1 ] Rolling to steal from P 0 ... rolled 11 . Stole!
Goroutine 15 finished
🎲 [P 0 ] Rolling to steal from P 2 ... rolled 7 . Failed!
🎲 [P 0 ] Rolling to steal from P 3 ... rolled 20 . Stole!
Goroutine 1 finished
Goroutine 12 finished
...
=== All goroutines completed! ===
🎲 [P %
```

你会看到骰子结果和 goroutine 完成消息交织在一起。掷出 1–10 失败，11–20 成功。注意：没活干的 P 会不停换目标重试，直到掷出足够高的点数！

> **📖 为什么在 `=== D&D Work Stealing Demo ===` 之前就有骰子输出？**
>
> 那些不是你写的——是 Go runtime 干的。从 `mainStarted` 变成 `true` 到你第一次 `println` 之间，runtime 会启动 `sysmon`、启用 GC，并跑完所有 `init()`——每一步都会创建 goroutine，空闲的 P 立刻就想偷。你已经给 scheduler 本身打了桩，所以现在能看见那些一直存在、只是以前静默的活动。

> **📖 为什么在 `=== All goroutines completed! ===` 之后会出现截断的 `🎲 [P %`？**
>
> 故事的另一头。`wg.Wait()` 返回后，空闲的 P 还在 `stealWork` 里空转找活。有个 P 刚开始 `println`，`main()` 就返回了、进程退出——打印没写完。末尾的 `%` 是 shell 在告诉你：输出没有以换行结尾。scheduler 在你离开时也不会停下来等你。

> 欢迎来到 Go 的内部世界！


## 理解我们做了什么

1. **找到了 Steal 逻辑**：定位到 `proc.go` 里 P 互相偷 goroutine 的 `stealWork`
2. **加了骰子门槛**：用 `cheaprandn(20) + 1` 在每次 steal 尝试前生成 d20（1–20）
3. **记录了掷骰结果**：用 `println()` 打印哪个 P 在偷谁、掷了多少、是否成功
4. **观察到了效果**：实时看到 work stealing 尝试，有些因点数太低而失败

## 我们学到了什么

- **Work Stealing**：当队列不平衡时，Go 如何把 goroutine 分发到各个 processor
- **`stealWork` 函数**：遍历各个 P、寻找可偷工作的核心循环
- **`cheaprandn`**：runtime 的快速伪随机数生成器，scheduler 里到处在用
- **Scheduler 可观测性**：如何给 scheduler 加日志，又不破坏其行为
- **P 的身份**：每个 processor 都有唯一的 `id` 字段，用于调度决策

## 扩展想法

1. 让难度可配置：必须掷过 15 而不是 10（更难偷）
2. 掷出 natural 20 时来一次 "critical hit"：偷走目标全部 goroutine，而不只是一半
3. 掷出 natural 1 时来一次 "fumble"：偷取方用 `Gosched()` 让出一轮
4. 在程序退出时统计并打印总掷骰次数、成功次数、失败次数

## 清理

要去掉骰子逻辑：

```bash
cd go/src/runtime
git checkout proc.go
cd ../
./make.bash
```

## 总结

你已经把 Go 的 work stealing scheduler 变成了一场桌面 RPG 遭遇战：

```
Before:  P tries to steal -> always succeeds if target has work
After:   P tries to steal -> must roll > 10 on a d20 first

Changes: runtime/proc.go stealWork() function (~14 lines)
Result:  The scheduler now plays D&D!
```

---

*恭喜完成全部工作坊练习！返回 [工作坊主页](../README.md)*
