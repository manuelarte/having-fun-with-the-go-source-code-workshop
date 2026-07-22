# 练习 10: Java 风格 Stack Trace —— 让 Go 的 Panic 看起来更眼熟

在本练习中，你会修改 Go 的 stack trace 格式，让它更像 Java。我们不再输出 Go 原有的 stack trace，而是做成 Java 风格的堆栈信息。

## 学习目标

完成本练习后，你将能够：

- 理解 Go 如何在 runtime 中格式化 stack trace
- 知道 panic 消息是在哪里生成的
- 修改 runtime 的输出格式

## 背景：Stack Trace 风格对比

我们要把 Go 的 stacktrace 格式：

```
panic: Something went wrong

goroutine 1 [running]:
main.methodC()
        /Users/dev/project/main.go:15 +0x45
main.methodB()
        /Users/dev/project/main.go:11 +0x23
main.methodA()
        /Users/dev/project/main.go:7 +0x12
```

改成这种 Java 风格：

```
Exception in thread "main" go.runtime.Panic: Something went wrong
    at main.methodC(main.go:15)
    at main.methodB(main.go:11)
    at main.methodA(main.go:7)
```

## 第 1 步：创建测试程序

创建 `stack_trace_demo.go` 文件：

```go
package main

import "fmt"

func methodC() {
    panic("Something went wrong")
}

func methodB() {
    methodC()
}

func methodA() {
    methodB()
}

func main() {
    fmt.Println("Starting the program...")
    methodA()
}
```

用当前的 Go 运行，观察原始 stacktrace 格式：

```bash
go run stack_trace_demo.go
```

## 第 2 步：定位 Runtime 相关文件

```bash
cd go/src/runtime
```

我们会改这些关键文件：
- **`panic.go`** —— Panic 头部消息
- **`traceback.go`** —— 栈帧格式化

## 第 3 步：修改 Panic 头部

**编辑 `panic.go`：**

找到大约第 734 行的 `printpanics` 函数。找到类似这样的代码：

```go
print("panic: ")
printpanicval(p.arg)
```

改成：

```go
print("Exception in thread \"main\" go.runtime.Panic: ")
printpanicval(p.arg)
```

## 第 4 步：去掉 Goroutine 头部

**编辑 `traceback.go`：**

找到大约第 1215 行的 `goroutineheader` 函数。在函数开头加一条 return：

```go
func goroutineheader(gp *g) {
    return  // Add this line to skip printing goroutine info
    level, _, _ := gotraceback()
    // ... rest of original code below (now unreachable)
}
```

## 第 5 步：改造栈帧格式

**继续在 `traceback.go` 中操作：**

找到大约第 945 行的 `traceback2` 函数。把 `gotraceback()` 调用注释掉（大约第 966 行）：

```go
gp := u.g.ptr()
// level, _, _ := gotraceback()  // Comment this out
var cgoBuf [32]uintptr
```

然后找到打印栈帧的位置（大约第 991–1008 行）。把整段替换掉：

```go
printFuncName(name)
print("(")
if iu.isInlined(uf) {
    print("...")
} else {
    argp := unsafe.Pointer(u.frame.argp)
    printArgs(f, argp, u.symPC())
}
print(")\n")
print("\t", file, ":", line)
if !iu.isInlined(uf) {
    if u.frame.pc > f.entry() {
        print(" +", hex(u.frame.pc-f.entry()))
    }
    if gp.m != nil && gp.m.throwing >= throwTypeRuntime && gp == gp.m.curg || level >= 2 {
        print(" fp=", hex(u.frame.fp), " sp=", hex(u.frame.sp), " pc=", hex(u.frame.pc))
    }
}
print("\n")
```

替换成这种 Java 风格格式：

```go
// Extract just the filename (not full path)
fileName := file
for i := len(file) - 1; i >= 0; i-- {
    if file[i] == '/' || file[i] == '\\' {
        fileName = file[i+1:]
        break
    }
}
print("    at ", name, "(", fileName, ":", line, ")\n")
```

## 第 6 步：重新编译 Go Runtime

```bash
cd ../  # back to go/src
./make.bash
```

## 第 7 步：测试 Java 风格 Stack Trace

```bash
../go/bin/go run stack_trace_demo.go
```

你应该看到：

```
Starting the program...
Exception in thread "main" go.runtime.Panic: Something went wrong
    at main.methodC(stack_trace_demo.go:6)
    at main.methodB(stack_trace_demo.go:10)
    at main.methodA(stack_trace_demo.go:14)
    at main.main(stack_trace_demo.go:19)
```

## 理解我们做了什么

1. **改了 Panic 头部**（`panic.go` 约第 747 行）：把 `"panic: "` 改成 `"Exception in thread \"main\" go.runtime.Panic: "`
2. **去掉了 Goroutine 信息**（`traceback.go` 约第 1215 行）：在 `goroutineheader()` 里提前 `return`
3. **简化了栈帧输出**（`traceback.go` 约第 991–1008 行）：把 Go 原有输出换成 Java 风格的 `"    at name(file:line)"`
4. **去掉了调试信息**：注释掉 `gotraceback()` 调用，并去掉十六进制偏移、frame pointer 等
5. **只保留文件名**：用循环从完整路径中截取 basename

## 我们学到了什么

- **Runtime 格式化**：Go 如何生成 stack trace
- **Panic 处理**：panic 消息从哪里来
- **输出控制**：如何改 runtime 的 print 语句

## 扩展想法

可以尝试这些额外改造：

1. 给输出加颜色（"Exception" 用红色）
2. 通过环境变量做成可配置
3. 再加一种 Python 风格格式
4. 把包路径转成类似 `github.com.user.pkg` 的形式（`github.com/user/pkg` → `github.com.user.pkg`）

## 清理

要恢复 Go 原始的 stack trace 格式：

```bash
cd go/src/runtime
git checkout panic.go traceback.go
cd ../
./make.bash
```

## 总结

你已经把 Go 的 stack trace 改成了 Java 风格：

```
// Before: Technical and verbose
goroutine 1 [running]:
main.methodC()
        /full/path/to/main.go:15 +0x45

// After: Clean and familiar
Exception in thread "main" go.runtime.Panic: ...
    at main.methodC(main.go:15)
```

---

*继续学习 [练习 11](11-dnd-work-stealing.zh.md)，或返回 [工作坊主页](../README.md)*
