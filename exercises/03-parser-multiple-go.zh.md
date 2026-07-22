# 练习 3: 多个 "go" 关键字 — 增强 Parser

> 📖 **想深入了解？** 阅读 Internals for Interns 上的 [The Parser](https://internals-for-interns.com/posts/the-go-parser/)，深入理解 Go 的 parser 如何构建 Abstract Syntax Tree。

在本练习中，你将修改 Go 的 parser，让它接受连续多个 "go" 关键字来启动 goroutine！这会教你如何扩展 parser 逻辑，以处理重复出现的语法模式，同时保持语义行为不变。

## 学习目标

完成本练习后，你将能够：

- 理解 Go parser 的结构以及 token 是如何被消费的
- 知道如何修改 parser 逻辑来扩展语法
- 用可运行的代码验证 parser 的改动

## 简介：什么是 Parser？

Parser 是编译器的第二阶段，紧跟在 scanner 之后。Scanner 产出的是一串扁平的 token 流，而 parser 的工作是给这串 token **结构化**——构建一棵 **Abstract Syntax Tree（AST）**，用树形结构表达代码中的层次关系。

例如，一条 `go sayHello()` 语句会变成一个类型为 `CallStmt` 的树节点，其中 `Tok: _Go`，子节点则表示函数调用 `sayHello()`。Parser 知道：在看到 `go` token 之后，后面必须跟一个函数调用表达式——这就是语言的文法规则。

Go 的 parser 使用一种叫 **recursive descent（递归下降）** 的技术：每条文法规则（文件、声明、语句、表达式）对应一个函数，这些函数自上而下互相调用。入口 `fileOrNil()` 先解析 package 子句，再解析 import，最后是声明。每个声明里可以有语句，每个语句里可以有表达式。

Parser 通过 `p.next()` 逐个消费 token，用 `p.tok` 检查当前 token。Parser 源码位于 `go/src/cmd/compile/internal/syntax/parser.go`。

## 第 1 步：定位到 Parser

```bash
cd go/src/cmd/compile/internal/syntax
```

### 理解当前的 Parser 逻辑

我们先看看 parser 目前如何处理 "go" 语句。打开 `parser.go`，大约在第 2675 行：

```go
// go/src/cmd/compile/internal/syntax/parser.go:2673-2676
...
return s

case _Go, _Defer:
    return p.callStmt()
...
```

Parser 识别到 `_Go` token 后，会立刻调用 `p.callStmt()` 来处理 goroutine 的创建。

在 `parser.go` 第 977 行附近找到 `callStmt()` 方法。这里就是我们要加入「多个 go」逻辑的地方：

```go
// go/src/cmd/compile/internal/syntax/parser.go:976-985
// callStmt parses call-like statements that can be preceded by 'defer' and 'go'.
func (p *parser) callStmt() *CallStmt {
    if trace {
        defer p.trace("callStmt")()
    }

    s := new(CallStmt)
    s.pos = p.pos()
    s.Tok = p.tok // _Defer or _Go
    p.next()
    ...
}
```

关键两行是：`s.Tok = p.tok` 记录这是 "defer" 还是 "go" 语句，随后 `p.next()` 消费掉当前 token。

## 第 2 步：支持多个 "go"

我们需要修改 `callStmt()`：在保持语义不变的前提下，消费掉连续出现的多个 "go" token。

**编辑 `parser.go`：**

找到大约第 985 行调用 `p.next()` 的位置，在其后加入支持多个 "go" 的逻辑：

```go
// go/src/cmd/compile/internal/syntax/parser.go:982-990
s := new(CallStmt)
s.pos = p.pos()
s.Tok = p.tok // _Defer or _Go
p.next()

// Allow multiple consecutive "go" keywords (go go go ...)
if s.Tok == _Go {
    for p.tok == _Go {
        p.next()
    }
}

...
```

### 理解这段改动

- **`if s.Tok == _Go`**：只对 "go" 语句生效（不影响 "defer"）
- **`for p.tok == _Go`**：只要后面还是连续的 "go" token，就一直消费
- **`p.next()`**：跳过每一个额外的 "go" token
- **语义保持**：`s.Tok` 始终是 `_Go`，所以语义含义不变

## 第 3 步：重新编译编译器

用我们的改动重新构建 Go toolchain：

```bash
cd ../../../  # back to go/src
./make.bash
```

如果出现编译错误，检查并修正你的改动。

## 第 4 步：测试多个 "go" 关键字

写一个测试程序，验证多个 "go" 的语法是否生效：

```bash
mkdir -p /tmp/multiple-go-test
cd /tmp/multiple-go-test
```

创建 test.go 文件：

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
    fmt.Println("Testing multiple go keywords...")

    // Test regular single go
    go sayHello("single go")

    // Test double go
    go go sayHello("double go")

    // Test triple go
    go go go sayHello("triple go")

    // Test quadruple go
    go go go go sayHello("quadruple go")

    // Wait a bit to see output
    time.Sleep(100 * time.Millisecond)
    fmt.Println("All done!")
}
```

用你定制的 Go 运行测试程序：

```bash
/path/to/workshop/go/bin/go run test.go
```

你应该会看到类似这样的输出：

```
Testing multiple go keywords...
Hello from single go!
Hello from double go!
Hello from triple go!
Hello from quadruple go!
All done!
```

## 第 5 步：跑 Parser 测试

确认我们没有把 parser 搞坏：

```bash
cd /path/to/workshop/go/src
../bin/go test cmd/compile/internal/syntax -short
```

## 我们做了什么

1. **增强 Parser**：修改 `callStmt()`，让它能处理连续多个 "go" token
2. **消费 Token**：在第一个 "go" 之后用循环继续消费后续的 "go"
3. **保持语义**：多个 "go" 关键字仍然只创建**一个** goroutine
4. **改动精准**：只影响 "go" 语句，不影响 "defer"

## 你学到了什么

- **Parser 逻辑**：Go 如何把 token 序列处理成语句
- **Token 消费**：如何连续消费同一类型的多个 token
- **Parser 测试**：用多样化的用例验证 parser 改动

## 扩展思路

可以试试这些额外改动：

1. 给 "defer defer defer" 加类似支持（更有挑战！）
2. 加一个上限（例如最多连续 5 个 "go"）
3. 记录实际用了多少个 "go" 关键字，方便调试
4. 让多个关键字影响 goroutine 的优先级

## 下一步

你已经成功扩展了 Go 的 parser，让它能处理重复的语法模式。

在 [练习 4: 编译器 Inlining 参数](./04-compiler-inlining-parameters.zh.md) 中，我们会转向探索 Go 编译器的优化机制，学习如何调节 inlining 参数来控制 binary 体积。

## 清理

恢复原始的 Go 源码：

```bash
cd /path/to/workshop/go/src/cmd/compile/internal/syntax
git checkout parser.go
cd ../../../
./make.bash  # Rebuild with original code
```

## 总结

现在可以用多个 "go" 关键字启动 goroutine 了：

```go
// These are all equivalent and create exactly one goroutine:
go myFunction()
go go myFunction()
go go go myFunction() 
go go go go myFunction()

// The parser consumes all consecutive "go" tokens
// but the semantic behavior remains the same!
```

本练习展示了：在 parser 层面做改动，可以在不改变底层语言语义的前提下，加入富有表现力的语法糖。

---

*继续 [练习 4](04-compiler-inlining-parameters.zh.md)，或返回 [工作坊主页](../README.md)*
