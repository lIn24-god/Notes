
## 第一步：创建项目目录并初始化 Go Module

```bash
mkdir my-agent
cd my-agent
go mod init my-agent
```

---

## 第二步：安装 Eino 框架及必要的扩展包

根据第16节课件，你需要：

```bash
go get github.com/cloudwego/eino
go get github.com/cloudwego/eino-ext/components/model/ollama
```

如果你之后用到 OpenAI，再加：
```bash
go get github.com/cloudwego/eino-ext/components/model/openai
```

如果要用 DuckDuckGo 搜索工具（可选）：
```bash
go get github.com/cloudwego/eino-ext/components/tool/duckduckgo
```

> 注意：`eino-ext` 下的包可能会依赖一些传递包，正常 `go get` 会自动处理。

---

## 第三步：确认本地 Ollama 已安装并运行（或准备 OpenAI Key）

### 选择 A：Ollama（推荐，免费、本地）

1. 访问 [ollama.com](https://ollama.com) 下载安装。
2. 拉取一个支持工具调用的模型（如 `qwen2.5:7b` 或 `llama3.2`，`deepseek-r1:8b` 也可以，但它可能工具支持不好）：
   ```bash
   ollama pull qwen2.5:7b
   ```
3. 确认服务运行：
   ```bash
   ollama serve
   ```
   默认端口 `11434`。

### 选择 B：OpenAI

准备好 API Key，并设置环境变量：
```bash
export OPENAI_API_KEY="sk-xxx"
```

---

## 第四步：写一个最简单的测试代码（不使用模板，只调用 Generate）

创建 `main.go`：

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/cloudwego/eino-ext/components/model/ollama"
	"github.com/cloudwego/eino/schema"
)

func main() {
	ctx := context.Background()

	// 1. 创建 Ollama 模型
	cm, err := ollama.NewChatModel(ctx, &ollama.ChatModelConfig{
		BaseURL: "http://localhost:11434",
		Model:   "qwen2.5:7b", // 换成你拉取的模型名
	})
	if err != nil {
		log.Fatal(err)
	}

	// 2. 构造简单消息
	messages := []*schema.Message{
		schema.SystemMessage("你是一个有用的助手。"),
		schema.UserMessage("用一句话介绍什么是 Agent。"),
	}

	// 3. 调用生成
	resp, err := cm.Generate(ctx, messages)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println(resp.Content)
}
```

运行：
```bash
go run main.go
```

如果能正常输出一段文字，说明环境完全 OK。

---

## 第五步：下一步做什么？

成功跑通基础对话后，开始加 Tool：

1. 定义一个简单的 Tool（例如 `get_current_time`）。
2. 把 Tool 绑定到 ChatModel。
3. 使用 `react.NewAgent` 构建循环 Agent。
4. 改成 CLI 交互模式。

