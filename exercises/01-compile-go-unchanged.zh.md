# 练习 1: 原样编译 Go

在本练习中，你会学习如何在不做任何修改的情况下，从源码构建 Go toolchain。这是我们动手改语言之前必须掌握的基本功！

## 学习目标

完成本练习后，你将：

- 理解 Go 的构建流程与 bootstrap 概念
- 成功从源码编译 Go
- 知道如何探索 Go 源码结构
- 知道如何测试自己构建的 Go

## 第 1 步: 理解 Bootstrap 过程

Go 本身就是用 Go 写的！这就带来了“先有鸡还是先有蛋”的问题——没有 Go，怎么编译 Go？答案就是 bootstrapping：

1. Go 团队提供预编译的二进制
2. 用这些二进制编译当前的 Go 源码
3. 新编译出来的版本再用于后续开发

先确认你已安装 Go（bootstrapping 需要）：

```bash
go version
# 必须显示 1.24 或更高版本
```

**⚠️ 关键**：构建 Go 1.26.1 必须安装 Go 1.24 或更高版本。若未安装或版本过旧，请从 <https://golang.org/dl/> 安装最新版。

## 第 2 步: 进入 Go 源码目录

```bash
cd go/src
pwd
# 应显示: /path/to/workshop/go/src

# 确认当前 Go 版本正确
git describe --tags
# 应显示: go1.26.1
```

## 第 3 步: 开始构建

Go 提供了多种构建脚本。我们先用 `make.bash` 构建 toolchain，构建过程中可以顺便逛逛源码！

**在类 Unix 系统上（Linux、macOS）：**

```bash
./make.bash
```

**在 Windows 上：**

```cmd
make.bat
```

该脚本会：

1. 构建 Go toolchain（编译器、链接器、runtime、标准库）
2. 首次构建会从头编译所有内容，视机器性能大约需要 2–10 分钟
3. 之后的构建会快很多，因为只需重新编译改动过的文件及其依赖

### `all.bash` 和 `run.bash` 是做什么的？

你可能会注意到 `src/` 目录下还有其他脚本：

- **`make.bash`**：只构建 Go toolchain（我们正在用的）
- **`run.bash`**：跑完整测试套件（需要先构建好 Go）
- **`all.bash`**：便捷脚本，依次执行 `make.bash` + `run.bash`，并打印构建信息

对本工作坊来说，`make.bash` 最合适，因为：

- 构建时间更短，少等一会儿
- 我们只需要一个能用的 Go 来做实验
- 需要时再单独用 `run.bash` 跑测试即可

## 第 4 步: 构建过程中探索源码

构建跑着的时候，打开**新终端**或 **IDE**，一起看看 Go 源码结构吧！这正是理解“我们在构建什么”的好时机。

**在新终端中：**

```bash
cd /path/to/workshop/go  # 进入你的 Go 源码目录
ls -la
```

### 仓库结构

你应该能看到这些关键目录：

- **`src/`**：Go 源码所在
  - `src/cmd/`：命令行工具（go、gofmt 等）——其中包括 `cmd/compile/`，也就是我们后面要改的编译器代码
  - `src/runtime/`：Go runtime 系统
  - `src/go/`：面向开发者工具的 Go 语言包（parser、AST 等）——编译器本身不用这些包
- **`test/`**：Go 语言相关测试
- **`api/`**：API 兼容性数据
- **`doc/`**：文档

### 查看 Go 编译器结构

再仔细看看 `src/cmd/compile/`：

```bash
cd src/cmd/compile
ls -la
```

关键文件与目录：

- **`main.go`**：编译器入口
- **`internal/`**：编译器内部包
  - `internal/syntax/`：把源码变成 token（scanner），再构建语法树（parser）
  - `internal/types2/`：类型检查（例如不能把 string 和 int 相加）
  - `internal/ir/`：中间表示（Intermediate Representation）——解析和类型检查之后，编译器对程序的内部模型，用于分析和优化，再生成机器码
  - `internal/ssa/`：Static Single Assignment 形式——把 IR 变成更底层的表示，每个变量只赋值一次，便于做死代码消除、常量传播等强力优化
  - `internal/gc/`：编排整个编译流水线，协调从解析到机器码生成的各个阶段

## 第 5 步: 理解构建输出

**切回原来正在构建的终端**。构建推进时，你会看到类似输出：

```
Building Go cmd/dist using /usr/local/go. (go1.26.1 darwin/amd64)
Building Go toolchain1 using /usr/local/go.
Building Go bootstrap cmd/go (go_bootstrap) using Go toolchain1.
Building Go toolchain2 using go_bootstrap and Go toolchain1.
Building Go toolchain3 using go_bootstrap and Go toolchain2.
Building packages and commands for darwin/amd64.
```

逐行含义：

1. **`Building Go cmd/dist using /usr/local/go`**：先构建 `dist`，一个管理后续构建流程的小工具。用系统里的 Go（`/usr/local/go`）来编译它。

2. **`Building Go toolchain1 using /usr/local/go`**：系统 Go 编译 Go 1.26.1 的编译器源码，得到 `toolchain1`——新编译器的第一版，但是由旧版 Go 构建出来的。

3. **`Building Go bootstrap cmd/go (go_bootstrap) using Go toolchain1`**：用 `toolchain1` 构建 `go_bootstrap`，这是管理后续构建步骤所需的精简版 `go` 命令。

4. **`Building Go toolchain2 using go_bootstrap and Go toolchain1`**：现在 `toolchain1` 开始“编译自己”——再次编译同一份 Go 1.26.1 编译器源码，但这次用的是新编译器，而不是系统 Go。结果是 `toolchain2`。

5. **`Building Go toolchain3 using go_bootstrap and Go toolchain2`**：`toolchain2` 再编译同一份源码，得到 `toolchain3`。因为 `toolchain2` 和 `toolchain3` 都由等价编译器从同一源码构建，二进制应当一致——用来验证构建可复现。

6. **`Building packages and commands for darwin/amd64`**：最后用验证过的 toolchain 编译标准库以及所有 Go 工具（`go`、`gofmt` 等），目标平台为你当前的系统。

## 第 6 步: 找到编译出的 Go 二进制

编译成功后，新的 Go 二进制位于：

```bash
ls -la /path/to/workshop/go/bin
```

你应该能看到：

- `go` - 主 Go 命令
- `gofmt` - Go 格式化工具

## 第 7 步: 测试自定义 Go 构建

来测一下刚编译好的 Go：

```bash
# 查看你编译出的 Go 版本
../bin/go version
```

在临时目录（例如 `/tmp`）创建一个 hello.go：

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello from my custom Go build!")
}
```

```bash
# 用你自定义的 Go 编译并运行
/path/to/workshop/go/bin/go run /tmp/hello.go
```

## ⚠️ 故障排查

### GOROOT 干扰

若运行 `../bin/go run /tmp/hello.go`（或二进制完整路径）结果异常，或实际用的是系统 Go 而不是新构建的版本，可能需要先取消 `GOROOT` 环境变量：

```bash
unset GOROOT
/path/to/workshop/go/bin/go run /tmp/hello.go
```

原因是系统 Go 安装可能设置了 `GOROOT`，导致新二进制指向了错误的标准库和工具。取消后，二进制会根据自身位置自动检测 root 目录。

## 你学到了什么

- **Bootstrap 过程**：Go 用已有的 Go 安装来编译自己
- **Go 源码结构**：组织清晰，职责分明（cmd/、runtime/ 等）
- **构建流程**：`./make.bash` 会把该构建的都构建好

## 下一步

恭喜！你已经拥有一套从源码构建的可用 Go toolchain。

接下来可以任选下列练习，深入了解 Go 的不同部分：

- [练习 2: 为 Goroutine 添加 "=>" 箭头运算符](./02-scanner-arrow-operator.zh.md) - Scanner 改造
- [练习 3: 多个 "go" 关键字 - Parser 增强](./03-parser-multiple-go.zh.md) - Parser 改造
- [练习 4: 内联参数 - 函数 Inlining 实验](./04-compiler-inlining-parameters.zh.md) - 编译器参数
- [练习 5: gofmt 变换 - "hello" 变成 "helo"](./05-gofmt-ast-transformation.zh.md) - AST 变换
- [练习 6: SSA Pass - 检测除以 2 的幂](./06-ssa-power-of-two-detector.zh.md) - SSA 编译器 pass
- [练习 7: 耐心的 Go - 让 Go 等待 Goroutine](./07-runtime-patient-go.zh.md) - Runtime 改造
- [练习 8: Goroutine 睡眠侦探 - Runtime 状态监控](./08-goroutine-sleep-detective.zh.md) - 调度器监控
- [练习 9: 可预测的 Select - 去掉 Select 的随机性](./09-predictable-select.zh.md) - Select 行为
- [练习 10: Java 风格 Stack Trace - 让 Go Panic 更眼熟](./10-java-style-stack-traces.zh.md) - 错误格式
- [练习 11: D&D Work Stealing - 为 Goroutine 掷骰](./11-dnd-work-stealing.zh.md) - 调度器 work stealing

或返回 [工作坊主页](../README.md) 选择练习。
