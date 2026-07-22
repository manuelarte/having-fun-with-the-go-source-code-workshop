# 练习 0: 简介与环境搭建

欢迎来到 Go 源码工作坊！在这个入门练习中，你会搭建好开发环境，并熟悉 Go 源码仓库。

## 学习目标

完成本练习后，你将：

- 拥有一套可用的 Go 开发环境
- 知道如何获取 Go 源码

## 前置要求

请确保已安装：

- Git
- 至少 4GB 可用磁盘空间

## 第 1 步: 安装或升级 Go

**⚠️ 重要**：从源码构建 Go 需要系统中已有 Go 安装（版本 1.24 或更高）。这称为 bootstrapping——用已有的 Go 编译器来构建新的 Go。

### 检查当前 Go 版本

```bash
go version
# 应显示: go version go1.24 或更高
```

### 若未安装 Go 或版本过旧

如果你还没有安装 Go，或版本低于 1.24：

1. **下载 Go**：访问 <https://go.dev/dl/>，下载适合你操作系统的安装包
2. **安装 Go**：按对应平台的官方安装指南操作
3. **验证安装**：打开新终端并运行：

   ```bash
   go version
   # 应显示: go version go1.24 或更高
   ```

**安装帮助**：如需详细安装步骤，请参阅 [Go 官方安装指南](https://go.dev/doc/install)。

## 第 2 步: Clone Go 源码

接下来 clone 官方 Go 仓库。使用 `--depth 1` 可避免下载完整历史，clone 会快很多：

为保证整个工作坊版本一致，我们使用 Go 1.26.1。

```bash
git clone --depth 1 --branch go1.26.1 https://go.googlesource.com/go
cd go
```

## 第 3 步: 确认 Go 版本为 1.26.1

确认当前处于正确版本：

```bash
git describe --tags
# 应显示: go1.26.1
```

## 我们完成了什么

- 安装或验证了用于 bootstrapping 的 Go 1.24+
- 以 1.26.1 版本 clone 了官方 Go 仓库
- 环境已就绪，可以开始从源码构建 Go

## 下一步

很好！环境已经搭好。在 [练习 1: 原样编译 Go](./01-compile-go-unchanged.zh.md) 中，你会从源码构建 Go toolchain，并在构建过程中探索 Go 编译器的结构！
