# 练习 5: 改造 gofmt — 缩进与 AST 变换

> 📖 **想深入了解？** 阅读 Internals for Interns 上的 [The Parser](https://internals-for-interns.com/posts/the-go-parser/)，深入理解 Go 如何构建并使用 Abstract Syntax Tree。

在本练习中，你将修改 Go 的格式化工具 `gofmt`：把缩进从 tab 改成 4 个空格，并加入自定义 AST 变换，自动把字符串字面量和注释里的 "hello" 替换成 "helo"。这会教你 gofmt 如何工作、printer mode 如何控制缩进，以及如何把自定义变换接入 AST 处理流水线。

## 学习目标

完成本练习后，你将能够：

- 理解 gofmt 如何控制缩进和 printer mode
- 学会同时修改 gofmt 与 go/format 包中的格式化行为
- 理解 gofmt 如何通过操作 AST 处理 Go 源码
- 知道如何修改 AST 中的字符串字面量和注释
- 探索 Go 的 AST（Abstract Syntax Tree）结构
- 实现自定义的源码变换

## 简介：什么是 AST？

**Abstract Syntax Tree（AST）** 是源码的树形表示：每个节点对应一种语言构造——函数、声明、表达式、语句等。树结构刻画了层次关系：函数节点包含语句节点，语句节点又包含表达式节点，以此类推。

Parser（练习 2–3 中已涉及）从 token 流构建这棵树。但 AST 不只被编译器使用——`gofmt`、`goimports`、`go vet` 等工具也会把代码解析成 AST，进行操作后再打印回去。

Go 通过 `go/ast` 包（位于 `src/go/`）对外暴露 AST，它与编译器内部的 AST 是分开的。这个公共包提供了诸如 `*ast.File`（整个源文件）、`*ast.FuncDecl`（函数声明）、`*ast.BasicLit`（字符串或数字等字面量）、`*ast.Comment` 等类型。`ast.Inspect()` 可以遍历整棵树、访问每一个节点——我们正是要用它来查找并修改字符串字面量和注释。

## 背景：gofmt 如何工作

gofmt 大致经过这些阶段：

1. **Parse** → 把源码转成 AST（Abstract Syntax Tree）
2. **Transform** → 对 AST 应用格式化规则
3. **Print** → 把修改后的 AST 按指定缩进再打印成格式化源码

缩进行为由两个关键常量控制：

- **`tabWidth`** → 缩进宽度（默认：8）
- **`printerMode`** → 控制间距行为的标志：
  - `printer.UseSpaces` → 用空格做 padding
  - `printer.TabIndent` → 用 tab 做缩进
  - `printerNormalizeNumbers` → 规范化数字字面量

### AST 结构

Go 把源码表示成节点树；本练习会用到这两个节点：

- **`*ast.BasicLit`** → 字符串字面量、数字等
- **`*ast.Comment`** → 源码中的注释

## 第 1 步：定位到 gofmt 源码

```bash
cd go/src/cmd/gofmt
ls -la
```

关键文件：

- **`gofmt.go`** → 主程序逻辑与文件处理
- **`simplify.go`** → AST 简化变换

## 第 2 步：把缩进改成 4 个空格

在加入自定义变换之前，先让 gofmt 用 4 个空格而不是 tab 做缩进。

### 修改 gofmt.go

**编辑 `go/src/cmd/gofmt/gofmt.go`：**

在大约第 50 行找到这些常量（附近有注释 "Keep these in sync with go/format/format.go"）：

```go
const (
	tabWidth    = 8
	printerMode = printer.UseSpaces | printer.TabIndent | printerNormalizeNumbers
```

改成：

```go
const (
	tabWidth    = 4
	printerMode = printer.UseSpaces | printerNormalizeNumbers
```

**改动说明：**

- **`tabWidth`**：从 `8` 改为 `4`（每一级缩进 4 个空格）
- **`printerMode`**：去掉 `printer.TabIndent` 标志（不再用 tab，只用空格）

### 修改 go/format 包

`go/format` 包也需要同步修改，以保持行为一致。

**编辑 `go/src/go/format/format.go`：**

在大约第 29 行找到同样的常量（注释相同）：

```go
const (
	tabWidth    = 8
	printerMode = printer.UseSpaces | printer.TabIndent | printerNormalizeNumbers
```

改成：

```go
const (
	tabWidth    = 4
	printerMode = printer.UseSpaces | printerNormalizeNumbers
```

### 理解这些改动

- **`tabWidth = 4`**：每一级缩进使用 4 个空格
- **去掉 `TabIndent`**：没有这个标志时，printer 只用空格（不输出 tab 字符）
- **`UseSpaces`**：确保 padding 和对齐使用空格
- **两个文件必须一致**：gofmt 与 go/format 要用同一套设置，行为才一致

## 第 3 步：重新构建并测试缩进

```bash
cd ../../../  # back to go/src
./make.bash
```

创建测试文件 `indent_test.go`：

```go
package main

import "fmt"

func main() {
	if true {
		for i := 0; i < 10; i++ {
			fmt.Println(i)
		}
	}
}
```

测试新的缩进：

```bash
cd ..  # to go/ directory
./bin/gofmt indent_test.go
```

预期输出（注意每一级是 4 个空格）：

```go
package main

import "fmt"

func main() {
    if true {
        for i := 0; i < 10; i++ {
            fmt.Println(i)
        }
    }
}
```

现在每一级缩进使用 4 个空格，而不再是 tab。

## 第 4 步：加入 Hello→Helo 变换

**编辑 `gofmt.go`：**

在大约第 76 行（`usage()` 函数之后）加入这个变换函数：

```go
// transformHelloToHelo walks the AST and replaces "hello" with "helo"
// in string literals and comments.
func transformHelloToHelo(file *ast.File) {
	ast.Inspect(file, func(n ast.Node) bool {
		switch node := n.(type) {
		case *ast.BasicLit:
			// Handle string literals
			if node.Kind == token.STRING {
				if strings.Contains(node.Value, "hello") {
					node.Value = strings.ReplaceAll(node.Value, "hello", "helo")
				}
			}
		case *ast.Comment:
			// Handle comments
			if strings.Contains(node.Text, "hello") {
				node.Text = strings.ReplaceAll(node.Text, "hello", "helo")
			}
		}
		return true // continue traversing
	})
}
```

### 理解这段代码

- **`ast.Inspect()`** — 遍历 AST 中的所有节点
- **`*ast.BasicLit`** — 匹配字面量（含字符串）
- **`node.Kind == token.STRING`** — 确认是字符串（而不是数字）
- **`*ast.Comment`** — 匹配注释
- **`strings.ReplaceAll()`** — 执行替换

## 第 5 步：接入变换

**仍然在 `gofmt.go` 中：**

找到大约第 238 行的 `processFile` 函数。再找到大约第 263 行附近的 `if *simplifyAST` 代码块：

```go
	if *simplifyAST {
		simplify(file)
	}
```

紧接其后加入我们的变换：

```go
	if *simplifyAST {
		simplify(file)
	}

	// Apply our custom hello→helo transformation
	transformHelloToHelo(file)
```

## 第 6 步：重新构建 gofmt

```bash
cd ../../../  # back to go/src
./make.bash
```

## 第 7 步：一起测试两项修改

创建 `hello_test.go` 文件：

```go
package main

import "fmt"

func main() {
    // Say hello to everyone
    message := "hello world"
    greeting := "Say hello!"

    /* This is a hello comment block */
    fmt.Println(message)
    fmt.Println(greeting)

    // Another hello comment
    fmt.Printf("hello %s\n", "Go")
}
```

```bash
../go/bin/gofmt hello_test.go
```

预期输出（注意同时有 4 空格缩进 **以及** hello→helo 变换）：

```go
package main

import "fmt"

func main() {
    // Say helo to everyone
    message := "helo world"
    greeting := "Say helo!"

    /* This is a helo comment block */
    fmt.Println(message)
    fmt.Println(greeting)

    // Another helo comment
    fmt.Printf("helo %s\n", "Go")
}
```

两项改动都生效了：

1. 所有 "hello" 都被替换成 "helo"
2. 缩进使用 4 个空格而不是 tab

## 第 8 步：测试就地格式化

```bash
# Format and overwrite the file
../go/bin/gofmt -w hello_test.go

# Verify the changes
cat hello_test.go
```

文件已被永久改写：内容里是 "helo" 而不是 "hello"，缩进也是 4 个空格！

## 理解我们做了什么

1. **修改 Printer 设置**：调整 tabWidth 和 printerMode，改用 4 个空格
2. **同步两个包**：同时更新 gofmt 与 go/format，保证一致
3. **加入 AST Visitor**：编写函数遍历并修改 AST 节点
4. **模式匹配**：识别字符串字面量和注释
5. **文本替换**：修改节点值，把 "hello" 换成 "helo"
6. **接入流水线**：在 gofmt 处理过程中调用变换
7. **测试验证**：确认缩进与变换两项改动都生效

## 你学到了什么

- **Printer 配置**：gofmt 如何通过 tabWidth 和 printerMode 控制缩进
- **包之间的一致性**：为何 gofmt 与 go/format 必须保持同步
- **AST 操作**：如何遍历并修改 Go 的 Abstract Syntax Tree
- **改造工具**：如何在现有 Go 工具上叠加多项改动
- **代码变换**：实现系统性的源码修改
- **构建流程**：重新构建 Go toolchain 组件
- **测试**：验证自定义工具行为

## 扩展思路

可以试试这些额外改动：

1. 加一个命令行 flag 来启用/关闭变换
2. 支持多组词替换（hello→helo、world→universe）
3. 增加大小写敏感选项
4. 只替换完整单词（不替换词内部的子串）
5. 让 tabWidth 可通过命令行 flag 配置
6. 增加在 tab 与空格之间切换的选项

示例：增加 flag

```go
var replaceHello = flag.Bool("helo", false, "replace hello with helo")

// In processFile():
if *replaceHello {
    transformHelloToHelo(file)
}
```

## 清理

恢复原始 gofmt：

```bash
cd go/src/cmd/gofmt
git checkout gofmt.go
cd ../go/format
git checkout format.go
cd ../../../src
./make.bash
```

## 总结

你已经用两种强有力的方式成功改造了 gofmt！

```
Indentation:   tabs (8 width) → 4 spaces
Transformation: "hello world"  → "helo world"
                // Say hello    → // Say helo

Changes:  tabWidth=4 + remove TabIndent flag
         + ast.Inspect() → pattern match → replace text
```

你现在理解了 `gofmt`、`goimports`、`go fix` 这类工具在 printer 与 AST 两个层面上是如何工作的。

---

*继续 [练习 6](06-ssa-power-of-two-detector.zh.md)，或返回 [工作坊主页](../README.md)*
