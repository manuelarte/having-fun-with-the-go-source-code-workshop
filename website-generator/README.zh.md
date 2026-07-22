# 网站生成器

这个 Go 程序会根据 markdown 练习文件生成工作坊网站。

## 功能

- 将 markdown 练习转换为 HTML 页面
- 生成带练习概览的索引页
- 内置 CSS 样式
- 自动导航链接（上一页 / 下一页）
- 保留全部 markdown 格式与代码块
- 修正相对链接，使其在 HTML 中可用

## 用法

### 基本用法

```bash
# 在 website-generator 目录下
go run . -exercises ../exercises -output ../website
```

### 安装

```bash
# 安装依赖
go mod download

# 构建生成器
go build -o website-generator

# 运行构建好的二进制
./website-generator -exercises ../exercises -output ../website
```

### 命令行参数

- `-exercises` - 练习目录路径（默认：`../exercises`）
- `-output` - 输出目录路径（默认：`../website`）

### 示例

```bash
# 生成到默认输出目录
go run .

# 生成到自定义位置
go run . -output /path/to/output

# 使用自定义练习目录
go run . -exercises /path/to/exercises -output /path/to/output
```

## 工作原理

1. **读取 Markdown 文件**：扫描 exercises 目录中的所有 `.md` 文件
2. **转换为 HTML**：使用 [blackfriday](https://github.com/russross/blackfriday) 将 markdown 转为 HTML
3. **应用模板**：用带导航与样式的 HTML 模板包装内容
4. **修正链接**：将相对 markdown 链接转换为 HTML 链接
5. **生成索引**：创建列出全部练习的索引页
6. **复制 CSS**：包含样式表

## 项目结构

```
website-generator/
├── main.go          # 主程序逻辑
├── templates.go     # HTML 与 CSS 模板
├── go.mod          # Go module 定义
└── README.md       # 本文件
```

## 依赖

- [blackfriday v2](https://github.com/russross/blackfriday) - Markdown 处理器

## 生成结果

生成器会创建：

- `index.html` - 带练习概览的首页
- `00-introduction-setup.html` 到 `10-java-style-stack-traces.html` - 各练习页面
- `style.css` - 样式表

## 自定义

### 练习元数据

编辑 `main.go` 中的 `exerciseMetadata` 数组，可自定义：

- 练习标题
- Emoji
- 描述
- 文件名

### 模板

修改 `templates.go` 中的模板：

- `exerciseTemplate` - 单个练习页布局
- `indexTemplate` - 首页布局
- `cssTemplate` - 样式

### Markdown 处理

可自定义 `markdownToHTML()` 函数以添加：

- 自定义 markdown 扩展
- 后处理步骤
- 链接转换

## 重新生成网站

修改 markdown 文件后：

```bash
cd website-generator
go run .
```

网站会重新生成到 `../website` 目录。

## 开发

### 测试

```bash
# 生成本地并测试
go run . -output /tmp/test-website
open /tmp/test-website/index.html
```

### 添加新练习

1. 将 markdown 文件加入 `../exercises/`
2. 在 `main.go` 的 `exerciseMetadata` 数组中添加元数据
3. 运行生成器
4. 检查输出结果

## 许可证

属于 "Having Fun with the Go Source Code Workshop"（「玩转 Go 源码」工作坊）项目的一部分。
