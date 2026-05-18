## 阶段性总结：为 Agent 集成向量 RAG 知识库

### 🎯 本阶段目标

在前一阶段（多工具 CLI Agent）的基础上，引入 **RAG（检索增强生成）** 能力，让 Agent 能够：
- 读取本地文档（`.txt` / `.md`）
- 将其向量化并存入本地向量数据库
- 根据用户问题检索相关片段
- 基于检索到的内容生成更准确、可溯源的回答

核心挑战：**在保持 Agent 原有工具调用能力的前提下，无缝嵌入 RAG 流程**。

---

### 🧩 实现路径

1. **选择 Embedding 模型**：`nomic-embed-text`（Ollama 原生支持，768 维）
2. **选择向量数据库**：`chromem-go`（纯 Go 嵌入，无需额外服务）
3. **文档处理**：按段落分块（~1000 字符），持久化到本地目录
4. **新增工具**：`search_docs`，接收查询词，返回最相似的文档片段
5. **修改 Agent 指令**：强制要求模型在回答知识类问题时先调用 `search_docs`

---

### ⚠️ 遇到的主要困难与解决方案

| 困难                                  | 原因                                                    | 解决方案                                                                  |
| ----------------------------------- | ----------------------------------------------------- | --------------------------------------------------------------------- |
| `chromem-go` API 不匹配                | `go get` 拉取了新版本（v0.7.0），而文档教程基于 v0.2.0。               | 回退到 `github.com/philippgille/chromem-go@v0.2.0`，或查阅最新版源码调整调用方式。       |
| 计算器工具模型把数字传成字符串 `{"a":"233"}`       | 本地小模型（qwen2.5:7b）对 JSON Schema 类型校验不严格。               | 将工具参数改为 `expression` 字符串，由工具自己解析 `"233+9890"`。                        |
| 二次 JSON 编码导致 `Unmarshal` 失败         | 模型输出的 `arguments` 外层被多转义一次（`"{"content":"abc"}"`）。    | 在工具内部增加容错：先尝试解析为对象，失败则把整个字符串当原始输入。                                    |
| RAG 检索结果不相关（问时间答错）                  | ①分块策略太简单（仅按空行切）；②`nomic-embed-text` 中文语义弱；③`topK` 太小。 | ①改用更合理分块（保留重叠、按标题切）；②换 `bge-m3` 或 `mxbai-embed-large`；③调高 `topK` 到 5。 |
| 模型不遵守“仅从文档回答”，检索失败仍编造               | 系统提示约束不够强。                                            | 强化指令：“如果检索内容中没有答案，必须回答‘未找到相关信息’，禁止编造。”                                |
| 无法判断错误环节（检索失败还是生成失败）                | 工具返回只有文档内容，没有状态信息。                                    | 工具返回时附带检索到几个结果，例如 `找到 0 个相关文档` 便于定位。                                  |

---

### 🧠 最终架构（代码结构）

```
main.go
├── 工具定义（InferTool 方式）
│   ├── calculator          // 计算器
│   ├── add_todo            // 添加待办
│   ├── list_todo           // 列出待办
│   └── search_docs         // RAG 检索核心
├── 知识库加载
│   ├── loadKnowledgeBase()  // 启动时读取 ./kb 目录
│   ├── splitIntoChunks()    // 文档分块
│   └── chromem-go 集合      // 向量存储
├── Agent 创建
│   └── react.NewAgent(tools, model)
└── CLI 循环
    └── 对话历史管理 + 错误回滚
```

**关键依赖：**
- `github.com/cloudwego/eino`（Agent 框架）
- `github.com/cloudwego/eino-ext/components/model/ollama`（本地 LLM）
- `github.com/philippgille/chromem-go`（向量数据库）
- Ollama 中的 `nomic-embed-text`（embedding 模型）

---

### 🔍 核心代码片段讲解

#### 1. 知识库加载与向量化

```go
ragDB, _ = chromem.NewPersistentDB("./kb_db")
ragCollection, _ = ragDB.GetOrCreateCollection("knowledge_base", nil,
    chromem.NewEmbeddingFuncOllama("nomic-embed-text"))
```

- `NewPersistentDB`：将向量数据持久化到磁盘，重启不丢失。
- `NewEmbeddingFuncOllama`：自动调用 Ollama 的 `/api/embeddings` 接口生成向量。

#### 2. 文档分块

```go
func splitIntoChunks(text string, maxChunkSize int) []string {
    // 按双空行切段落，逐段累积到接近 maxChunkSize
}
```

- 保留语义边界（段落），避免切断句子。
- 可进一步优化：重叠分块、标题感知分块。

#### 3. `search_docs` 工具

```go
type SearchDocsInput struct {
    Query string `json:"query" jsonschema:"required,description=搜索查询词"`
}
func searchDocsFunc(ctx context.Context, input SearchDocsInput) (string, error) {
    results, _ := ragCollection.Query(input.Query, 3, nil, nil)
    // 格式化返回，附带来源和内容
}
```

- 极简参数（仅 `query` 字符串），避免模型输出复杂 JSON。
- 返回结果包含文档片段和文件名，便于模型引用。

#### 4. 系统提示强化

```go
schema.SystemMessage(`...
重要规则：
1. 如果用户询问的问题涉及你自己的知识库或文档内容，必须先调用 search_docs 工具检索相关信息，然后再回答。
2. 如果知识库中找不到答案，请如实告诉用户。
...`)
```

- 明确要求“先检索后回答”，约束力强。

---

### 📈 效果对比

| 场景 | 无 RAG | 有 RAG（本地 nomic-embed） | 有 RAG + 闭源模型 |
|------|--------|----------------------------|-------------------|
| 问“深蓝时间几点到几点” | 胡乱编造 | 能检索到文档，但可能回答 9:00-18:00（错误） | 能正确回答 8:00-20:00 |
| 问计算题 | 经常传错参数类型 | 同左 | 稳定正确 |
| 问私有文档内容 | 完全不知道 | 能检索，但摘要可能不准确 | 准确概括 |

---

### 💡 关键经验总结

1. **本地小模型做 RAG 是可行的，但需要反复调优**  
   - 分块策略、Embedding 模型质量、topK 值都直接影响最终效果。

2. **工具参数越简单越稳定**  
   - 尽量用字符串代替结构化对象，在工具内部自己解析。

3. **向量数据库选型要轻量**  
   - `chromem-go` 非常合适 Go 语言嵌入式场景，但注意版本锁定。

4. **调试手段要充足**  
   - 在工具返回中携带检索状态（如找到几条结果），便于区分检索失败还是生成失败。

5. **闭源 API 可以大幅降低模型行为层面的问题**  
   - 如果追求稳定性和最终考核效果，建议切换到 DeepSeek 或 OpenAI。

---

### 🚀 后续可扩展方向

- **升级 Embedding 模型**：换用 `bge-m3` 或 `mxbai-embed-large` 提升中文检索精度。
- **改进分块策略**：实现 RecursiveCharacterTextSplitter（按段落→句子→字符逐级切）。
- **增加 Rerank 阶段**：对检索结果重排序，提升 top1 准确率。
- **持久化对话记忆**：保存历史消息到文件，支持 `/resume` 恢复会话。
- **切换闭源 LLM API**：仅需替换 `NewChatModel` 组件，上层代码零改动。

