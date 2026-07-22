# 「玩转 Go 源码」工作坊

欢迎来到这个动手实践工作坊！你将学习如何修改和实验 Go 编程语言的源码。本工作坊会带你一步步理解、构建，并对 Go 编译器与 runtime 动手改造。

**本工作坊使用 Go 1.26.1** —— 我们会 checkout 对应的 release tag，保证所有练习环境一致。

## Languages / 语言

- English: [README.md](./README.md) · exercises as `*.md`
- Español: exercises as `*.es.md`
- 中文: [README.zh.md](./README.zh.md) · exercises as `*.zh.md`

## 前置要求

- 具备基础的 Go 编程知识
- 熟悉命令行工具
- 系统已安装 Git
- **Go 编译器版本 1.24 或更高**（bootstrap 构建过程需要）
- 至少 4GB 可用磁盘空间

## 工作坊概览

本工作坊共 12 个练习（练习 0–11），从源码构建 Go 开始，再到在编译器、工具链和 runtime 的不同位置动手修改。你会接触到 Go 内部实现的一些关键点，从 lexer、parser，到 runtime 行为：

### [练习 0：简介与环境搭建](./exercises/00-introduction-setup.zh.md)

克隆并搭建 Go 源码环境，迈出第一步。

### [练习 1：原样编译 Go](./exercises/01-compile-go-unchanged.zh.md)

学习在不做任何修改的情况下，从源码构建 Go toolchain。

### [练习 2：为 Goroutine 添加 "=>" 箭头运算符](./exercises/02-scanner-arrow-operator.zh.md)

通过添加 "=>" 作为启动 goroutine 的另一种语法，学习修改 scanner/lexer。

### [练习 3：多个 "go" 关键字 —— Parser 增强](./exercises/03-parser-multiple-go.zh.md)

通过支持连续多个 "go" 关键字（如 `go go go myFunction`），学习修改 parser。

### [练习 4：Inlining 参数 —— 函数内联实验](./exercises/04-compiler-inlining-parameters.zh.md)

通过修改函数 inlining 相关参数，探索 inliner 的行为。

### [练习 5：修改 gofmt —— 缩进与 AST 变换](./exercises/05-gofmt-ast-transformation.zh.md)

修改 gofmt，使其使用 4 个空格代替 tab，并加入自定义 AST 变换，将 "hello" 替换为 "helo"。

### [练习 6：SSA Pass —— 检测除以 2 的幂](./exercises/06-ssa-power-of-two-detector.zh.md)

编写自定义 SSA 编译器 pass，检测可优化为位移的「除以 2 的幂」运算。

### [练习 7：Patient Go —— 让 Go 等待所有 Goroutine](./exercises/07-runtime-patient-go.zh.md)

修改 Go runtime，使程序在退出前等待所有 goroutine 完成。

### [练习 8：Goroutine 休眠侦探 —— Runtime 状态监控](./exercises/08-goroutine-sleep-detective.zh.md)

在 Go scheduler 中加入日志，监控进入休眠的 goroutine。

### [练习 9：可预测的 Select —— 去掉 Select 中的随机性](./exercises/09-predictable-select.zh.md)

修改 Go 的 select 语句实现，使其变为确定性的，而不是随机选择。

### [练习 10：Java 风格 Stack Trace —— 让 Go Panic 看起来更眼熟](./exercises/10-java-style-stack-traces.zh.md)

把 Go 冗长的 stack trace 改成 Java 风格的格式。

### [练习 11：D&D Work Stealing —— 为 Goroutine 掷骰](./exercises/11-dnd-work-stealing.zh.md)

在 scheduler 的 work stealing 算法中加入掷骰：P 必须在 d20 上掷出大于 10 才能偷取 goroutine。

## 如何开始

1. 从 [练习 0](./exercises/00-introduction-setup.zh.md) 开始，搭建环境
2. 按顺序完成练习
3. 完成练习 1 之后，可以按兴趣挑选后续练习

## 仓库结构

```
.
├── README.md                 # 本文件（英文）
├── README.zh.md              # 中文说明
├── exercises/               # 各练习的 markdown 文件
│   ├── 00-introduction-setup.md
│   ├── 00-introduction-setup.zh.md
│   ├── 01-compile-go-unchanged.md
│   ├── 02-scanner-arrow-operator.md
│   └── ...
├── website-generator/       # 从 markdown 生成网站的 Go 程序
│   ├── main.go
│   ├── templates.go
│   └── README.md
├── website/                 # 生成的网站（HTML）
│   ├── index.html
│   ├── 00-introduction-setup.html
│   └── ...
├── Makefile                 # 构建自动化
└── go/                      # Go 源码（搭建环境时克隆）
```

## 网站生成器

本仓库包含一个 Go 程序，可自动从 markdown 练习文件生成静态网站。

### 生成网站

```bash
# 推荐使用 make
make website

# 或直接运行
cd website-generator
go run . -exercises ../exercises -output ../website
```

### 本地预览

```bash
# 启动本地 Web 服务器
make serve

# 然后在浏览器中打开 http://localhost:8000
```

网站生成器会：

- 使用 [blackfriday](https://github.com/russross/blackfriday) 将 markdown 转为 HTML
- 保留全部格式、emoji 与代码块
- 生成练习之间的导航
- 创建带练习概览的索引页
- 提供响应式 CSS 样式

更多细节见 [website-generator/README.md](website-generator/README.md)（中文版：[website-generator/README.zh.md](website-generator/README.zh.md)）。

## 成功小贴士

- 每个练习都慢慢来 —— 编译器内部并不简单！
- 不妨多逛逛 Go 源码，不必局限于练习要求
- 用 `git` 跟踪改动，必要时随时回退
- 用各种 Go 程序充分测试你的修改

## 参考资源

- [Go Compiler Overview](https://github.com/golang/go/tree/master/src/cmd/compile)
- [Go Language Specification](https://go.dev/ref/spec)
- [Go Runtime Documentation](https://pkg.go.dev/runtime)

### 视频参考

这些练习的思路来自我的分享：

- [Understanding the Go Compiler](https://www.youtube.com/watch?v=qnmoAA0WRgE) —— 深入 Go 编译过程
- [Understanding the Go Runtime](https://www.youtube.com/watch?v=YpRNFNFaLGY) —— 探索 Go runtime 系统

## 完成工作坊后

做完全部练习，你将能够：

- **从源码构建 Go**，并理解 bootstrap 过程
- **修改语言语法**，通过改变 scanner 与 parser 行为
- **定制开发工具**，如 gofmt 与编译器优化
- **实现 SSA 优化**，在编译器后端动手
- **修改 runtime 行为**，包括程序入口与 scheduler 监控
- **调整并发算法**，例如 select 语句的随机选择
- **定制错误报告**，做成 Java 风格的栈追踪格式

**恭喜！** 你将更有信心继续探索 Go 源码。这些知识能帮你：

- 开始为 Go 项目做一些小贡献
- 构建自定义语言变体和工具
- 理解语言与 runtime 设计中的一些取舍

## 贡献

发现问题、有改进想法，或想增加练习？欢迎 [提交 issue](https://github.com/jespino/having-fun-with-the-go-source-code-workshop/issues) 或 pull request！

## 许可证

本项目采用 MIT License —— 详见 [LICENSE](LICENSE) 文件。

---

**编码愉快，欢迎来到 Go 内部实现的世界！**
