# 练习 2: 为 Goroutine 添加 "=>" 箭头运算符

> 📖 **想深入了解？** 阅读 Internals for Interns 上的 [The Scanner](https://internals-for-interns.com/posts/the-go-lexer/)，深入理解 Go 的 lexer/scanner 如何工作。

在本练习中，你会为 Go 新增一个 "=>" 箭头运算符，作为启动 goroutine 的另一种写法。借此学习如何改 Go 的 scanner，让它识别新运算符，并映射到已有功能。

## 学习目标

完成本练习后，你将：

- 理解 Go 的 scanner 如何把运算符 token 化
- 知道如何给 Go 加新的运算符语法
- 修改 scanner 的词法分析逻辑
- 用可运行的代码验证 scanner 改动
- 成功扩展 Go 的运算符“词汇表”

## 简介: 什么是 Scanner？

Scanner（也叫 lexer）是编译器的第一阶段。它逐字符读取源码，把字符组合成 **token**——语言中最小的有意义单元，例如 keyword（`func`、`go`、`return`）、运算符（`+`、`==`、`=>`）、标识符（`myVariable`）和字面量（`"hello"`、`42`）。

例如，scanner 看到 `go sayHello()` 时，会产出这些 token：`_Go`、`_Name("sayHello")`、`_Lparen`、`_Rparen`。

Scanner 靠一个大的 `switch` 处理当前字符。当看到 `=` 时，会再看下一个字符：若还是 `=`，就产出 `==`（相等比较）token；否则产出 `=`（赋值）。这种前瞻（lookahead）机制，正是我们接下来要扩展来识别 `=>` 的地方。

Scanner 位于 `go/src/cmd/compile/internal/syntax/scanner.go`。

## 背景: 这次 Scanner 改造在做什么

本练习演示的是**在 scanner 层**给 Go 加新运算符语法。我们会改 scanner 逻辑，识别新的运算符序列 "=>"，并映射到已有 token。具体会做到：

- **增强 Scanner**：增加对 "=>" 运算符序列的识别
- **Token 映射**：把 "=>" 映射到已有的 `_Go` token（与 "go" keyword 相同）
- **替代语法**：让 `=> myFunction()` 等价于 `go myFunction()`
- **影响最小**：不需要改 parser 或编译器更深层——只动 scanner 逻辑

这样就能用很轻量的改动，做出更优雅的替代写法，而不必动编译器深处！

## 第 1 步: 进入 Scanner 目录

```bash
cd go/src/cmd/compile/internal/syntax
```

### 理解 Scanner 结构

看看 `scanner.go` 里如何处理 "=" 运算符。定位到第 325 行附近：

```go
// go/src/cmd/compile/internal/syntax/scanner.go:325
case '=':
    s.nextch()
    if s.ch == '=' {
        s.nextch()
        s.op, s.prec = Eql, precCmp
        s.tok = _Operator
        break
    }
    s.tok = _Assign
```

Scanner 先用 `s.nextch()` 消费掉 "="，再检查下一个字符是不是也是 "="（对应 `==` 比较运算符）。如果不是，就回退为 "="（赋值）。

## 第 2 步: 加上箭头运算符逻辑

我们需要增加识别 "=>" 的逻辑，并把它当成 `_Go` token。

**编辑 `scanner.go`：**

找到大约第 325 行的 "=" 分支，改成也检查 ">"：

```go
// go/src/cmd/compile/internal/syntax/scanner.go:325
case '=':
    s.nextch()
    if s.ch == '=' {
        s.nextch()
        s.op, s.prec = Eql, precCmp
        s.tok = _Operator
        break
    }
    if s.ch == '>' {
        s.nextch()
        s.lit = "=>"
        s.tok = _Go
        break
    }
    s.tok = _Assign
```

### 理解代码改动

- **`if s.ch == '>'`**：检查 "=" 之后的下一个字符是不是 ">"（注意 `s.nextch()` 已经消费掉了 "="）
- **`s.nextch()`**：从 lexer 中消费掉 ">" 字符
- **`s.lit = "=>"`**：设置字面量，便于调试和错误信息
- **`s.tok = _Go`**：赋成与 "go" keyword 相同的 token
- **`break`**：跳出 case，避免落到 `_Assign`

## 第 3 步: 重新构建编译器

带着改动重建 Go toolchain：

```bash
cd ../../../  # 回到 go/src
./make.bash
```

若有编译错误，检查并修正你的改动。

## 第 4 步: 测试新的箭头运算符

写一个测试程序，验证 "=>" 运算符可用：

```bash
mkdir -p /tmp/arrow-test
cd /tmp/arrow-test
```

创建 test.go：

```go
package main

import (
    "fmt"
    "time"
)

func sayHello(name string) {
    fmt.Printf("Hello from %s!\n", name)
}

func main() {
    fmt.Println("Testing => arrow operator...")

    // 测试普通 go keyword
    go sayHello("regular go")

    // 测试我们新加的 => 运算符
    => sayHello("arrow operator")

    // 稍等片刻，便于看到输出
    time.Sleep(100 * time.Millisecond)
    fmt.Println("All done!")
}
```

用你自定义的 Go 运行测试程序：

```bash
/path/to/workshop/go/bin/go run test.go
```

你应该会看到类似输出：

```
Testing => arrow operator...
Hello from regular go!
Hello from arrow operator!
All done!
```

## 第 5 步: 混合使用 Go 运算符

再测混合场景：同时用传统的 "go" keyword 和新的 "=>" 箭头运算符。

创建 mixed-test.go：

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func worker(id int, wg *sync.WaitGroup) {
    defer wg.Done()
    fmt.Printf("Worker %d starting\n", id)
    time.Sleep(50 * time.Millisecond)
    fmt.Printf("Worker %d done\n", id)
}

func main() {
    var wg sync.WaitGroup

    fmt.Println("Starting workers with mixed syntax...")

    // 混合使用普通 go 和 => 运算符
    for i := 1; i <= 4; i++ {
        wg.Add(1)
        if i%2 == 0 {
            go worker(i, &wg)  // 普通 go
        } else {
            => worker(i, &wg)  // 箭头运算符
        }
    }

    wg.Wait()
    fmt.Println("All workers completed!")
}
```

运行混合测试：

```bash
/path/to/workshop/go/bin/go run mixed-test.go
```

## 第 6 步: 运行 Scanner 测试

确认我们没有把 scanner 弄坏：

```bash
cd /path/to/workshop/go/src
../bin/go test cmd/compile/internal/syntax -short
```

## 我们做了什么

1. **改了 Scanner 逻辑**：在已有的 "=" 分支里增加对 "=>" 的识别
2. **复用已有 Token**：把 "=>" 映射到 `_Go`，而不是新建 token
3. **保留原有功能**："=" 和 "==" 仍然正常工作
4. **改动面很小**：不需要改 parser 或 IR

## 你学到了什么

- **Scanner 逻辑**：Go 如何对运算符序列做 token 化
- **运算符识别**：通过改 scanner 添加新运算符
- **Token 复用**：把新语法映射到已有 token
- **测试策略**：用真实代码验证 scanner 改动
- **构建流程**：带着 scanner 改动重建 Go

## 扩展想法

可以再试试这些改动：

1. 把 ":>" 也做成 "go" 的另一种写法
2. 用 "~>" 表示异步操作
3. 加一个 ">>>" 三箭头运算符
4. 让箭头运算符在更多上下文中生效

## 下一步

你已经成功给 Go 的 scanner 加了一个新运算符，也明白了如何通过改 scanner 为已有功能提供替代语法。同一套手法还可以用来做其他运算符快捷写法和语法糖。

在练习 3 中，我们会换一条路，探索 **parser 改造**——学习如何改 parser，让它处理连续的多个 token。

## 清理

恢复原始 Go 源码：

```bash
cd /path/to/workshop/go/src/cmd/compile/internal/syntax
git checkout scanner.go
cd ../../../
./make.bash  # 用原始代码重新构建
```

## 总结

"=>" 箭头运算符现在可以作为 "go" 的替代，用来启动 goroutine：

```go
// 下面两种写法现在等价：
go myFunction()
=> myFunction()

// 两者都以同样的方式创建 goroutine！
```

本练习展示了：在 scanner 层做改动，就能用很少的代码变更加入新语法。

---

*继续前往 [练习 3](03-parser-multiple-go.zh.md)，或返回 [工作坊主页](../README.md)*
