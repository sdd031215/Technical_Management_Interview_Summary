# LangChain 面试真题 13 题 — Day2 扩展专题

> 来源：公众号「大志说编程」《Agent 面试真题 03：突击 LangChain 面试！13 个核心问题》
> 定位：本篇是 Day2（Agent 技术栈详解）的 **LangChain 专项补充**。
> Day2 主文件覆盖 Agent 通用架构、ReAct、Function Calling、Multi-Agent、可靠性；本篇聚焦 LangChain v1.x 的组件、LCEL、Structured Output、Tool Calling、MCP、框架选型。
> 适用题：「为什么用 LangChain」「Runnable / LCEL 是什么」「LangChain 怎么接 MCP」「LangChain vs LangGraph vs LlamaIndex 怎么选」

> **图标约定**：🎯 面试官追问 / ⚠️ 踩坑提示 / 💬 示例答

---

## Q1. 为什么选择使用 LangChain？

直接调用大模型 API 构建应用，会很快遇到三个问题：

- **Prompt 管理混乱** —— 提示词散落在代码各处，改一处漏一处，难以维护。
- **切换模型成本高** —— OpenAI、DeepSeek、Claude 各家 API 格式不同，切换模型要改大量代码。
- **复杂功能难维护** —— 多轮对话、工具调用、知识库检索等逻辑越写越乱。

LangChain 对模型封装、记忆管理、工具调用等模块做了抽象，提供统一接口：**切换模型只需改模型名称**，模块间交互与具体模型无关。

```python
# 切换模型只需要改这一个地方
llm = init_chat_model("gpt-4o-mini", temperature=0)
# 或者
llm = init_chat_model("deepseek-chat", temperature=0)
```

🎯 **面试官追问**：
- "既然原生 SDK 也能调，为什么非要用框架？" → 框架的价值在「抽象 + 复用 + 切换成本低」，项目小可以不用，项目大、模型多、要接工具/RAG 时收益才明显。
- "LangChain 的抽象有没有代价？" → 有：额外学习成本、版本升级破坏性变更（v0→v1 就是例子）、对底层控制力下降。

---

## Q2. LangChain 有哪些核心组件？

LangChain v1.0 主要包含六个核心模块：

### ① Models（模型）
统一不同厂商模型的调用方式。v1.0 把各模型集成包独立拆分（`langchain-openai`、`langchain-anthropic` 等），通过 `init_chat_model` 统一初始化：

```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("openai:gpt-5.4")
# 切换到 Claude，只改模型名称
llm = init_chat_model("anthropic:claude-sonnet-4-6")
```

### ② Agent（智能体）
v1.0 的核心。本质是 **模型在循环中调用工具，直到任务完成**。用 `create_agent` 快速创建，四个核心参数：

| 参数 | 作用 |
|------|------|
| `model` | 大语言模型 |
| `tools` | 可调用的工具列表 |
| `system_prompt` | 系统提示词 |
| `middleware` | 中间件，扩展 Agent 能力 |

```python
from langchain.agents import create_agent

agent = create_agent(
    model="claude-sonnet-4-6",
    tools=[search_web, analyze_data],
    system_prompt="你是一个专为程序员服务的智能助手"
)
```

`create_agent` 底层基于 LangGraph 构建，**天然支持** Checkpoint、Streaming、Human-in-the-loop、Time Travel。

### ③ Tools（工具）
让 Agent 与外部世界交互：实时数据、代码执行、数据库查询、API 调用。任何 Python 函数都能通过 `@tool` 装饰器定义为工具，函数参数类型作为输入 Schema，方法注释作为工具描述：

```python
from langchain.tools import tool

@tool
def search_database(query: str, limit: int = 10) -> str:
    """查询数据库匹配对应记录"""
    return f"Found {limit} results for '{query}'"
```

工具内部还能通过 `ToolRuntime` 访问运行时上下文：短期记忆（State）、长期记忆（Store）、用户上下文（Context）。

### ④ Middleware（中间件）
v1.0 最重要的扩展机制，在 Agent 执行各步骤插入自定义逻辑：

| 钩子 | 触发时机 | 典型用途 |
|------|----------|----------|
| `before_agent` | Agent 开始执行前 | 加载记忆、校验输入 |
| `before_model` | 每次 LLM 调用前 | 更新提示词、裁剪消息 |
| `wrap_model_call` | 包裹 LLM 调用 | 动态选择模型、拦截请求 |
| `wrap_tool_call` | 包裹工具调用 | 错误处理、动态工具注册 |
| `after_model` | 每次 LLM 响应后 | 输出校验、内容审查 |
| `after_agent` | Agent 结束后 | 保存结果、清理资源 |

内置常用中间件：

```python
from langchain.agents.middleware import (
    PIIMiddleware,              # PII 脱敏
    SummarizationMiddleware,    # 对话摘要压缩
    HumanInTheLoopMiddleware    # 人工审批
)

agent = create_agent(
    model="gpt-5.5",
    tools=[read_email, send_email],
    middleware=[
        PIIMiddleware("email", strategy="redact"),
        SummarizationMiddleware(model="claude-sonnet-4-6", trigger={"tokens": 500}),
        HumanInTheLoopMiddleware(interrupt_on={"send_email": True}),
    ]
)
```

### ⑤ Memory（记忆）
v1.0 通过 LangGraph 机制实现两种记忆：
- **短期记忆（State）**：基于 LangGraph 的 State 对象，存当前会话消息历史和自定义字段，会话结束即消失。
- **长期记忆（Store）**：基于 LangGraph 的 Store，以命名空间 + 键值对方式持久化，跨会话可用。

`SummarizationMiddleware` 可在对话过长时自动压缩历史，防止超出上下文窗口。

### ⑥ RAG 相关组件
通过 Document Loader、Text Splitter、Embedding、Vector Store、Retriever 等模块组合实现 RAG，让大模型基于向量检索匹配语义最相近的文本片段，减少幻觉。

⚠️ **踩坑**：v0.x 的 `ConversationBufferMemory` 等老 Memory 类在 v1.0 已被 LangGraph 的 State/Store 取代，面试别讲老 API。

---

## Q3. Runnable 是什么？

`Runnable` 是 LangChain 里最基础的执行单元。模型、Prompt 模板、Retriever、Output Parser，甚至一个普通函数，只要实现了 `Runnable` 类，就能用同一套方法调用：

| 方法 | 作用 |
|------|------|
| `invoke()` | 单次调用 |
| `batch()` | 批量调用 |
| `stream()` | 流式输出 |
| `ainvoke()` / `abatch()` / `astream()` | 异步版本 |

因为大部分组件都实现了 Runnable，LangChain 才能轻松实现 LCEL 表达式。

🎯 **面试官追问**：
- "Runnable 抽象的好处是什么？" → 统一接口 = 可组合、可流式、可异步、可批处理，组件之间天然能串起来。

---

## Q4. RunnableSequence 是什么？

把多个 Runnable 按顺序连接，前一个的输出自动作为后一个的输入。最常见的写法就是 LCEL：

```python
chain = prompt | model | output_parser
```

调用时依次执行：渲染 Prompt → 调用模型 → 解析输出。

**与 Agent 的区别**：Agent 自主决定下一步做什么；RunnableSequence 完全按顺序执行。**流程固定、步骤明确**的场景（问答、摘要、分类）用 RunnableSequence；需要动态决策用 Agent。

---

## Q5. LCEL 是什么？

LCEL（LangChain Expression Language）是 LangChain 提供的链式组合写法，核心是用管道符 `|` 将 Runnable 连起来。

```python
chain = (
    {"context": retriever, "question": lambda x: x["question"]}
    | prompt
    | model
    | parser
)
```

**解决的问题**：以前要把上一个组件的结果手动传给下一个，需要很多中间变量；现在管道符一表达，流程一目了然。

支持几种模式：
- **顺序执行**：`prompt | model | parser`
- **并行执行**：用 `RunnableParallel` 同时跑多个 Runnable，结果合并成字典
- **分支逻辑**：根据条件走不同链路

⚠️ **边界**：LCEL 适合步骤固定的场景。涉及循环、人工审批、断点恢复，用 LangGraph。

---

## Q6. Output Parser 的作用是什么？

把大模型返回的文本转换成特定数据结构。模型默认输出自然语言（"用户名是张三，年龄 28 岁"），程序更希望拿到结构化结果（`{"name": "张三", "age": 28}`）。

两个作用：
1. 把字符串解析成 JSON、对象、列表、枚举等结构化数据。
2. 检查返回数据是否符合预期格式（字段缺失、类型错误）。

⚠️ **趋势**：现在很多模型已支持原生 Structured Output 和 Tool Calling，比纯文本解析更稳定。Output Parser 在纯文本模型场景仍有用，但不再是首选。

---

## Q7. Structured Output 如何实现？

让模型按预定义结构返回结果。三种常见实现：

### 方式一：通过提示词定义结构
用 Pydantic / JSON Schema 描述字段，提示词里告诉 LLM 按 Schema 返回。

```python
from pydantic import BaseModel, Field

class UserInfo(BaseModel):
    name: str = Field(description="用户名")
    age: int = Field(description="年龄")
```

### 方式二：使用模型原生 Structured Output 能力
gpt、claude 等模型都支持。

```python
structured_llm = llm.with_structured_output(UserInfo)
result = structured_llm.invoke("提取用户信息：张三今年 28 岁")
# result 是 UserInfo 对象，不是字符串
```

### 方式三：使用 Tool Calling 实现结构化输出
把要结构化的对象结构作为工具入参，通过提示词让 LLM 调用工具，返回的工具调用参数就是结构化数据。

🎯 **面试官追问**：
- "三种方式怎么选？" → 模型支持原生 Structured Output 优先用方式二（最稳）；不支持的工具调用（方式三）；都不行才退回提示词 + Parser（方式一，最脆弱）。

---

## Q8. LangChain v1.x 与 v0.x 有哪些区别？

| 维度 | v0.x | v1.x |
|------|------|------|
| Agent 创建 | `AgentExecutor` / `initialize_agent` | `create_agent`（基于 LangGraph） |
| LangGraph 关系 | 与 LangChain 相对独立 | Agent 构建在 LangGraph 之上 |
| 包结构 | 后期开始拆分 | 深度拆分：`langchain` / `langchain-core` / `langchain-community` / `langchain-openai` / `langchain-anthropic` |
| 状态管理 | 弱 | LangGraph 提供 State、Checkpoint、断点恢复 |
| Memory | `ConversationBufferMemory` 等老类 | LangGraph 的 State（短期）+ Store（长期） |

**核心变化**：v1.x 中 LangGraph 成为 Agent 的底层基础。简单 Agent 用 LangChain；复杂流程、状态机、多 Agent 编排，直接用 LangGraph。

⚠️ **踩坑**：面试别再讲 v0.x 的 `initialize_agent`、`AgentExecutor`、`ConversationBufferMemory`，会被认为没跟进版本。

---

## Q9. LangChain 如何实现 Tool Calling？

让 LLM 在需要时调用外部工具。分五步：

**① 定义工具**（用 `@tool` 装饰器，必须有清晰的名称、参数、说明）：
```python
from langchain_core.tools import tool

@tool
def get_weather(city: str) -> str:
    """查询指定城市的天气。"""
    return f"{city} 今天晴，25 度。"
```

**② 绑定工具到模型**：
```python
llm_with_tools = llm.bind_tools([get_weather])
```

**③ 模型生成工具调用**（用户问"北京今天天气怎么样？"，模型返回）：
```json
{"name": "get_weather", "args": {"city": "北京"}}
```

**④ 执行工具**：LangChain 根据工具名和参数调用对应 Python 函数，返回 `"北京 今天晴，25 度。"`。

**⑤ 把工具结果交回模型**：工具结果包装成工具消息，模型生成最终回答。

💬 **用 `create_agent` 自动完成整个流程**：
```python
from langchain.agents import create_agent

agent = create_agent(
    model=llm,
    tools=[get_weather],
    system_prompt="你是一个天气助手"
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "北京今天天气怎么样？"}]
})
```

---

## Q10. LangChain 如何接入 MCP？

MCP（Model Context Protocol）的作用是把外部工具、服务以统一协议暴露给模型。**核心思路：把 MCP Server 提供的工具转换成 LangChain 工具，再交给 Agent 使用。**

**① 启动 MCP Server** —— 提供文件系统、数据库、浏览器、GitHub 等工具，通过 stdio / SSE / HTTP 通信。

**② 加载 MCP 工具**（先装 `langchain-mcp-adapters`）：
```python
from langchain_mcp_adapters.client import MultiServerMCPClient

client = MultiServerMCPClient({
    "math": {
        "command": "python",
        "args": ["./math_server.py"],
        "transport": "stdio",
    }
})

tools = await client.get_tools()
# tools 就是 LangChain 可识别的工具列表
```

**③ 将工具绑定到 Agent**：
```python
from langchain.agents import create_agent

agent = create_agent(
    model=llm,
    tools=tools,
    system_prompt="你可以使用外部工具完成任务"
)
```

**④ 调用 MCP 工具**：模型决定调用工具时，LangChain 通过适配器把调用转发给 MCP Server，结果返回给模型。

🎯 **MCP 的意义**：把工具**标准化**，不同 Agent 可共享，无需重复实现；工具本身可能是 Java/C#/Python 实现的，MCP 屏蔽实现细节。

---

## Q11. LangChain 和 LlamaIndex 有什么区别？

| 对比维度 | LangChain | LlamaIndex |
|----------|-----------|------------|
| 核心定位 | 构建 LLM 应用和 Agent | 构建数据索引和 RAG 系统 |
| 关注重点 | 模型调用、工具调用、Agent 编排、工作流 | 文档加载、索引构建、检索增强生成 |
| 适合场景 | Agent、多工具调用、复杂任务链路 | 企业知识库、文档问答、数据检索 |
| 抽象方式 | Model、Tool、Agent、Middleware、Memory | Document、Node、Index、Retriever、Query Engine |
| 编排能力 | 更强，尤其结合 LangGraph | 相对弱，更聚焦检索链路 |
| 数据处理 | 有 RAG 能力，但不是唯一核心 | 更强，特别是多数据源和索引策略 |

**实践中常组合使用**：LlamaIndex 负责文档解析、索引构建和检索；LangChain / LangGraph 负责任务编排、工具调用和 Agent 流程控制。

---

## Q12. 你用过哪些 Agent 框架？选型如何选？

常用框架：**LangChain / LangGraph、AutoGen、CrewAI**。

| 框架 | 优势 | 劣势 | 适用场景 |
|------|------|------|----------|
| **LangChain** | 快速构建 Agent，组件全（模型、工具、记忆、中间件） | 复杂流程控制力有限 | 典型 Agent 应用：工具调用、记忆、上下文管理 |
| **LangGraph** | 生产级，图结构显式定义 State/Node/Edge，支持 Checkpoint、断点恢复、人工审批、循环 | 学习曲线较陡 | 生产级 Agent：分支、循环、人工审批、断点恢复 |
| **AutoGen** | 多 Agent 对话协作（Planner/Coder/Reviewer/Executor） | 严格控制流程需额外开发 | 多 Agent 交互协作 |
| **CrewAI** | 简单，定义好 Agent/任务/流程就能跑 | 流程变复杂后可控性下降 | 多角色协作，快速验证 |

**选型原则**：
- 简单 LLM 调用或少量工具调用 → 不一定需要 Agent 框架，模型 SDK + 函数调用即可。
- 典型 Agent（工具、记忆、上下文）→ **LangChain**。
- 生产级 Agent（分支、循环、人工审批、断点恢复）→ **LangGraph**。
- 多角色协作和对话式任务拆解 → **AutoGen / CrewAI**。
- 知识库问答和 RAG → **LlamaIndex** + LangGraph 做流程编排。

🎯 **面试官追问**：
- "为什么生产环境优先 LangGraph？" → 流程可控、状态可持久化、可观测、可恢复。这些是生产 Agent 的硬需求，其他框架要么不支持要么要自己造。
- "CrewAI 简单为什么不用？" → 简单是优点也是限制。PoC 阶段用 CrewAI 快速验证；上生产、要加分支/状态/审批时，LangGraph 更合适。

---

## Q13. LangChain 和 LangGraph 有什么区别？

| 维度 | LangChain | LangGraph |
|------|-----------|-----------|
| 解决的问题 | Agent **能做什么** | Agent **应该怎么做** |
| 关注点 | 调用大模型、管理 Prompt、工具调用、RAG、结构化输出 | 任务拆分、下一步执行什么、失败重试、人工介入、中断恢复 |
| 定位 | 提供各种能力组件 | 把组件按特定流程组织起来 |
| 适合场景 | 简单问答机器人、知识库助手 | 多步骤任务、多 Agent 协作、复杂状态流转 |

**一句话理解**：LangChain 解决"Agent 能做什么"，LangGraph 解决"Agent 应该怎么做"。

**实践中配合使用**：多 Agent 系统中，Planner 制定计划、Researcher 搜索资料、Writer 生成报告 —— 整个执行流程由 LangGraph 控制；每个 Agent 内部调用大模型、搜索工具、向量数据库等能力，由 LangChain 提供。

🎯 **判断标准**：普通 RAG、Tool Calling、结构化输出 → LangChain 够了；多步骤任务、多 Agent 协作、复杂状态流转 → LangGraph 更合适。生产环境很多项目是 LangChain 构建 Agent + LangGraph 做流程编排。

---

## 速记卡片

| 题号 | 一句话答案 |
|------|-----------|
| Q1 | LangChain 解决 Prompt 管理、模型切换、复杂功能维护三大痛点 |
| Q2 | 六大核心组件：Models / Agent / Tools / Middleware / Memory / RAG |
| Q3 | Runnable 是最基础执行单元，提供 invoke/batch/stream 统一接口 |
| Q4 | RunnableSequence 用 `\|` 串多个 Runnable，流程固定 |
| Q5 | LCEL 用管道符组合 Runnable，适合步骤固定场景 |
| Q6 | Output Parser 把文本转结构化数据，正被原生能力取代 |
| Q7 | Structured Output 三种：提示词 / 原生能力 / Tool Calling |
| Q8 | v1.x：Agent 基于 LangGraph，包结构模块化，Memory 用 State/Store |
| Q9 | Tool Calling 五步：定义→绑定→生成→执行→回传 |
| Q10 | 接 MCP = 把 MCP Server 工具转成 LangChain 工具给 Agent |
| Q11 | LangChain 做 Agent，LlamaIndex 做 RAG，常组合 |
| Q12 | 选型：简单用 SDK，典型用 LangChain，生产用 LangGraph，协作用 AutoGen/CrewAI |
| Q13 | LangChain = 能做什么，LangGraph = 怎么做，生产常配合 |
