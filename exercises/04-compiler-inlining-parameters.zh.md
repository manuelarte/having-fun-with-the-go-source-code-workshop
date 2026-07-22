# 练习 4: 编译器 Inlining 参数 — 调参控制 Binary 体积

> 📖 **想深入了解？** 阅读 Internals for Interns 上的 [The IR](https://internals-for-interns.com/posts/the-go-ir/)，深入理解 Go 的 intermediate representation，包括函数 inlining 决策是如何做出的。

在本练习中，你将探索并修改 Go 的 inlining 参数，观察它们对 binary 体积的显著影响。这会教你 Go 编译器如何决定何时 inline 函数，以及微调这些参数会如何大幅改变编译结果。

## 学习目标

完成本练习后，你将能够：

- 理解 Go 的 inlining budget 体系及其参数
- 知道编译器中 inlining 决策发生在哪里
- 修改 inlining 阈值以控制优化行为
- 衡量改动对 binary 体积的影响

## 简介：什么是 IR？

在 parsing 和 type-checking 之后，编译器会把 AST 转换成 **Intermediate Representation（IR）**。AST 更贴近你在源码里写下的结构，而 IR 是另一种表示形式，更适合编译器做分析和变换。

IR 用大约 150 种操作码（如 `OADD`、`OCALL`、`OIF`）表示代码中的每一步操作。每个 IR 节点都带有类型信息，并按 package 组织。这一阶段会做出许多重要的优化决策——其中影响最大的之一就是 **function inlining**。

编译器会用一个 "hairiness visitor" 遍历 IR 树，给每个节点打上成本分。若某个函数的总成本落在 **inlining budget**（默认：80 个节点）以内，该函数就有资格被 inline。函数调用的成本是 57 个节点，简单语句是 1 个节点。当在调用点 inline 一个函数时，编译器会复制其函数体，并把参数替换为实参。

你可以用 `go build -gcflags='-m'`（或 `-m=2` 看更详细原因）观察 inlining 决策。

## 背景：Go 中的 Function Inlining

Function inlining 是一种编译器优化：用函数体本身替换函数调用。这是在用 binary 体积换性能：

**收益：**

- 消除调用开销
- 便于在调用点做进一步优化
- 更好的指令流水线利用率

**代价：**

- binary 体积变大
- 程序运行时占用更多内存

Go 使用一套精巧的 **budget 体系** 来判断何时 inline 是划算的。

## 第 1 步：理解 Go 的 Inlining Budget

先看看当前的 inlining 参数：

```bash
cd go/src/cmd/compile/internal/inline
```

打开 `inl.go`，查看大约第 49–85 行的关键参数：

### 关键 Inlining 参数

摘自 `go/src/cmd/compile/internal/inline/inl.go:49-85`：

```go
const (
    inlineMaxBudget       = 80
    inlineExtraAppendCost = 0
    inlineExtraCallCost   = 57              // benchmarked to provide most benefit
    inlineParamCallCost   = 17              // calling a parameter costs less
    inlineExtraPanicCost  = 1               // do not penalize inlining panics
    inlineExtraThrowCost  = inlineMaxBudget // inlining runtime.throw does not help

    inlineBigFunctionNodes      = 5000                 // Functions with this many nodes are "big"
    inlineBigFunctionMaxCost    = 20                   // Max cost when inlining into a "big" function
    inlineClosureCalledOnceCost = 10 * inlineMaxBudget // if a closure is called once, inline it
)

var (
    // ...
    // Budget increased due to hotness (PGO).
    inlineHotMaxBudget int32 = 2000
)
```

**说明：** `inlineHotMaxBudget` 是 `var` 而不是 `const`，因为它服务于 PGO（Profile Guided Optimization），可能在运行时被修改。

### Budget 体系如何工作

每条 Go 语句/表达式都有一个 **成本**：

- 简单语句：1 分
- 函数调用：57+ 分
- 循环、条件：各 1 分
- 复杂表达式：分数不定

编译器把成本累加，再与 budget 比较。

## 第 2 步：用 Go 编译器 Binary 做体积对比

与其写玩具程序，不如直接把 Go 编译器 binary 本身当作测试对象！`bin/go` 非常适合展示 inlining 的效果，因为：

- **代码体量大** — 能看出有意义的体积差异
- **真实世界代码** — 包含我们真正在优化的那些模式
- **与工作坊相关** — 整个练习过程都在构建它
- **效果明显** — 体量足够大，能展现显著的 inlining 影响

### 在 Go Binary 上测试不同 Inlining 设置

我们用不同的 inlining 设置重新构建整个 Go toolchain，然后对比 `bin/go` 的体积：

```bash
cd go/src
```

### 基线构建 — 默认设置

先用默认 inlining 设置构建，并备份 binary：

```bash
# Build with default settings
./make.bash

# Copy the default Go binary for comparison
cp ../bin/go ../bin/go-default

# Check the size
ls -lh ../bin/go-default
wc -c ../bin/go-default
```

### 查看 Inlining 对 Go 编译器自身构建的影响

我们可以观察编译 Go 编译器时，inlining 是如何起作用的：

```bash
# See inlining decisions when compiling the Go compiler
# This shows how inlining parameters affect the compiler's own build process
cd cmd/compile
../../bin/go build -gcflags="-m" . 2>&1 | grep "can inline" | wc -l
echo "Functions that can be inlined during Go compiler build"
```

## 第 3 步：修改 Inlining 参数

现在来改 inlining 参数，看看效果！

### 实验 1：激进 Inlining

编辑 `go/src/cmd/compile/internal/inline/inl.go` 大约第 50 行：

```go
const (
    inlineMaxBudget       = 95    // Increased from 80
    inlineExtraCallCost   = 40    // Decreased from 57
    inlineBigFunctionMaxCost = 30 // Increased from 20
)
```

> **⚠️ 注意：** 别把这些值调得过高！在 Go 1.26.1 中，runtime 对 write barrier 有严格约束；若把 inlining budget 调到大约超过 95，编译器会把函数 inline 进禁止 write barrier 的上下文，导致构建失败。这本身就是很好的一课：编译器参数需要精细平衡。

**重新构建编译器：**

```bash
cd go/src
./make.bash
```

**在 Go binary 上测试激进 inlining：**

```bash
# Copy the aggressively-inlined Go binary
cp ../bin/go ../bin/go-aggressive

# Compare sizes
echo "Default size: $(wc -c < ../bin/go-default)"
echo "Aggressive size: $(wc -c < ../bin/go-aggressive)"

# Calculate size difference
default_size=$(wc -c < ../bin/go-default)
aggressive_size=$(wc -c < ../bin/go-aggressive)
echo "Size difference: $(($aggressive_size - $default_size)) bytes"
echo "Percentage increase: $(echo "scale=2; ($aggressive_size - $default_size) * 100 / $default_size" | bc)%"
```

### 实验 2：保守 Inlining

接下来试试保守设置。编辑参数：

```go
const (
    inlineMaxBudget       = 40    // Decreased from 80
    inlineExtraCallCost   = 100   // Increased from 57
    inlineBigFunctionMaxCost = 5  // Decreased from 20
)
```

**重新构建并测试：**

```bash
cd go/src
./make.bash

# Copy the conservatively-inlined Go binary
cp ../bin/go ../bin/go-conservative

# Compare all three Go binaries
echo "Conservative size: $(wc -c < ../bin/go-conservative)"
echo "Default size: $(wc -c < ../bin/go-default)"
echo "Aggressive size: $(wc -c < ../bin/go-aggressive)"
```

## 第 4 步：全面的 Binary 体积分析

再试试更极端的 inlining 设置，观察对 Go 编译器 binary 的戏剧性影响：

### 实验 3：完全不做 Inlining

作为对比，彻底关闭 inlining：

```go
const (
    inlineMaxBudget       = 0     // No inlining budget
    inlineExtraCallCost   = 1000  // Prohibitive call cost
    inlineBigFunctionMaxCost = 0  // No big function inlining
)
```

```bash
cd go/src
./make.bash

# Copy the no-inlining Go binary
cp ../bin/go ../bin/go-no-inline
```

### 实验 4：极端 Inlining — 演示崩溃临界点

试试非常激进的设置，看看把 inlining 推得过远会发生什么：

```go
const (
    inlineMaxBudget       = 500   // Very high budget
    inlineExtraCallCost   = 5     // Very low call cost
    inlineBigFunctionMaxCost = 200 // Very high big function budget
)
```

```bash
cd go/src
./make.bash
```

**⚠️ 预期结果：** 这次构建会失败！你会看到 "write barrier prohibited by caller" 之类的错误。原因是：编译器把 runtime 函数 inline 进了不允许 write barrier 的上下文，形成了非法的调用链。

如果失败了（这是预期的），你会学到：

- 极端 inlining 会在 runtime 中触发 write barrier 违规
- Go runtime 里有 `//go:nowritebarrierrec` 注解，禁止某些调用链中出现 write barrier
- 一旦 inlining 暴露了这些调用链，编译器会正确地拒绝构建
- 默认参数经过精心平衡，是有道理的

## 第 5 步：分析结果

对比各个 Go 编译器 binary 的体积：

```bash
cd go

echo "=== GO COMPILER BINARY SIZE COMPARISON ==="
echo "No Inlining:  $(wc -c < bin/go-no-inline) bytes"
echo "Conservative: $(wc -c < bin/go-conservative) bytes"
echo "Default:      $(wc -c < bin/go-default) bytes"
echo "Aggressive:   $(wc -c < bin/go-aggressive) bytes"

echo ""
echo "=== SIZE DIFFERENCES ==="
no_inline_size=$(wc -c < bin/go-no-inline)
conservative_size=$(wc -c < bin/go-conservative)
default_size=$(wc -c < bin/go-default)
aggressive_size=$(wc -c < bin/go-aggressive)

echo "No-inline vs Default: $(($default_size - $no_inline_size)) bytes difference"
echo "Default vs Aggressive: $(($aggressive_size - $default_size)) bytes difference"
echo "Full Range (No-inline to Aggressive): $(($aggressive_size - $no_inline_size)) bytes difference"

# Calculate percentages
echo ""
echo "=== PERCENTAGE DIFFERENCES ==="
echo "Aggressive vs Default: $(echo "scale=2; ($aggressive_size - $default_size) * 100 / $default_size" | bc)%"
echo "Default vs No-inline: $(echo "scale=2; ($default_size - $no_inline_size) * 100 / $no_inline_size" | bc)%"
```


## 理解我们改了什么

### 关键参数的作用

| 参数 | 作用 | 影响 |
|-----------|---------|--------|
| `inlineMaxBudget` | 任意被 inline 函数的最大成本 | 越高 → 越多 inlining |
| `inlineExtraCallCost` | 被 inline 函数内部再调用函数的惩罚 | 越低 → 越激进 |
| `inlineBigFunctionMaxCost` | 向大型函数中 inline 时的最大成本 | 越高 → 大函数里越多 inlining |
| `inlineBigFunctionNodes` | 判定「大函数」的节点数阈值 | 越低 → 更多函数被当作「大函数」 |

### 你通常会看到的结果

以 Go 编译器 binary 为例，你应该能观察到明显的体积差异：

- **完全不 Inlining**：体积最小
- **保守**：比默认略小
- **默认**：体积较均衡
- **激进**：比默认更大

**关键洞察：**

- 即便只是温和地改 inlining 参数，也能测出可感知的 binary 体积差异
- 从完全不做 inlining 到激进 inlining 的跨度，能直观展现这项优化的影响
- 更激进的取值会受 runtime 约束限制（write barrier）

具体体积因系统而异，但你应该会看到类似的显著差异。

## 你学到了什么

- **Budget 体系**：Go 如何基于成本分析做 inlining 决策
- **参数影响**：不同设置如何影响 binary 体积与性能
- **测量技巧**：用 debug 标志理解编译器决策
- **权衡取舍**：binary 体积与性能之间的根本张力
- **编译器调参**：如何按特定需求修改编译器行为

## 扩展思路

可以试试这些额外实验：

1. 写脚本自动测试不同参数组合
2. 用真实世界的 Go 程序测试（比如构建 Go 自身！）
3. 测量不同设置下的编译时间差异
4. 实验 PGO（Profile-Guided Optimization）相关参数
5. 分析 inline 与未 inline 调用在汇编输出上的差异

## 下一步

你已经学会了如何调节 Go 的 inlining 行为，并亲眼看到了它对 binary 体积和性能的真实影响。接下来的练习中，我们会探索如何修改 gofmt 工具。

## 清理

恢复原始 inlining 参数，并清理测试 binary：

```bash
cd go/src/cmd/compile/internal/inline
git checkout inl.go
cd ../../../../

# Rebuild with original parameters
cd src
./make.bash

# Clean up test binaries
rm -f ../bin/go-default ../bin/go-aggressive ../bin/go-conservative ../bin/go-no-inline
```

## 关键收获

1. **Inlining 是一种权衡**：更多 inlining = 更大的 binary，但可能更快的执行
2. **Budget 体系**：Go 用精巧的成本分析做 inlining 决策
3. **参数影响大**：参数上的小改动，输出上可能有显著差别
4. **调试工具**：Go 提供了很好的工具来理解编译器决策
5. **贴近现实**：这些参数会影响你编译的每一个 Go 程序

Go 编译器团队通过大量 benchmark 精心调过这些默认值——但现在你已经明白如何按自己的需求去调整它们了。

---

*继续 [练习 5](05-gofmt-ast-transformation.zh.md)，或返回 [工作坊主页](../README.md)*
