## 阶段性总结：从多工具实现到稳定 CLI Agent

### 🎯 这一阶段的目标
在前一阶段（单个时间工具 + ReactAgent 循环）的基础上，扩展为：
- 实现 **三个不同用途的工具**（计算器、添加待办、列出待办）
- 分别使用课件中的 **三种实现方式**（`InferTool`、`NewTool`、接口）
- 封装成 **CLI 交互式程序**，支持多轮对话和会话记忆

---

### ⚠️ 遇到的主要困难

| 困难现象 | 原因分析 | 解决路径 |
|---------|----------|----------|
| `add_todo` 工具调用时 JSON 解析失败，错误提示 `Mismatch type string with value object` | 模型（qwen2.5:7b 在 Ollama 上）工具调用的实现不稳定，输出参数不是标准 JSON 对象，或包含残缺字段（如 `{"content":"...","` 被截断） | ① 增加容错：先尝试解析 JSON，失败则把整个字符串当作 `content`；<br>② 最终放弃让模型输出 `deadline` 字段，将工具参数简化为只接收 `content`（字符串） |
| 模型倾向于不调用工具，直接回答“我不知道” | 系统提示不够明确，或模型对工具调用能力不自信 | 在 SystemMessage 中显式列出可用工具和调用场景，强调“必须使用工具获取实时信息” |
| 手动解析 `NewTool` 的 `argumentsInJSON` 时类型转换复杂 | `deadline` 可能为数字、字符串或缺失，导致 `json.Unmarshal` 到固定结构体失败 | 改用 `map[string]interface{}` 动态判断，并增加默认值；最终简化设计，去掉 `deadline` 参数 |
| 三种实现方式混用导致代码复杂度高，调试困难 | 接口实现需要额外定义结构体和方法，`NewTool` 需要手动处理 JSON | 统一使用 `InferTool`（最简单），让框架自动处理参数 schema 和解析，减少人为错误 |

---

### 💡 解决思路总结

1. **简化工具接口**  
   - 工具应只接收**必要参数**，复杂逻辑（如时间解析、默认值）交给工具内部处理，而不是依赖模型生成精确的时间戳。  
   - 例如：`add_todo` 只要求 `content` 字段，截止时间固定为“添加时间 + 24 小时”。

2. **统一使用最稳定的实现方式**  
   - 经过对比，`utils.InferTool` 对普通函数的支持最完善，自动生成 JSON schema，且无需手动处理参数 JSON。  
   - 因此最终三个工具全部采用 `InferTool`，放弃 `NewTool` 和接口实现（虽然课件展示了灵活性，但实战中稳定性优先）。

3. **增强系统提示（System Prompt）**  
   - 明确告诉模型每个工具的名称、用途、参数格式。  
   - 强调必须使用工具，并示例调用方式。

4. **增加对话历史管理**  
   - 每次用户输入加入 `messages`，Agent 回复也加入，实现短期记忆。  
   - 出错时回滚用户消息，避免历史污染。

---

### 🧠 最终版代码结构与讲解

#### 完整代码（已简化并稳定）

```go
package main

import (
	"bufio"
	"context"
	"fmt"
	"log"
	"os"
	"strings"
	"time"

	"github.com/cloudwego/eino/components/tool"
	"github.com/cloudwego/eino/components/tool/utils"
	"github.com/cloudwego/eino/compose"
	"github.com/cloudwego/eino/flow/agent/react"
	"github.com/cloudwego/eino/schema"
	"github.com/cloudwego/eino-ext/components/model/ollama"
)

// 1️⃣ 计算器工具（需要 a, b 两个数字）
type CalcInput struct {
	A float64 `json:"a" jsonschema:"required"`
	B float64 `json:"b" jsonschema:"required"`
}

func calcFunc(ctx context.Context, input CalcInput) (string, error) {
	sum := input.A + input.B
	return fmt.Sprintf("%.2f + %.2f = %.2f", input.A, input.B, sum), nil
}

func newCalcTool() tool.BaseTool {
	t, _ := utils.InferTool("calculator", "Add two numbers", calcFunc)
	return t
}

// 2️⃣ 添加待办工具（只接收 content，deadline 由内部默认）
type AddTodoInput struct {
	Content string `json:"content" jsonschema:"required,description=待办内容"`
}

var (
	todoStore  = make(map[int]string)
	nextTodoID = 1
)

func addTodoFunc(ctx context.Context, input AddTodoInput) (string, error) {
	deadline := time.Now().Add(24 * time.Hour).Format("01-02 15:04")
	item := fmt.Sprintf("#%d: %s (截止: %s)", nextTodoID, input.Content, deadline)
	todoStore[nextTodoID] = item
	nextTodoID++
	return fmt.Sprintf("✅ 已添加：%s", item), nil
}

func newAddTodoTool() tool.BaseTool {
	t, _ := utils.InferTool("add_todo", "Add a todo item", addTodoFunc)
	return t
}

// 3️⃣ 列出待办工具（无参数）
type ListTodoInput struct{}

func listTodoFunc(ctx context.Context, input ListTodoInput) (string, error) {
	if len(todoStore) == 0 {
		return "📋 暂无待办事项", nil
	}
	var sb strings.Builder
	sb.WriteString("📋 待办列表：\n")
	for id, item := range todoStore {
		sb.WriteString(fmt.Sprintf("%s\n", item))
	}
	return sb.String(), nil
}

func newListTodoTool() tool.BaseTool {
	t, _ := utils.InferTool("list_todo", "List all todo items", listTodoFunc)
	return t
}

// ----------------------------
// CLI Agent 主程序
// ----------------------------
func main() {
	ctx := context.Background()

	// 创建 Ollama 模型（确保本地已拉取 qwen2.5:7b）
	cm, err := ollama.NewChatModel(ctx, &ollama.ChatModelConfig{
		BaseURL: "http://localhost:11434",
		Model:   "qwen2.5:7b",
	})
	if err != nil {
		log.Fatal(err)
	}

	// 组装 Agent
	agent, err := react.NewAgent(ctx, &react.AgentConfig{
		ToolCallingModel: cm,
		ToolsConfig: compose.ToolsNodeConfig{
			Tools: []tool.BaseTool{
				newCalcTool(),
				newAddTodoTool(),
				newListTodoTool(),
			},
		},
		MaxStep: 5,
	})
	if err != nil {
		log.Fatal(err)
	}

	// 初始化对话历史（短期记忆）
	messages := []*schema.Message{
		schema.SystemMessage("You are a helpful assistant. Use tools: calculator (add two numbers), add_todo (content only), list_todo (no arguments). Respond in Chinese."),
	}

	fmt.Println("🤖 Agent CLI 已启动，输入 /exit 退出。")
	scanner := bufio.NewScanner(os.Stdin)

	for {
		fmt.Print("\n> ")
		if !scanner.Scan() {
			break
		}
		userInput := scanner.Text()
		if userInput == "/exit" {
			fmt.Println("再见！")
			break
		}
		if userInput == "" {
			continue
		}

		// 添加用户消息
		messages = append(messages, schema.UserMessage(userInput))

		// 调用 Agent
		resp, err := agent.Generate(ctx, messages)
		if err != nil {
			fmt.Printf("❌ 错误: %v\n", err)
			// 移除刚添加的用户消息，避免破坏历史
			messages = messages[:len(messages)-1]
			continue
		}

		// 输出回复
		fmt.Printf("\n🤖: %s\n", resp.Content)

		// 保存助手回复到历史
		messages = append(messages, resp)
	}
}
```

---

### 🔍 代码逐段讲解

#### 1. 工具定义部分（三个工具）

- **计算器**：定义结构体 `CalcInput` 包含 `A` 和 `B`，`jsonschema:"required"` 表示这两个字段必须由模型提供。`utils.InferTool` 会根据结构体自动生成参数描述，模型调用时输出 `{"a": 12.5, "b": 68.3}`。
- **添加待办**：结构体 `AddTodoInput` 只有 `Content` 字段，模型只需输出 `{"content": "今晚11点睡觉"}`。工具内部使用 `todoStore`（内存 map）存储，并自动分配 ID 和默认截止时间（当前+24小时）。
- **列出待办**：空结构体 `ListTodoInput`，模型调用时输出 `{}` 或不传参数。工具遍历 `todoStore` 并返回格式化的列表。

#### 2. Agent 创建

- `react.NewAgent` 需要传入 `ToolCallingModel`（支持工具调用的模型）和 `ToolsConfig`（工具列表）。`MaxStep` 限制最多执行 5 次“思考-行动”循环，防止死循环。

#### 3. CLI 循环与记忆

- 使用 `bufio.Scanner` 读取标准输入，支持多行（但这里每行一个命令）。
- `messages` 切片保存了完整的对话历史：SystemMessage + 所有用户消息 + 所有助手回复。每次调用 `agent.Generate` 时传入完整历史，模型就能记住上下文。
- 如果调用出错（例如模型输出格式异常），我们**回滚**最近添加的用户消息，避免错误消息污染历史导致后续全错。

#### 4. 为什么最终选择全部使用 `InferTool`？

- `InferTool` 利用 Go 的反射和结构体 tag，自动生成 JSON schema，模型按此格式输出即可，**无需手动处理参数解析**。
- `NewTool` 和接口实现需要手动 `json.Unmarshal`，且容易因参数类型变化而 panic。
- 在模型工具调用能力不完美的情况下，减少手动代码就是减少 bug 来源。

---

### 📌 关键经验教训

| 教训 | 说明 |
|------|------|
| **不要高估模型输出 JSON 的能力** | 本地小模型（如 qwen2.5:7b）在 Ollama 上的工具调用经常输出不完整的 JSON，应尽量简化参数结构。 |
| **工具参数越少越稳定** | 能不要的参数就不要，复杂逻辑（如时间解析）放在工具内部，而不是交给模型生成。 |
| **统一实现方式降低心智负担** | 虽然课件讲了三种方式，但实战中优先选择最简单、最自动化的 `InferTool`。 |
| **错误恢复机制很重要** | Agent 调用可能失败，必须设计好历史回滚，否则会话无法继续。 |

---

### 🚀 后续可扩展方向

- **持久化记忆**：将 `messages` 和 `todoStore` 保存到本地文件，支持 `/resume` 恢复历史会话。
- **流式输出**：使用 `agent.Stream` 实现逐字输出，提升交互体验。
- **集成更多工具**：天气查询、网络搜索、代码执行等，依然遵循“参数极简”原则。
- **RAG 知识库**：让 Agent 读取本地文档回答特定领域问题。

