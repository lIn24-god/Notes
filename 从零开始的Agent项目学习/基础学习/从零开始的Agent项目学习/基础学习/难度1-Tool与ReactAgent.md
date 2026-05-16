## 添加一个 Tool，并用 ReactAgent 跑通“决策‑执行‑再回答”的循环

### 🎯 目标
在基础的 LLM 对话之上，使模型能够自主决定**调用外部工具**（获取当前时间），并将工具结果融入最终回答，形成「思考 → 行为 → 观察 → 回答」的 Agent 闭环。

---

### 📦 关键实现步骤

1. **定义一个普通 Go 函数**作为工具的业务逻辑  
   ```go
   func getCurrentTime(ctx context.Context, args noArgs) (string, error) {
       now := time.Now().Format("2006-01-02 15:04:05")
       return fmt.Sprintf("当前时间：%s", now), nil
   }
   ```

2. **使用 `utils.InferTool` 将函数包装成 Eino 的 Tool**  
   ```go
   timeTool, err := utils.InferTool("get_current_time", "获取当前的日期和时间", getCurrentTime)
   ```

3. **创建 `react.Agent`**，绑定模型和工具  
   ```go
   agent, err := react.NewAgent(ctx, &react.AgentConfig{
       ToolCallingModel: cm,
       ToolsConfig:      compose.ToolsNodeConfig{Tools: []tool.BaseTool{timeTool}},
       MaxStep:          5,
   })
   ```

4. **执行 Agent**，输入包含“现在什么时间”的用户消息，Agent 自动完成工具调用循环。

---

***完整代码***  :
```go
package main  
  
import (  
    "context"  
    "fmt"    "log"    "time"  
    "github.com/cloudwego/eino-ext/components/model/ollama"    "github.com/cloudwego/eino/components/tool"    "github.com/cloudwego/eino/components/tool/utils"    "github.com/cloudwego/eino/compose"    "github.com/cloudwego/eino/flow/agent/react"    "github.com/cloudwego/eino/schema")  
  
type noArgs struct{}  
  
func getCurrentTime(ctx context.Context, args noArgs) (string, error) {  
    now := time.Now().Format("2006-01-02 15:04:05")  
    return fmt.Sprintf("当前时间：%s", now), nil  
}  
  
func createTimeTool() tool.BaseTool {  
    timeTool, err := utils.InferTool("get_current_time", "获取当前日期和时间", getCurrentTime)  
    if err != nil {  
       log.Fatal("创建工具失败:", err)  
    }  
    return timeTool  
}  
  
func main() {  
    ctx := context.Background()  
  
    cm, err := ollama.NewChatModel(ctx, &ollama.ChatModelConfig{  
       BaseURL: "http://localhost:11434",  
       Model:   "qwen2.5:7b",  
    })  
    if err != nil {  
       log.Fatal(err)  
    }  
  
    timeTool := createTimeTool()  
  
    agent, err := react.NewAgent(ctx, &react.AgentConfig{  
       ToolCallingModel: cm,  
       ToolsConfig: compose.ToolsNodeConfig{  
          Tools: []tool.BaseTool{timeTool},  
       },  
       MaxStep: 5,  
    })  
    if err != nil {  
       log.Fatal(err)  
    }  
  
    messages := []*schema.Message{  
       schema.SystemMessage("你是一个有帮助的助手，可以调用工具获取信息。"),  
       schema.UserMessage("请问现在是什么时间？"),  
    }  
  
    resp, err := agent.Generate(ctx, messages)  
    if err != nil {  
       log.Fatal(err)  
    }  
  
    fmt.Println("最终回答：", resp.Content)  
}
```

### ⚠️ 遇到的困难与解决方法

| 问题现象 | 原因 | 解决方案 |
|---------|------|----------|
| `utils.InferTool` 编译错误：`无法推断T`、`未使用的形参 'ctx'` | `InferTool` 要求的函数签名为 `func(ctx context.Context, input T) (output U, err error)`。直接写 `func(ctx context.Context) (string, error)` 缺少第二个输入参数，导致泛型推断失败。 | 定义一个**空的输入结构体**（如 `type noArgs struct{}`），让函数接收第二个参数。这样既满足签名要求，又代表“无实际输入参数”。 |
| 部分模型（如 `deepseek-r1:8b`）在 Ollama 上不支持工具调用 | 并非所有模型都实现了 Function Calling 能力，或者 Ollama 对该模型的支持不完整。 | 更换为明确支持工具的模型，例如 `qwen2.5:7b` 或 `llama3.2`。 |
| Agent 直接回答“我不知道时间”而没有调用工具 | 系统提示不够强，或模型倾向于不信任工具。 | 在 SystemMessage 中强调“必须使用工具获取实时信息”，并将用户问题改为显式请求“请调用 get_current_time 工具”。 |

---

### ✅ 最终成果

- 成功运行代码，Agent 首先输出包含工具调用信息的中间步骤，最终输出类似：
  ```
  最终回答： 当前时间是 2025-05-11 15:30:22。
  ```
- 证明 Agent 具备了**自主决策、调用工具、依据结果生成回答**的能力。

---

### 💡 后续可扩展方向

- 将此逻辑封装为 **CLI 交互式工具**，支持多轮对话和会话记忆。
- 添加更多工具（天气查询、计算器、网络搜索等）。
- 接入 RAG 知识库，让 Agent 能回答基于本地文档的问题。

这一步是 Eino Agent 开发的核心里程碑，后续的复杂功能都建立在这个基础之上。