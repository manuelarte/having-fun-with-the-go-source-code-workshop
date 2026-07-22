# 练习 6: SSA Pass - 检测除以 2 的幂

> 📖 **想深入了解？** 阅读 Internals for Interns 上的 [The SSA Phase](https://internals-for-interns.com/posts/the-go-ssa/)，深入理解 Go 的 SSA 优化 pass。

本练习中，你将通过编写一个自定义优化 pass，来理解 Go 的 SSA（Static Single Assignment，静态单赋值）编译器 pass 如何工作。这个 pass 会检测「除以 2 的幂」的除法操作。

## 学习目标

完成本练习后，你将能够：

- 理解 Go 的 SSA 编译器 pass 架构
- 掌握如何遍历 SSA block 和 value
- 从零创建一个自定义分析 pass
- 把你的 pass 接入编译器流水线
- 使用 SSA dump 验证 pass 是否生效

## 引言：什么是 SSA？

**Static Single Assignment（SSA，静态单赋值）** 是一种编译器中间表示：每个变量只被赋值一次。普通代码会复用变量，例如 `x = 1; x = x + 2`；SSA 则会生成新版本：`x1 = 1; x2 = x1 + 2`。这一约束消除了歧义——分析某个值时，编译器能明确知道它只来自唯一的定义点，从而支撑大量强大的优化。

SSA 代码主要围绕两个结构组织：

- **Values（值）**：单个计算，例如 `v3 = Add64 v1 v2`。每个 value 都有操作码（Add64、Div32、Const64 等）、类型，以及对输入的引用。
- **Blocks（基本块）**：中间没有分支的 value 序列。block 之间通过控制流边连接，构成函数的控制流图。

当控制流路径汇合时（例如 `if/else` 之后），SSA 使用 **PHI 节点** 来统一不同路径上的值：`v5 = Phi v3 v4` 的含义是「v5 是 v3 或 v4，取决于我们从哪条分支过来」。

编译器会按顺序对 SSA 图运行 30 多个 **pass**。每个 pass 遍历 block 和 value，进行分析或变换。pass 会在 **lowering** 之前和之后运行——lowering 把通用操作（如 `Add64`）转换成架构相关指令（如 `AMD64ADDQ`）。

## 背景：SSA 编译器 Pass

Go 编译器会把你的代码依次变换多轮：

1. **Parse** - 源码 → AST
2. **Type Check** - 类型检查
3. **IR Generation** - 生成 IR（中间表示）
3. **SSA Generation** - 生成 SSA（Static Single Assignment）形式
4. **Optimization Passes** - 变换 SSA（本练习的重点！）
5. **Code Generation** - 生成机器码

我们接下来会在 SSA 形式上动手，看看「除以 2 的幂」有哪些优化机会。

## 第 1 步：理解 SSA Pass 的结构

SSA pass 在 `compile.go` 中注册，并以函数为单位运行。先看一下结构：

```bash
cd go/src/cmd/compile/internal/ssa
```

打开 `compile.go`，搜索 `var passes`（大约在第 457 行附近），你会看到：

```go
var passes = [...]pass{
	{name: "number lines", fn: numberLines, required: true},
	{name: "early phielim and copyelim", fn: copyelim},
	// ... many more passes
}
```

每个 pass 包含：

- **name** - 调试输出中显示的名字
- **fn** - 实际执行变换/分析的函数
- **required** - 该 pass 是否必须运行

## 第 2 步：创建「2 的幂除法」检测 Pass

新建一个文件来放我们的检测 pass：

```bash
cd go/src/cmd/compile/internal/ssa
```

**创建 `powoftwodetector.go`：**

```go
package ssa

import (
	"fmt"
	"math/bits"
)

func detectDivByPowerOfTwo(f *Func) {
	count := 0

	for _, b := range f.Blocks {
		for _, v := range b.Values {
			// Check for division operations
			if v.Op == OpDiv64 || v.Op == OpDiv32 || v.Op == OpDiv16 || v.Op == OpDiv8 ||
				v.Op == OpDiv64u || v.Op == OpDiv32u || v.Op == OpDiv16u || v.Op == OpDiv8u {

				// Check if the divisor (second argument) is a constant
				if len(v.Args) >= 2 {
					divisor := v.Args[1]

					// Check if it's a constant value
					if divisor.Op == OpConst64 || divisor.Op == OpConst32 ||
						divisor.Op == OpConst16 || divisor.Op == OpConst8 {

						constValue := divisor.AuxInt

						// Check if the constant is a power of two
						if isPowerOfTwo(constValue) {
							count++
							if f.pass.debug > 0 {
								fmt.Printf("  [PowerOfTwo] Found division by power of 2: %v / %d (could be >> %d) at %v\n",
									v.Args[0], constValue, bits.TrailingZeros64(uint64(constValue)), v.Pos)
							}
						}
					}
				}
			}
		}
	}

	if count > 0 {
		fmt.Printf("[PowerOfTwo Detector] Function %s: found %d division(s) by power of 2\n", f.Name, count)
	}
}
```

### 理解这段代码

- **`f *Func`** - 正在分析的 SSA 函数
- **`f.Blocks`** - 函数中的所有基本块
- **`b.Values`** - 某个 block 中的所有 SSA value（操作）
- **`v.Op`** - 操作类型（除法、加法等）
- **`v.Args`** - 操作的操作数
- **`divisor.AuxInt`** - 常量的值
- **`isPowerOfTwo()`** - 辅助函数，已经在 `rewrite.go` 中存在
- **`bits.TrailingZeros64()`** - 计算右移多少位

## 第 3 步：在编译器中注册 Pass

**编辑 `compile.go`：**

找到 `var passes` 数组（大约第 457 行），把你的 pass 加到**第一个**条目：

```go
var passes = [...]pass{
	{name: "detect div by power of two", fn: detectDivByPowerOfTwo, required: true},
	{name: "number lines", fn: numberLines, required: true},
	// ... rest of the passes
```

这样会让检测器尽早运行，赶在其他优化可能消掉这些除法之前。

## 第 4 步：重新构建编译器

```bash
cd go/src
./make.bash
```

这会把你的新 pass 编进 Go 编译器。

## 第 5 步：编写测试程序

创建 `test_divisions.go`：

```go
package main

func testDivisions() int {
	x := 100

	// These should be detected (powers of 2)
	a := x / 2   // 2 = 2^1, could be >> 1
	b := x / 4   // 4 = 2^2, could be >> 2
	c := x / 8   // 8 = 2^3, could be >> 3
	d := x / 16  // 16 = 2^4, could be >> 4

	// These should NOT be detected (not powers of 2)
	e := x / 3
	f := x / 5
	g := x / 7

	return a + b + c + d + e + f + g
}

func main() {
	result := testDivisions()
	println("Result:", result)
}
```

## 第 6 步：运行并查看检测结果

```bash
../go/bin/go build test_divisions.go
```

**期望输出：**
```
[PowerOfTwo Detector] Function main.testDivisions: found 4 division(s) by power of 2
```

检测器找到了 4 处「除以 2 的幂」的除法。

## 第 7 步：用 Debug 输出做更细的验证

想看每次检测的详细信息：

```bash
GOSSAFUNC=testDivisions ../go/bin/go build -gcflags="-d=ssa/detect_div_by_power_of_two/debug=1" test_divisions.go
```

**期望输出：**
```
  [PowerOfTwo] Found division by power of 2: v10 / 2 (could be >> 1) at test_divisions.go:6
  [PowerOfTwo] Found division by power of 2: v14 / 4 (could be >> 2) at test_divisions.go:7
  [PowerOfTwo] Found division by power of 2: v18 / 8 (could be >> 3) at test_divisions.go:8
  [PowerOfTwo] Found division by power of 2: v22 / 16 (could be >> 4) at test_divisions.go:9
[PowerOfTwo Detector] Function main.testDivisions: found 4 division(s) by power of 2
```

可以看到精确位置和对应的移位位数！

## 我们学到了什么

- **SSA Pass 架构**：如何创建并注册编译器 pass
- **SSA 遍历**：如何遍历 block 和 value 来分析代码
- **操作检测**：识别特定的 SSA 操作
- **分析 vs 变换**：我们的 pass 目前只分析，还不改写（之后可以再做！）

## 扩展想法

可以继续尝试：

1. **真正实现优化**：把除法替换成移位
2. **检测乘以 2 的幂**：可以用左移代替
3. **统计优化总量**：统计整个构建过程中发现了多少处
4. **估算收益**：估算优化能省多少周期

## 清理

要移除自定义 pass：

```bash
cd go/src/cmd/compile/internal/ssa
rm powoftwodetector.go
# Edit compile.go and remove your pass from the passes array
cd ../../src
./make.bash
```

## 总结

你已经成功创建了一个自定义 SSA 编译器 pass，用来发现优化机会！

```
Pass Name:     "detect div by power of two"
Input:         SSA function representation
Analysis:      Finds x / (power of 2) operations
Output:        Reports potential optimizations
Location:      Early in compiler pipeline

Example:       x / 8  →  Reports: "could be >> 3"
```

这说明 Go 的编译器基础设施允许你插入自定义的分析和优化 pass。真正的优化 pass 用的也是同一套模式——它们只是会改写 SSA，而不只是报告。

---

*继续 [练习 7](07-runtime-patient-go.zh.md)，或返回 [workshop 主页](../README.md)*
