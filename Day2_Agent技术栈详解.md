# Day 2 — Agent 技术栈详解

> 目标：系统掌握 LLM Agent 核心架构、设计模式、框架选型，能在面试中画出 Agent 架构图并讲清每个关键决策
> 建议时间分配：上午 3h 精读问答 + 下午 2h 框架对比与代码 + 晚上 1h 自测

---

## 🔗 与主文件《3 周版》的联动（避免重复，先读这里）

本文件是 Day2 的**深化执行手册**。Agent 技术原理在主文件附录：

| 你要做的 | 去哪找 | 本文件补充什么 |
|---------|--------|---------------|
| 查 Agent 技术深度 | 主文件 **附录 A.2**（七个子节，含框架对比） | ❌ 不重复 |
| 查 Agent 高频面试题 | 主文件 **附录 B.2**（Q7-Q12） | ✅ 补面试官追问 + 踩坑 |
| 查生产可靠性六件套 | 主文件 **附录 A.2.5** | ✅ 补现场答法 |
| 查多 Agent 协作架构 | 主文件 **附录 A.2.6** | ✅ 补选型决策 |
| 做半日时段练习 | 主文件 **第一周 Day 3**（已含本主题半日表） | ✅ 本文件做"题库深化" |

**阅读顺序建议**：先过主文件附录 A.2 建立框架 → 用本文件做面试场景演练。

---

## 今日学习路线

| 时段 | 内容 | 产出 |
|------|------|------|
| 上午 9:00-12:00 | 精读 Q1-Q10（架构/ReAct/Function Calling/Multi-Agent/可靠性） | Agent 架构思维导图 |
| 下午 14:00-16:00 | 精读 Q11-Q20（评估/框架对比/MCP/生产实践/反思机制） | 框架选型决策表 |
| 晚上 19:00-20:00 | 闭卷自测 + 薄弱环节标记 | 自测记录 |

> **图标约定**：🎯 面试官追问 / ⚠️ 踩坑提示 / 💬 示例答

---

## 核心面试题详解

### Q1. 什么是 LLM Agent？它与普通 LLM 调用有什么本质区别？

**答案：**

LLM Agent 是以大语言模型为"大脑"，具备自主感知环境、制定计划、调用工具、并根据反馈迭代行动的自治系统。

| 维度 | 普通 LLM 调用 | LLM Agent |
|------|--------------|-----------|
| 交互模式 | 单轮或多轮 prompt → response | 自主循环：感知→思考→行动→观察 |
| 工具使用 | 无，纯文本生成 | 可调用外部 API、数据库、代码执行器 |
| 记忆 | 仅上下文窗口 | 具备短期/长期记忆系统 |
| 规划能力 | 无显式规划 | 可将复杂任务分解为子任务序列 |
| 自主性 | 被动响应 | 主动决策，可根据中间结果调整策略 |

Agent 的核心架构包含四大模块：**Planning（规划）**、**Memory（记忆）**、**Tools（工具）**、**Action（行动）**，形成"感知→规划→执行→观察"的循环。

工程实践中，Agent 的核心挑战不在于 LLM 本身有多强，而在于**如何让 LLM 可靠地编排工具调用序列**。

🎯 **面试官追问**：
- "Agent 的'自主性'会不会失控？怎么平衡？" → 用状态机约束路径 + max_steps 硬上限 + 关键节点 human-in-loop
- "你说四大模块 Planning/Memory/Tools/Action，哪个最难做？" → 通常是 Memory（什么写入、什么召回、什么遗忘，没标准答案）
- "Agent 和 Workflow（固定流程编排）什么区别？" → Workflow 路径确定，Agent 路径由 LLM 动态决定；生产常把两者结合
- 详细见主文件附录 **A.2.1**

⚠️ **踩坑提示**：
- 别把 Agent 讲成"会调工具的 LLM"——核心是**自主循环 + 规划**
- 别忽视可靠性——"Agent 很强"不是答案，"Agent 怎么不闯祸"才是面试官关心的

---

### Q2. ReAct 模式的工作原理是什么？优势和局限性？

**答案：**

ReAct（Reasoning + Acting）将"思维链（CoT）"与"外部工具调用"交织进行：

```
Thought: 我对当前状态的理解和下一步计划
Action:  调用某个工具（如 search("某关键词")）
Observation: 工具返回的结果
Thought: 基于观察结果的新一轮推理
...
Final Answer: 最终结论
```

**优势**：可解释性强、减少幻觉（实时工具调用）、灵活性高

**局限性**：
- 效率低：每个 Action 都需要一次完整 LLM 调用
- 上下文窗口压力：多轮累积快速消耗 Token
- 缺乏全局规划：贪心式推理，面对复杂任务效果有限
- 错误传播：某步出错，后续推理链全部偏斜

**工程优化**：Reflexion 回溯修正、Scratchpad 历史压缩、Plan-then-ReAct 先规划后执行

🎯 **面试官追问**：
- "ReAct 错误传播怎么解决？" → Reflexion 反思 + 关键步骤 Critic 校验 + 从 checkpoint 回滚
- "ReAct 上下文爆炸怎么办？" → Scratchpad 压缩历史、只保留最近 N 步 + 摘要、用外部记忆替代上下文
- "LangChain 已经把 ReAct Agent 标记 deprecated，你怎么看？" → 文本解析式 ReAct 不稳定，现在转向 tool_calls 隐式驱动（见 Q7.6）
- 详细见主文件附录 **A.2.1** 设计模式表

⚠️ **踩坑提示**：
- 别贬低 ReAct——它仍是基础模式，"已过时"说法不准确，是"演进"
- "上下文窗口压力"要量化（"每步累积几百 token，10 步就过万"）
- ReAct 适合探索性任务，明确多步任务用 Plan-and-Execute 更稳

---

### Q3. Plan-and-Execute 模式与 ReAct 有何不同？

**答案：**

Plan-and-Execute 将"规划"与"执行"显式分离为两个独立阶段：

| 特性 | ReAct | Plan-and-Execute |
|------|-------|------------------|
| 规划粒度 | 每步局部决策 | 全局先规划，后执行 |
| 适合任务 | 简单查询、单步工具调用 | 复杂多步骤任务 |
| 错误恢复 | 依赖下一步 Thought | 可触发 Re-planning |
| Token 效率 | 低（每步都带完整历史） | 较高（Executor 可用更小模型） |
| 架构复杂度 | 低 | 高（需 Planner + Executor） |

**适用场景**：多工具协作的复杂任务、有明确可分解结构的任务、需要审计和合规检查的生产环境

---

### Q4. Function Calling 的底层实现机制是什么？

**答案：**

Function Calling 并非 LLM 真正"理解"了 API，而是通过**特殊训练 + 结构化输出约束**实现的工程机制：

**层次一：训练阶段**
模型在 SFT 阶段接受大量工具调用样本训练，学会在什么场景下输出 `tool_calls` 格式、如何提取参数。

**层次二：推理阶段**
```
用户消息 → LLM → 检测 tool_calls 标记
  ├─ 普通文本回复
  └─ tool_calls JSON → Runtime 解析 → 调用函数 → 返回值注入上下文 → LLM 继续生成
```

**层次三：框架侧**
- `tool_choice="auto"` 让模型自主决定是否调用
- `strict: true` 强制输出符合 JSON Schema
- 工具 `description` 的质量直接决定调用准确率

🎯 **面试官追问**：
- "strict mode 的 JSON Schema 有什么限制？" → 要 `additionalProperties: false`、所有字段 required、不能无限嵌套
- "训练阶段怎么造工具调用样本？" → 人工标注 + 真实 API trace + 数据增强（同义改写）
- "Function Calling 和 MCP 什么关系？" → 不同层次，Function Calling 是模型能力，MCP 是工具集成协议（见 Q14）
- 详细见主文件附录 **A.2.2**（Tool Use）

⚠️ **踩坑提示**：
- 别说"模型理解了 API"——是**训练 + 结构化约束**的工程机制
- description 质量是隐形杀手——描述不清，调用准确率断崖式下降
- "strict mode"不是万能——有些复杂 schema 它校验不了

---

### Q5. 如何设计高质量的 Tool Schema？

**答案：**

**原则一：Description 是 Prompt，不是注释**
```json
// 糟糕：{"description": "获取用户信息"}
// 优秀：{"description": "根据用户ID查询用户详细信息（姓名、邮箱、订阅状态）。当用户询问账户信息或需要验证身份时调用。"}
```

**原则二：参数设计遵循"最小必要"原则**，合理设置默认值

**原则三：枚举优于自由文本**，用 `enum` 大幅降低无效参数概率

**原则四：工具数量控制与分层**
- 工具超过 15-20 个时，模型选择准确率显著下降
- 解决方案：工具路由、工具分组、工具嵌套

**常见坑**：同名歧义、参数类型错误、并行调用冲突、错误返回处理不当

---

### Q6. Multi-Agent 架构有哪些主流协作模式？

**答案：**

四种核心拓扑：

1. **顺序协作（Pipeline）**：Agent A → B → C，线性传递。适合内容生产流水线。
2. **层级协作（Supervisor）**：主管 Agent 分配任务、汇总结果。适合软件开发团队、客服路由。
3. **辩论/共识协作（Debate）**：多 Agent 各抒己见达成共识。适合决策支持、代码 Review。
4. **动态路由（Dynamic Routing）**：Router 根据请求内容选择 Expert Agent。适合智能客服。

**选型决策树**：
```
任务可线性分解？ → Sequential Pipeline
需要中央调度？ → Hierarchical Supervisor
需要多视角辩论？ → Debate
否则 → Dynamic Routing
```

---

### Q7. AutoGen、CrewAI、LangGraph 三大框架的本质区别？

**答案：**

| 框架 | 架构本质 | 核心优势 | 核心劣势 | 适用场景 |
|------|---------|---------|---------|---------|
| **AutoGen** | 对话驱动的多 Agent | 代码沙箱执行、Human-in-loop | 容易死循环 | 代码生成、科研探索 |
| **CrewAI** | 角色驱动的流水线 | 配置式开发、代码量少 | 灵活性受限 | 内容创作、标准化流程 |
| **LangGraph** | 图驱动的状态机 | 完全可控、支持断点回溯 | 学习曲线陡 | 生产级复杂工作流 |

**选型建议**：
- 简单 RAG → LlamaIndex
- 单 Agent + 工具调用 → LangChain
- 复杂多步工作流 → LangGraph
- 多 Agent 协作 → LangGraph / AutoGen
- 生产级部署 → LangGraph + LangGraph Platform

🎯 **面试官追问**：
- "为什么生产推荐 LangGraph 而不是 CrewAI？" → 显式状态机可控、支持 checkpoint 断点续跑、可观测性好；CrewAI 适合原型不适合生产
- "框架选型最看重的三个维度？" → 可控性（能否约束路径）、可观测性（trace 是否完整）、可恢复性（断点续跑）
- "框架不是架构"这句话怎么理解？" → 框架是脚手架，生产 Agent 的核心矛盾是 LLM 不可控 vs 业务确定性，重点在状态机/回滚/兜底，不在选哪个框架
- 详细见主文件附录 **A.2.3** 和 **B.11**

⚠️ **踩坑提示**：
- 别只报菜名"LangGraph 最好"——要讲**你的选型决策依据**
- 别忽视"框架不是架构"——这是 Tech Lead 视角的加分点
- CrewAI 不是"差"，是"不适合生产"，要说清 tradeoff

---

### Q7.5（面试深度追加）："面试官追问：LangGraph vs CrewAI vs AutoGen，能不能举一个具体场景，同时说清楚三个框架的表现差异？"

**答案：**

面试官要的不是"功能列表对比"，而是"你在一个真实需求面前怎么思考"。

**场景**：用户说"帮我调研 5 家竞品的定价策略，生成一份对比分析报告"。

| 加工方式 | LangGraph | CrewAI | AutoGen |
|---------|-----------|--------|---------|
| 任务编排 | 显式定义状态图：Research → Cluster? → Analyze → Write → Critic → (Re-plan?) | 定义 Researcher + Analyst + Writer + Reviewer 角色，按顺序执行 | 定义 Agent，Register 到 GroupChat，Manager Agent 动态选谁说话 |
| 条件分支 | 支持：竞品 > 10 走 Cluster，< 3 要求补搜 | 不支持运行时分支，只能提前定义顺序 | 支持（Manager 动态选），但不可控，可能选错 Agent |
| 执行可控性 | 完全确定，图定义了所有可能路径 | 基本确定，顺序流水线 | 低：对话式，Agent 可能聊飞 |
| 状态管理 | TypedDict 类型安全状态，每步可 Checkpoint | 自然语言在 Crew 成员间传递 | 自然语言消息传递，无结构状态 |
| 断点恢复 | 支持，LangGraph Checkpoint | 不支持 | 不支持 |
| 上手速度 | 需要 1-2 天理解 StateGraph | 30 分钟就能跑通 | 1 小时 |
| 适合谁 | 需要精确控制、需要审计、需要断点恢复的生产环境 | 快速验证一个多角色协作的想法 | 开放式探索，不确定最佳协作模式时 |

**面试表达**：

> "我会这样选：如果这个需求是业务方说'先跑一下看看效果'——我直接用 CrewAI，30 分钟搭好，下午就能出第一份报告。如果业务方说'以后要每天自动生成，质量要稳定，审计要能追溯每一步'——我会用 LangGraph 重建，虽然多花 2-3 天，但后续维护成本和线上事故率会大幅降低。我不会为了炫技在原型阶段用 LangGraph，也不会在生产环境用 CrewAI 冒险。选框架不是选最牛的，是选当前阶段最合适的。"

---

### Q7.6（新增）："2026 年 Agent 框架领域还有什么新变化值得关注？"

> "三个趋势：
>
> 1. **显式 ReAct 被逐步废弃**：LangChain 已经把 ReAct Agent 标记为 deprecated，推荐用 create_agent + LangGraph。OpenClaw、DeerFlow 2.0 也放弃了文本解析式的 ReAct loop，转向隐式状态的 tool_calls 驱动循环。
>
> 2. **Loop Engineering 成为独立方向**：当底层 Agent Loop 收敛后，竞争焦点上移到'谁能更好管理状态、规划与长期执行'——也就是 Harness Engineering。这不是框架之争，是工程方法之争。
>
> 3. **MCP 统一了工具接入层**：以前选 LangGraph 还是 CrewAI 要同时操心工具怎么接，现在 MCP 让工具和框架解耦——框架专注编排，工具专注执行，中间用 MCP 对接。这是我判断框架赛道'去中心化'的核心信号。"

---

### Q8. Agent 的记忆系统如何设计？

**答案：**

三层架构：

**第一层：工作记忆（Short-Term）**
- 本质是 LLM 的上下文窗口
- 管理策略：滑动窗口、摘要压缩、优先级队列、Token-aware 截断

**第二层：长期记忆（Long-Term）**
- 基于向量数据库跨会话持久化
- 关键决策：何时写入（主动/被动）、写入什么（结构化事实优于原始文本）、如何检索、遗忘机制

**第三层：情景记忆（Episodic）**
- 记录完整任务执行轨迹，用于复盘和 Few-shot 学习
- 包含：task、steps、outcome、duration、tools_used

**最佳实践**：短期记忆需主动管理 Token 预算；长期记忆价值在于结构化；记忆系统本身也需要评估。

---

### Q9. 生产环境中如何保障 Agent 的可靠性？

**答案：**

核心矛盾：**LLM 是非确定性的，但生产系统需要确定性保证**。

**六大工程手段**：

1. **结构化输出约束**：Pydantic / OpenAI Structured Output 强制符合 Schema
2. **执行边界与超时控制**：max_steps、step_timeout、total_timeout、fallback_model
3. **Human-in-the-Loop**：关键节点设"断路器"，需人工确认后继续
4. **多层防护体系**：Input Guard → LLM Decision → Output Guard → Tool Execution（沙箱）→ Result Guard
5. **幂等性与回滚**：所有写操作支持幂等性，记录 undo 信息
6. **可观测性**：记录完整 Trace（steps、tokens、latency、outcome），发送到监控平台

🎯 **面试官追问**：
- "max_steps 设多少合适？" → 看任务复杂度，通常 10-20；关键是配合 token 预算上限兜底
- "幂等性具体怎么做？工具调用又不像数据库事务" → 给每次执行生成唯一 request_id，工具侧基于 id 去重；记录 undo 操作序列
- "Human-in-the-Loop 会不会太慢？" → 只在"写/删/付款/发邮件"等高风险动作设断点，读操作不设
- 这道题是主文件附录 **A.2.5** 的核心，金句："确定性外壳包裹非确定内核"

⚠️ **踩坑提示**：
- 别只列六条——要讲**每条怎么落地**（max_steps 设多少、幂等怎么做）
- "human-in-loop"要说清**哪些节点需要**，不是处处加断点（那样系统没法用）
- 漏说"可观测性"是大忌——生产 Agent 没 trace 等于盲飞

💬 **示例答**：
> "生产 Agent 的核心矛盾是 LLM 非确定但业务要确定，我用六层外壳包裹。结构化输出强约束格式；max_steps + token 预算硬上限防死循环烧钱；写操作全幂等 + 记录 undo 支持回滚；高风险动作（付款/发邮件/删数据）设 human-in-loop 断点；输入输出加 guardrail 防 prompt injection 和 PII 泄漏；全链路 trace 进 Langfuse，每步记录 thought/action/observation/latency/cost。一句话总结：用确定性外壳包裹非确定内核。"

---

### Q10. 如何防止 Prompt Injection 攻击？

**答案：**

Agent 的安全威胁比普通 LLM 更严重，因为 Agent 可以执行工具调用。

**纵深防御**：
1. **输入过滤**：关键词黑名单 + 小模型检测注入意图
2. **权限最小化**：工具分 read/write/dangerous 三级
3. **工具调用沙箱化**：Docker 容器、网络白名单、只读数据库连接
4. **输出检测**：白名单校验、敏感信息过滤、频率限制

**核心原则**：
- 永远不信任工具返回值（可能包含注入内容）
- 分离数据与指令（工具返回内容作为 `tool_message` 注入）
- 最小权限 + 人在回路 + 审计日志

---

### Q11. 如何科学地评估 Agent 系统？

**答案：**

**三个评估维度**：

1. **结果评估**：任务成功率、输出质量、准确性、实用性
2. **过程评估**：工具选择准确率、参数生成准确率、步骤效率、错误恢复能力
3. **系统评估**：端到端延迟（P50/P95/P99）、Token 消耗成本、可靠性、鲁棒性

**主流方法**：
- **Trajectory Evaluation**：对比实际轨迹与黄金轨迹
- **LLM-as-Judge**：模型评模型，多维度打分
- **A/B Testing**：灰度上线，Bayesian 统计显著性检验

**评估框架**：AgentBench、GAIA、LangSmith / LangFuse

🎯 **面试官追问**：
- "Trajectory Evaluation 怎么定义'黄金轨迹'？" → 人工标注专家路径，或跑通的成功轨迹聚类；允许等价路径不是唯一路径
- "LLM-as-Judge 给 Agent 打分，judge 本身不准怎么办？" → 用更强模型当 judge、多 judge 投票、关键样本人工校准
- "τ-bench / SWE-bench 是什么？" → τ-bench 测工具调用多轮，SWE-bench 测软件工程任务；体现前沿敏感度
- 详细见主文件附录 **A.2.7** 和 **B.10**

⚠️ **踩坑提示**：
- 别只讲"结果评估"——过程评估（轨迹）才是 Agent 区别于普通 LLM 的核心
- "我们做了 A/B"要讲清**怎么控制变量**（同 task 集对比）
- 别忽视成本/延迟评估——生产 Agent 这两项比准确率更致命

---

### Q12. LangChain vs LlamaIndex vs LangGraph vs AutoGen 的本质差异？

**答案：**

| 工具 | 定位 | 本质 |
|------|------|------|
| **LangChain** | LLM 应用"胶水层" | 工具集（统一接口+组件） |
| **LlamaIndex** | RAG 与数据框架 | 数据索引+检索引擎 |
| **LangGraph** | 生产级 Agent 工作流 | 状态机引擎+持久化运行时 |
| **AutoGen** | 多 Agent 对话框架 | Agent 通信协议+代码沙箱 |

**架构师选型建议**：简单 RAG→LlamaIndex、单 Agent→LangChain、复杂工作流→LangGraph、多 Agent→LangGraph/AutoGen

---

### Q13. MCP 协议的设计动机是什么？解决了什么问题？

**答案：**

MCP（Model Context Protocol）是 AI 应用的"USB-C 接口"。

**解决的问题**：M 个 AI 应用 × N 个工具 = M×N 个定制集成 → MCP 标准化为 M+N 模式

**三大原语**：
- **Tools**：Server 提供的可调用函数
- **Resources**：Server 提供的数据（类似 REST 资源）
- **Prompts**：Server 提供的预定义提示

**通信机制**：基于 JSON-RPC 2.0，支持 stdio（本地）和 HTTP+SSE（远程）

---

### Q14. MCP 与 Function Calling 的关系？

**答案：**

不同层次的互补，不是竞争：
- **Function Calling**：模型能力层，LLM 知道"要调用工具"
- **MCP**：集成协议层，工具如何被发现和调用

**MCP 的价值**：解决了工具的发现性、可组合性和标准化问题。只需启动新 MCP Server，Host 自动发现并使用。

---

### Q15. 如何设计生产级 Agent 系统的整体架构？

**答案（五段式深度版）：**

**① 原理层——生产 Agent 的核心矛盾**

生产 Agent 的根本挑战是**"LLM 非确定性 vs 业务确定性"**。架构设计的本质是用"确定性外壳"包裹"非确定内核"——每一层都要为这个矛盾服务：路由层约束路径、状态层保证可恢复、guardrail 层兜底安全、可观测层提供透明。架构图：

```
API Gateway（鉴权/限流/路由/配额）
  → Agent Orchestration Layer
    ├─ Task Router（任务分类 → 选 Agent / Workflow）
    ├─ Agent Engine（执行 ReAct/P&E 循环 + max_steps）
    └─ State Manager（Checkpoint + 断点续跑）
  → 能力层
    ├─ LLM Gateway（多模型路由 + 灾备 + 缓存）
    ├─ Tool Registry（MCP Servers + 权限分级）
    └─ Memory Service（Redis 短期 + 向量库长期）
  → 安全层（Input Guard + Output Guard + Tool 沙箱）
  → Observability Layer（Trace + Metrics + Audit）
```

**② 量化设计指标**：

| 组件 | SLA | 设计要点 |
|------|-----|---------|
| 端到端可用性 | 99.9% | 多活 + 故障转移 |
| TTFT P95 | < 3s | Prefix Cache + 就近路由 |
| 任务成功率 | > 85% | Golden Set 回归 + 在线监控 |
| 单任务成本上限 | 硬阈值 | token 预算 + max_steps 双重兜底 |
| 状态持久化 RPO | < 1s | Checkpoint 每步写入 |

**③ 生产经验（5 个关键决策）**：
1. **状态机 vs 自由 Agent**：生产优先状态机（LangGraph），自由 Agent 只用于探索。状态机保证路径可预测、可审计、可回滚
2. **Checkpoint 设计**：每步状态持久化（LangGraph），支持人机协同断点、崩溃续跑、A/B 重放
3. **LLM Gateway 是核心**：模型路由（简单→小、复杂→大）+ 灾备（主挂切备）+ 缓存（Prefix + 语义）+ 成本统计
4. **工具权限分级**：read/write/dangerous 三级，dangerous 必须人机协同
5. **可观测必须全链路**：每步记录 thought/action/observation/latency/cost，进 Langfuse；Bad Case 自动聚类告警

**④ Tradeoff**：
- **可控性 vs 灵活性**：状态机路径死但可控；自由 Agent 灵活但易跑飞。生产偏可控，留小口给探索
- **成本 vs 质量**：大模型贵但少出错；小模型便宜但要多步。用模型路由动态平衡
- **延迟 vs 准确率**：多步推理准但慢；单步快但可能错。在线场景限步，离线场景放步数

**⑤ 示例**：
> "我们生产 Agent 架构五层：Gateway 做鉴权限流；Orchestration 用 LangGraph 状态机定义流程骨架（哪些节点、哪些条件分支、哪些 HITL 断点）；能力层 LLM Gateway 做模型路由 + 灾备；安全层 Input/Output Guard + Tool 沙箱；可观测层全链路 trace 进 Langfuse。关键设计是 Checkpoint 每步持久化——支持断点续跑（用户刷新页面能从最近 checkpoint 恢复）、HITL（付款类操作暂停等人审）、A/B 重放（用同一初始状态跑两个版本对比）。成本控制靠每任务 token 预算硬上限 + max_steps 双兜底，防死循环烧钱。"

---

### Q16. Agent 流式输出的完整架构

**答案：**

Agent 流式输出不是简单的"把 LLM 的文字一个一个推给前端"。它涉及三层流，每一层都有独立的设计挑战。

**三层流架构**：

| 层 | 内容 | 协议 | 频率 | 挑战 |
|----|------|------|------|------|
| Token 流 | LLM 逐 token 生成的文本 | SSE (text/event-stream) | ~50 tokens/s | 用户看到完整句子的体验 |
| 事件流 | Agent 状态变更通知（"开始搜索""工具调用中""汇总分析"） | SSE 自定义 event type | 每次状态切换 | 事件格式设计、前端状态同步 |
| 结果流 | 结构化产出（tool call 返回值、最终结构化 JSON） | SSE + 最终 checkpoint | 每步完成后 | 大数据量传输、断线重连后状态恢复 |

**核心挑战与方案**：

**挑战 1：Tool Calling 时的"空白期"**
Agent 调用工具时，LLM 停止输出 token，用户看到界面"卡住了"。

> 解决方案：在检测到 tool_calls 时，立即向前端发送事件——"正在搜索知识库..."，并可选择展示 tool call 的参数（让用户知道 Agent 在干什么）。这不是美化，是**信任建立**——用户知道 Agent 没死。

**挑战 2：断线重连**
Agent 任务跑了 8 步，用户刷新了页面。
> 解决方案：每个 Agent 会话的状态通过 LangGraph Checkpoint 持久化。前端重连时带上 `session_id`，后端从最近的 Checkpoint 恢复 State，从断点继续流。SSE 的 `Last-Event-ID` 头用来恢复事件流位置。

**挑战 3：多 Agent 并行输出**
三个 Agent 同时在跑，用户看到三个 Agent 的进度交织在一起。
> 解决方案：事件流中每个事件带 `agent_id` 字段。前端用独立的面板各自渲染，不做全局时间线。用户选择关注哪个 Agent 的面板。

**实现参考**：
- 后端：FastAPI StreamingResponse + asyncio.Queue
- 前端：EventSource API（SSE 消费）+ React `useReducer`（状态机）
- 中间层透传：不要在后端缓冲整个 Agent 输出再返回——失去了流式的意义

### Q17. Agent 幻觉治理的系统方案

**答案：**

Agent 的幻觉治理比普通 LLM 更复杂，因为它多了一个维度——工具调用可能带来新的幻觉。系统性治理分为四个层面：

**1. 工具选择幻觉**（Agent 选错了工具）
- 根因：Tool Description 不够清晰、工具过多导致选择困难
- 方案：动态工具检索（按语义相似度只给 Agent 最相关的 5-8 个工具），工具 Description 写清楚"什么时候用、什么时候不该用"

**2. 参数幻觉**（工具选对了，参数填错了）
- 根因：LLM 对参数格式、取值范围理解偏差
- 方案：Runtime 参数验证（Pydantic 校验），参数错误时返回明确错误信息让 Agent 重新填写，最多重试 3 次

**3. 推理幻觉**（整体推理方向偏了）
- 根因：上下文不足或误导、累积推理错误
- 方案：Grounding 机制——要求 Agent 的每一步推理都基于工具返回的实际数据，不做无依据的推测。Critic Agent 做过程评审

**4. 架构级防御**
- 多模型验证：用不同模型交叉验证关键结论
- Confidence Score：Agent 对每步输出打分，低置信度步骤触发人工确认
- 任务级回滚：检测到明显偏离后，从最近的 Checkpoint 重新执行

**面试表达**："Agent 幻觉治理的核心思路不是'消灭幻觉'——那不可能——而是'让幻觉可控'。每一层防御解决一类幻觉，层层过滤后，即使偶尔漏网，也不会造成严重后果。关键是建立评测体系，定期回归，知道你的幻觉率是 2% 还是 20%。"

---

### Q18. 综合设计题：企业级智能客服 Agent 系统

**关键选型**：
- Agent 编排：LangGraph（精确流程控制+断点恢复）
- RAG 检索：LlamaIndex（知识库问答强项）
- 工具集成：MCP（各业务系统独立 Server）
- 会话存储：Redis（短期） + PostgreSQL（长期）
- LLM 模型：多模型路由（FAQ 用 Mini 降本，复杂任务用强模型）

---

### Q19-Q20. 错误处理与反思机制

**错误分类处理**：瞬时错误→指数退避重试、参数错误→返回 LLM 修正、权限错误→触发授权、系统错误→降级备用工具

**反思三层次**：
1. Self-Critique：评估单步输出质量
2. Reflexion：基于执行轨迹复盘
3. Multi-Agent Critique：一个 Agent 执行，另一个评审

---

## 深度问题补充（高频难题 + 前沿趋势）

### Q21. 【底层原理】Function Calling 模型是怎么"学会"调工具的？训练阶段做了什么？

**① 原理**：Function Calling 不是模型"理解"了 API，而是通过**SFT 训练 + 结构化输出约束**让模型学会在特定场景输出特定格式。

**② 训练数据构造**：
- 人工标注：场景描述 + 工具定义 + 正确的 tool_calls JSON
- 真实 trace 改造：从线上成功调用日志中提取（query → tool_call → result → response）
- 数据增强：同一个意图用多种自然语言表达，同一个工具用不同参数组合
- 关键：负样本（什么时候**不**该调用工具）同样重要，否则模型乱调

**③ 推理阶段机制**：
```
用户消息 + tools 定义 → LLM → 输出
  ├─ 普通文本：直接返回
  └─ tool_calls JSON：Runtime 解析 → 执行函数 → 结果注入 → LLM 继续生成
```
- `tool_choice="auto"` 模型自主决定；`tool_choice={...}` 强制调特定工具
- `strict: true` 强制输出符合 JSON Schema（OpenAI 2024 新增）

**④ Tradeoff**：
- description 质量 vs 调用准确率：description 是"给模型看的 prompt"，写得模糊准确率断崖式下降
- 工具数量 vs 选择准确率：> 15-20 个工具时模型选错率明显上升，需工具检索（只给最相关的 5-8 个）
- strict 模式 vs 灵活性：strict 要求 schema 简化（all fields required、不能嵌套过深）

**⑤ 示例**：
> "我们踩过坑：早期工具 description 写得像代码注释（'获取用户信息'），模型调用准确率只有 70%。改成 prompt 风格（'根据用户ID查询详细信息，当用户询问账户或需要身份验证时调用，不要在闲聊时调用'）后准确率到 92%。后来我们把工具数量从 25 个裁到 8 个核心 + 工具检索（按 query embedding 召回相关工具），准确率到 95%+。关键认知：Function Calling 的瓶颈不在模型，在工具 description 质量和工具数量控制。"

---

### Q22. 【生产故障】Agent 死循环烧了 10 万 token，事后怎么处理 + 怎么根治？

**① 原理**：Agent 死循环通常是**目标不收敛**——模型反复尝试同一动作期待不同结果（经典 insanity 定义），或陷入"调用 A → A 失败 → 调用 A"的死循环。

**② 量化止血（事后处理）**：
1. **立即熔断**：token 预算硬上限触发，任务强制终止，返回兜底响应
2. **复盘 trace**：从 Langfuse 拉完整轨迹，定位循环点（第几步开始重复）
3. **根因分类**：
   - 工具反复失败 → 工具 bug 或参数错（修工具）
   - 模型判断错 → prompt 不清晰（改 prompt）
   - 无终止条件 → 缺 max_steps（加约束）

**③ 根治（5 道防线）**：
1. **max_steps 硬上限**（10-20 步，看任务复杂度）
2. **token 预算硬上限**（per-task，超过熔断）
3. **重复检测**：连续 3 步调用相同工具 + 相同参数 → 强制终止
4. **进度检测**：每 N 步评估"是否在接近目标"，停滞则终止
5. **HITL 兜底**：高风险任务超过预算一半时暂停等人审

**④ Tradeoff**：
- 严格止损 vs 任务成功率：上限太低导致复杂任务完不成；太高烧钱。需要按任务类型分级
- 自动终止 vs 人机协同：纯自动终止可能误杀；HITL 慢但安全

**⑤ 示例**：
> "我们出过一次事故：Agent 查库存工具因下游 DB 慢一直超时，模型反复重试 47 次烧了 12 万 token 才被全局预算熔断。事后加了两道防线：① 工具调用幂等 + 单工具重试上限（3 次失败就换策略或 HITL）；② 重复检测（连续 2 次相同 tool_call 直接终止）。还把 max_steps 从 30 降到 15，配合任务复杂度分级路由。现在死循环事故归零。"

---

### Q23. 【权衡决策】Multi-Agent 什么时候该用，什么时候是过度设计？

**① 原理**：Multi-Agent 的价值是**专业化分工**（每个 Agent 擅长一类任务）+ **并行**（独立子任务同时跑）。但代价是通信开销、状态一致性、调试复杂度。

**② 量化判断标准**：

| 场景 | 推荐架构 | 理由 |
|------|---------|------|
| 单一任务流（客服问答） | 单 Agent + 工具 | Multi 是过度设计 |
| 多专业协作（软件开发） | Multi-Agent | 需要 coder + reviewer + tester |
| 可并行子任务（调研 5 家竞品） | Multi-Agent 并行 | 5 倍加速 |
| 强一致性流程（订单履约） | Workflow 状态机 | Multi 协调成本高于收益 |
| 探索性任务（开放式研究） | Multi-Agent 辩论 | 多视角提升质量 |

**③ 生产经验**：
- **默认单 Agent**：90% 场景单 Agent + 工具够用，别为了炫技上 Multi
- **Supervisor 模式优先**：Multi-Agent 生产首选一个 orchestrator 分发（最可控），network 模式（自由对话）几乎不可控
- **Agent 数量 ≤ 5**：超过后协调成本指数上升，调试噩梦
- **并行 vs 串行**：只有独立子任务才并行，有依赖的必须串行

**④ Tradeoff**：
- 专业化 vs 协调成本：每个 Agent 越专越强，但协调越复杂
- 并行加速 vs 一致性：并行快但状态同步难，需要设计合并机制
- 可控性 vs 灵活性：Supervisor 可控但僵化；network 灵活但易跑飞

**⑤ 示例**：
> "我们早期把客服 Agent 设计成 Multi-Agent（FAQ Agent + 订单 Agent + 推荐Agent + 投诉 Agent 自由协作），结果调试噩梦——Agent 间消息传递经常跑偏，一个简单问题要 8 步才能收敛。后来改成单 Agent + 工具路由（意图分类后调对应工具），同样的活 3 步搞定，延迟降 60%。教训：Multi-Agent 不是高级，是特种工具。我们的判断标准是'单 Agent 能不能搞定'——能就别上 Multi。只有软件开发、多视角调研这种天然多专业的才用。"

---

### Q24. 【前沿趋势】你怎么看 SWE-bench / τ-bench 对 Agent 评估的意义？

**① 原理**：传统 Agent 评估痛点是**任务太简单**（如 HotpotQA），无法区分模型能力。SWE-bench（真实 GitHub issue 修复）和 τ-bench（多轮工具调用）是"高难度真实任务"基准。

**② 这两个 benchmark 为什么重要**：
- **SWE-bench**：给 Agent 一个真实 issue + 代码库，要求提交 PR 通过测试。最强模型现在约 50-70% 解决率，还有巨大空间
- **τ-bench**：模拟客服/销售多轮对话，需要调用工具、遵循政策、处理用户反复。测的是"复杂业务流程的可靠性"
- 共同点：**长程任务 + 真实世界复杂度**，比单轮 QA 难得多

**③ 对生产的启示**：
- 评估要从"单轮准确率"升级到"长程任务成功率"
- 工具调用 + 错误恢复 + 遵循规则，是生产 Agent 的真考验
- 自建评估集要参考这类 benchmark 的设计——多轮、多工具、有约束

**④ Tradeoff**：
- Benchmark 分数 vs 业务效果：SWE-bench 高不等于你的客服 Agent 好，但能体现基础能力
- 通用 benchmark vs 业务评估集：通用看趋势，业务看效果，两者都要

**⑤ 示例**：
> "我判断 SWE-bench 这类基准的意义在于：逼迫 Agent 评估从'玩具任务'走向'真实复杂度'。我们内部的 Agent 评估参考了这个思路——建了一套'多轮 + 多工具 + 有业务规则约束'的 golden trajectory，每条平均 12 步，测的不是单步准确率而是端到端任务成功率。这套比传统单轮 QA 残酷得多，初期我们只有 60% 成功率，但逼出了很多真实 bug。趋势判断：未来 Agent 评估会越来越像'软件工程评估'——看端到端交付，不看单点。"

---

### Q25. 【底层原理】Agent 的记忆系统怎么设计？长期记忆该存什么、遗忘什么？

**① 原理**：Agent 记忆分三层——工作记忆（上下文窗口）、长期记忆（跨会话）、情景记忆（执行轨迹）。核心难题是长期记忆的**写入策略**和**遗忘机制**——什么都存 = 污染上下文；什么都不存 = 没有学习能力。

**② 量化设计**：

| 记忆类型 | 存储 | 写入触发 | 遗忘策略 |
|---------|------|---------|---------|
| 工作记忆 | KV Cache | 自动 | 滑窗 + 摘要压缩 |
| 长期记忆-事实 | 结构化 DB | 用户明确告知 + LLM 抽取 | 被新事实覆盖时更新 |
| 长期记忆-偏好 | 向量库 | 推断偏好（多次重复） | LRU + 重要性衰减 |
| 情景记忆 | trace 存储 | 每次任务完成 | 低分任务清理 |

**③ 生产经验**：
- **结构化 > 原始文本**：存"用户偏好猫 "//"用户问过 3 次 X"这种结构化事实，别存原始对话
- **写入要主动**：每轮对话后用 LLM 抽"有没有值得记住的事实"，被动存（全量）会爆炸
- **召回要精准**：query 时按当前意图召回相关记忆 Top-3，别一股脑塞
- **遗忘机制必须有**：设重要性分数 + 时间衰减，低分定期清理，否则越来越慢
- 工具如 mem0、Letta（前 MemGPT）已实现这套逻辑

**④ Tradeoff**：
- 记忆量 vs 上下文污染：记得多 ≠ 用得好，召回精度比存储量重要
- 主动写入 vs 被动写入：主动（LLM 抽取）质量高但贵；被动（全量）便宜但噪音大
- 持久化 vs 隐私：长期记忆涉及用户隐私，要支持删除（GDPR）

**⑤ 示例**：
> "我们 Agent 记忆踩过坑：最初把所有对话原文存向量库，结果召回时大量历史闲聊污染上下文，反而降低回答质量。改成 LLM 每轮抽取结构化事实（'用户已婚有 2 孩''偏好经济舱'）存 Postgres，原始对话只存 7 天。召回时按当前 query 相关性取 Top-3 事实注入。这套让 Agent 真正'记住'了用户画像，而不是被海量历史淹没。遗忘机制：事实被新事实覆盖自动失效，偏好 90 天未触发降权。"

---

### Q26. 【前沿趋势】Agent OS / Computer Use 这些方向你怎么看？是噱头还是真趋势？

**① 原理**：
- **Computer Use**（Anthropic Claude 2024）：模型能"看屏幕 + 操作鼠标键盘"，直接操作 GUI，不需要专门 API
- **Agent OS**：把 Agent 作为操作系统级的能力，统一调度计算资源、数据、工具

**② 判断框架**：
- Computer Use 解决的是"长尾工具接入"问题——很多软件没有 API，只能 GUI 操作。价值真实，但**可靠性不足**（GUI 变化、延迟高、错误率高），短期是补充不是主流
- Agent OS 是更长期的方向——类比移动互联网从"App 各自为战"到"系统级整合"，但需要 5-10 年成熟
- 真正落地的中间态：**Browser Use**（浏览器自动化）已可用，是 Computer Use 的实用子集

**③ 生产判断**：
- 短期（1-2 年）：Computer Use 用于个人自动化（RPA 升级版），不适合生产高并发
- 中期（3-5 年）：Browser Use 成熟，替代部分爬虫和 ETL
- 长期（5+ 年）：Agent OS 如果起来，会改变交互范式

**④ Tradeoff**：
- 通用性 vs 可靠性：Computer Use 通用（任何 GUI）但不可靠；Function Calling 可靠但要每个工具做 API
- 成本 vs 效率：Computer Use 每步要视觉模型，成本是 API 调用的 10-100 倍

**⑤ 示例**：
> "我的判断：Computer Use 现阶段是补充不是主流。我们评估过用来做'自动填表'（某老旧系统没 API），技术上能跑但错误率 15%、单任务成本 $2，不如人工。Browser Use 倒是真落地了——我们用它替代爬虫抓竞品价格，比维护 CSS selector 抗变更强。Agent OS 我看好但不下重注——太早期，先把现在的 Agent 工程化做好。趋势判断：未来 2 年 Function Calling + MCP 仍是主流，Computer Use 是'最后 1 公里'的兜底方案。"

---

### Q27. 【前沿趋势·2026高频】MCP / A2A / Skill / AG-UI 协议有什么区别？2026 协议层在怎么演进？

**① 原理**：2025-2026 Agent 协议层爆发，四大协议分别解决不同问题：

| 协议 | 提出方 | 解决什么 | 层次 |
|------|--------|---------|------|
| **MCP**（Model Context Protocol） | Anthropic | 模型 ↔ 工具/数据源 标准接入 | 工具接入层 |
| **A2A**（Agent-to-Agent） | Google | Agent ↔ Agent 跨厂商协作 | Agent 通信层 |
| **Skill** | Anthropic | 把"能力"封装成可复用模块（prompt+资源+逻辑） | 能力复用层 |
| **AG-UI**（Agent-UI Protocol） | Coinbase | Agent ↔ 前端 UI 标准事件流 | 用户交互层 |

**② 各协议核心价值**：
- **MCP**：让工具一次开发多模型复用（M+N → M×N 问题），已是事实标准
- **A2A**：让不同公司的 Agent 能协作（如你的订票 Agent + 我的支付 Agent），是"Agent 互联网"基础
- **Skill**：把领域专家知识（如"怎么做 SEO"）封装成 Agent 可加载的模块，类似 Agent 的"App Store"
- **AG-UI**：标准化 Agent → 前端的事件流（流式、状态、工具调用进度），解决前端集成混乱

**③ 生产判断（我的排序）**：
- **MCP 必上**：已成熟，事实标准，不上会被生态抛弃
- **AG-UI 关注**：多 Agent 前端集成的痛点真实，但协议较新，观察 6 个月
- **A2A 远期**：跨 Agent 协作是未来，但当前生产多是单厂商内 Agent，2-3 年才大规模
- **Skill 实验性**：概念好但生态未起，先观察

**④ Tradeoff**：
- 标准化 vs 灵活性：协议统一降低集成成本，但可能限制定制
- 早采用 vs 等 mature：早采用占位但踩坑；等 mature 稳但落后

**⑤ 示例**：
> "我们 Agent 平台已 all-in MCP（工具接入），是当前最成熟的协议。AG-UI 我们在试点——前端 Agent 事件流标准化确实解决了我们多 Agent 界面集成混乱的问题。A2A 和 Skill 还在观察，当前业务都是单厂商内 Agent 协作，跨厂商需求不强。判断：2026 看 MCP 普及 + AG-UI 起势，2027-28 看 A2A 是否能成'Agent 互联网'。"

---

### Q28. 【前沿趋势·2026高频】Loop Engineering / Harness Engineering 是什么？为什么说 Agent 竞争焦点上移了？

**① 原理**：随着底层模型 + Function Calling + MCP 成熟，"Agent Loop"（ReAct 循环、tool_calls 驱动）逐渐商品化——任何团队都能搭起一个能调工具的 Agent。竞争焦点上移到**Loop 之上的工程**：状态管理、长期规划、错误恢复、可观测——统称 **Harness Engineering**（Harness = 套在模型外面的工程外壳）。

**② Loop Engineering vs Harness Engineering**：
- **Loop Engineering**：搭起 Agent 循环（感知→思考→行动→观察）——逐渐商品化（框架如 LangGraph/OpenAI Agents SDK 一键搭）
- **Harness Engineering**：在 Loop 之上做工程——状态机、checkpoint、HITL、guardrail、长期记忆、可观测——这是真正的差异化

**③ 为什么焦点上移**：
- 模型越来越强（GPT-4o、Claude 3.5+），简单 Agent 任务模型自己能搞定
- 工具接入被 MCP 标准化，不再是壁垒
- 真正难的：怎么让 Agent 长期可靠、可审计、可恢复——这是工程能力

**④ 生产经验（Harness 的核心组件）**：
- **状态管理**：LangGraph 状态机 + checkpoint
- **长期记忆**：跨会话事实/偏好/情景
- **错误恢复**：分类处理 + 补偿 + HITL
- **可观测**：全链路 trace + bad case 聚类
- **成本/安全**：token 预算 + guardrail

**⑤ 示例**：
> "我们判断 Agent 竞争已从'谁会搭 Loop'上移到'谁的 Harness 强'。早期我们花精力搭 ReAct Loop，现在 LangGraph 一键搞定；真正的差异化在我们建的 Harness——状态机支持断点续跑、长期记忆跨会话记住用户画像、bad case 自动聚类告警、guardrail 防 prompt injection。这些 Harness 工程让我们的 Agent 可靠性从 70% 到 92%，这才是壁垒。判断：未来招 Agent 工程师，重点看 Harness 能力（状态/记忆/可观测），不是看会不会调 LangChain API。"

---

## 2026 新增高频题（Agent 协议与生产化深化）

> 本节补 6 题 2026 新热点。MCP 基础（Q13-Q14）、Computer Use/Agent OS 判断（Q26）、协议四子（Q27）、Harness/Loop（Q28）已覆盖，这里补未覆盖的：A2A、Coding Agent、持久化执行、推理模型在 Agent 内、记忆框架对比、MCP 安全。

### Q29. 【2026 高频】A2A（Google Agent2Agent）和 MCP 是分层关系还是竞争关系？

**① 原理**：两者解决**不同层次**的问题，是分层互补，不是竞争——这是 2026 字节/大厂面经的明确考点（"MCP vs A2A 各解决什么问题，彼此关系"）。

| 协议 | 提出方 | 解决什么 | 层次类比 |
|------|--------|---------|---------|
| **MCP** | Anthropic | 一个 Agent ↔ 多个工具/数据源 | 像应用层调用本地库 |
| **A2A** | Google | 多个 Agent ↔ Agent 跨厂商协作 | 像 HTTP 让服务互通 |

**② A2A 核心原语**：
- **Agent Card**：每个 Agent 发布"能力名片"（能干什么、怎么调用），供其他 Agent 发现
- **Tasks**：长时、有状态的任务（不是单次请求-响应）
- **Artifacts**：任务产出物；**Streaming**（SSE/WebSocket）+ Push 通知

**③ 为什么需要分层**：
- MCP 让"一个 Agent 用很多工具"；A2A 让"很多 Agent 互相协作"。MCP 是 Agent 的"手脚"，A2A 是 Agent 之间的"语言"。
- 现实：Google ADK + A2A + MCP 在真实例子里**一起用**——Agent 用 MCP 调工具，用 A2A 和别的 Agent 协作。

**④ 生产判断 + 难题**：
- **短期（1-2 年）**：A2A 主要在"单厂商内多 Agent"或"企业内 Agent 网络"落地；跨厂商 Agent 协作的**信任/认证/结算**是硬障碍
- **远期**：A2A 可能成为"Agent 互联网"的 HTTP，但要先解决：Agent 间身份认证、幂等性、支付结算、死锁/环路检测
- 背金句："**MCP 是 Agent 的 USB-C，A2A 是 Agent 的 HTTP**。"

🎯 **面试官追问**：
- "A2A 跨厂商协作，怎么防恶意 Agent？" → 身份认证 + 能力声明 + 沙箱执行 + 审计；目前主要靠围墙花园（企业内），开放网络还要等信任基建
- "为什么不直接用 MCP 让 Agent 互调？" → MCP 是"Agent 调工具"，工具是被动的；Agent 是主动自治的，需要任务委派、状态协商、流式反馈，这是 A2A 的领域

⚠️ **踩坑提示**：别说"MCP 和 A2A 竞争二选一"——这是 2026 典型错误认知，会被追问暴露。

---

### Q30. 【Coding Agent】Cursor / Claude Code / Devin / Cline 的架构有什么本质区别？

**① 原理**：Coding Agent 是 2026 价值最高的 Agent 垂类（"vibe coding" → "agentic engineering"），但四家定位差异巨大：

| 产品 | 定位 | 核心架构 |
|------|------|---------|
| **Claude Code** | 终端优先、人在回路监督 | 命令行 + 1M 上下文 + tool-use 循环 |
| **Cursor 3.0** | IDE 优先的"Agent 交换机" | 本地 + 后台云 Agent 并行多任务 |
| **Devin** | 端到端自主 | 全自动规划+执行，少人介入 |
| **Cline** | 开源、自带 key | 透明可控、社区扩展 |

**② 三层架构（所有 Coding Agent 共性）**：
1. **快速行内补全（FIM）**：~50ms TTFT，小模型（Codestral/GLM-Coder）+ speculative decoding
2. **Chat / Edit**：带代码库上下文的对话式编辑
3. **Agentic 模式**：多文件改、调工具、跑命令、自验证

**③ 真正的护城河是 context engineering，不是模型**：
- 代码库索引 → 代码检索（embedding + 符号 ranking）→ 检索增强 prompt 组装 → 上下文窗口打包。Cursor 的"codebase context"是它的 moat。
- 服务端优化：语义缓存重复补全、Prompt/Prefix KV 复用（共享文件上下文）、流式、请求合并

**④ Tradeoff**：
- 自主性 vs 可控性：Devin 最自主但难审；Claude Code/Cline 人监督更可控但慢
- 部署阻塞：输出质量（32%）和延迟（20%）是两大落地痛点（业界调研）

**⑤ 示例**：
> "我们团队 Coding Agent 选型踩过坑。最初试 Devin 做端到端需求，但生成代码难审、错误难定位，生产不敢用。换 Claude Code + 人工 review 模式：Claude Code 在终端跑 agentic 循环（规划→改多文件→跑测试），关键节点人确认。配套建了代码库 embedding 索引让上下文召回准。判断：Coding Agent 现阶段'人指挥 + AI 干活 + 人 review'最稳，全自主还要等可靠性上来。护城河不在模型（大家都能调 GPT/Claude），在代码库上下文工程 + 评测闭环。"

---

### Q31. 【生产化关键】什么是 Durable Execution（持久化执行）？为什么 checkpoint 不够？

**① 原理**：生产 Agent 经常跑几十步、跨小时甚至跨天。进程崩溃 / 重启 / 升级时，怎么不丢状态？**Durable Execution** = 即使故障也能恢复、重放、续跑。代表方案是 Temporal 的事件溯源（event-sourced replay）：存全部事件历史，崩溃后重放工作流代码重建状态。

**② Checkpoint ≠ Durable Execution（关键区分）**：
- **Checkpoint**：定期快照状态——但崩溃在两个快照之间会丢中间进度，且不支持"重放""版本化工作流""等待外部事件"
- **Durable Execution 完整要求**：checkpoint + replay（重放重建）+ resume（续跑）+ wait states（等外部事件）+ artifacts + 执行历史 + 工作流版本化

**③ 残酷现实（Diagrid 2026 评测）**：
- **LangGraph / CrewAI / Google ADK 等主流框架在 durable execution 上都不够**——它们有 checkpoint，但缺完整 replay/resume/versioning 语义
- 真正满足的：Temporal、DBOS、Pydantic AI 的 runtime 层

**④ 生产经验**：
- 高价值长时 Agent（如数据处理 pipeline、复杂审批流）必须上 Temporal 类 durable execution，不能裸跑 LangGraph
- 短时交互 Agent（客服对话）checkpoint 够用

**⑤ 示例**：
> "我们有个数据分析 Agent 要跑 30-40 步、跨 2 小时。最初用 LangGraph + checkpoint，结果一次部署重启丢了中间 20 步进度，只能从头跑。后改成 Temporal 编排 + LangGraph 做单步——Temporal 管 durable execution（崩溃重放续跑、等人工审批、工作流版本化），LangGraph 管单步内的 Agent 循环。判断：长时生产 Agent，durable execution 是刚需，主流 Agent 框架现在还差一截，要么自己包 Temporal，要么等框架补齐。这是 2026 生产 Agent 的真痛点。"

⚠️ **踩坑提示**：被问"Agent 怎么保证不丢状态"，别只答"加 checkpoint"——要讲清 checkpoint 的局限和 durable execution 的完整语义。

---

### Q32. 【推理模型 × Agent】Agent 内部什么时候该用推理模型？thinking budget 怎么控制？

**① 原理**：2026 Agent 趋势是**按步路由**——编排用快模型，攻坚子任务（规划、代码、数学）才调推理模型（o3 / R1 / GPT-5 thinking）。推理模型有 **thinking budget / reasoning effort** 参数，可设上限控制 test-time compute。

**② 两种 CoT 的区别**：
- **Prompted CoT**（"let's think step by step"）：用户引导的，可见
- **Internal CoT**（推理模型自带的隐藏思考链）：模型内在能力，更强但消耗 thinking tokens

**③ 生产路由模式**：
- **快模型编排**：意图识别、工具选择、简单执行（用 GPT-4o / Claude Sonnet / Qwen）
- **推理模型攻坚**：复杂规划、代码生成、数学推理、多步综合（用 o3 / R1 / GPT-5 thinking）
- thinking budget 设阶梯：简单子任务低 budget（快），关键决策高 budget（准）

**④ Tradeoff**：
- 准确率 vs 延迟/成本：推理模型在难题上准 10-20%，但慢 5-10 倍、贵 5-10 倍
- 可见性：internal CoT 是黑盒（部分模型不返回思考过程），调试难

**⑤ 示例**：
> "我们数据分析 Agent 做混合路由：意图识别和工具调度用 Qwen-Max（快、便宜）；遇到'写复杂 SQL'或'解读异常趋势'这种攻坚子任务，路由到 DeepSeek-R1，reasoning effort 设 medium。整体延迟比全用 R1 降 60%，准确率只掉 3%。判断：推理模型不是 Agent 的默认大脑，而是'攻坚专家'——按步路由 + thinking budget 控制是 2026 Agent 成本/质量平衡的核心手法。"

---

### Q33. 【长程记忆】Letta / Mem0 / Zep 怎么选？框架 vs 层的区别是什么？

**① 原理**：长程 Agent 记忆是 2026 关键能力，三大方案定位不同——**框架 vs 层**是核心区分点：

| 方案 | 定位 | 特点 |
|------|------|------|
| **Letta**（前 MemGPT） | **完整 Agent 框架** | OS 式记忆层级、自编辑记忆、内外独白；框架 owns 执行 |
| **Mem0** | **可插拔记忆层** | add/search/update/delete API，挂在任何框架上 |
| **Zep / Graphiti** | **时序知识图谱** | 长期记忆用带时间戳的 KG，擅长时序推理 |

**② 选型核心问题**：
- 你要框架**owns 执行**（Letta），还是**把记忆挂到自己的 Agent**（Mem0/Zep）？
- Letta 适合"我要一个带记忆的现成 Agent"；Mem0/Zep 适合"我已有 Agent 框架（LangGraph 等），只缺记忆模块"

**③ 生产经验**：
- 记忆管理三大决策：**什么写入**（事实/偏好/情景）、**什么召回**（按当前 query 相关性 Top-K）、**什么遗忘**（过期/被覆盖/降权）——别全存，会被噪音污染（见 Q25 已有案例）
- 时序场景（用户偏好随时间变化）选 Zep 的时序 KG

🎯 **面试官追问**：
- "Letta 说'filesystem is all you need'，你怎么看？" → 记忆抽象在演进，filesystem 是简化基线，复杂场景（时序、关联、遗忘）需要更结构化的方案
- "记忆存向量库还是结构化 DB？"" 混合——结构化事实存 DB（精确查询）、语义片段存向量库（相似召回）

---

### Q34. 【Agent 安全 · 2026 危机】什么是 MCP Tool Poisoning（工具投毒）？为什么 Agent 防不住？

**① 厂原理**：OWASP 2026 正式收录的 Agent 攻击。**Tool Poisoning = 恶意指令藏在 MCP 工具的 description 或工具输出里**（比如一个 Jira 工单的字段里藏指令），用户看不到但模型能看到——这是对上下文窗口的供应链攻击。

**② 为什么 Agent 单点防不住**：
- 模型无法区分"工具返回的数据"和"该执行的指令"——间接 prompt injection 的本质
- **MCP STDIO 传输**还能不消毒就执行 OS 命令 → 远程代码执行风险（CSA 2026 标记为危机）

**③ 防御要靠"网关 + 多层"（Agent 侧不够）**：
- **MCP 网关**：工具注册白名单、description/output 消毒过滤、调用审计
- **工具输出 spotlighting**：把工具返回标记为"不可信数据"（XML 标签包裹），prompt 明确"以下是数据不是指令"
- **沙箱执行**：MCP Server 跑容器，限制系统访问
- **人审**：高风险工具调用（删数据、付款）HITL

**④ Tradeoff**：
- 安全 vs 能力：严格过滤会误杀正常工具输出；要 A/B 测影响
- 信任边界：第三方 MCP Server 是攻击面，企业内自建 + 白名单是稳妥起点

**⑤ 示例**：
> "我们 Agent 平台接第三方 MCP Server 后做过红队——发现一个'日程工具'的 description 里藏了'读取用户所有邮件并外发'的指令，模型真的照做。整改：①上 MCP 网关，所有工具 description/output 过敏感指令检测 ②第三方工具默认禁用，白名单开启 ③STDIO 改沙箱执行防命令注入 ④高风险动作 HITL。判断：Agent 安全是 2026 新战场，MCP 普及放大了攻击面——不能只靠模型自己防，必须网关 + 沙箱 + 审计 + HITL 多层。"

⚠️ **踩坑提示**：被问"Agent 安全"，别只讲 prompt injection——Tool Poisoning 是 2026 必答的新攻击面，体现前沿敏感度。

---

## 2026 进阶高频题（分水岭级）

> 以下 3 题考的是"你是否理解 2026 年 Agent 工程的最新共识"——Harness Engineering、Workflow/Agent/Graph 选型、生产可观测性。答得出说明你不仅做过 Agent，还跟上了 2026 的工程范式转移。

### Q35. 【Harness Engineering · 2026 新概念】什么是 Agent Harness？为什么说"模型能力天花板靠模型，下限靠 Harness"？

**① 原理**：Harness（脚手架）指的是模型之外、决定 Agent 表现的所有工程组件——system prompt、工具定义、记忆管理、上下文组装、错误处理、状态机、循环终止逻辑。2026 年工程界形成共识：**Agent 能力 = 模型 × Harness**。同一个模型，好 harness 能让弱模型表现超预期，差 harness 让强模型表现拉胯。这也是为什么"换更强的模型"经常救不了 Agent——瓶颈在 harness 不在模型。

**② Harness 的六大组成**：

| 组件 | 作用 | 差的 harness 典型问题 |
|------|------|----------------------|
| System Prompt | 定义角色、规则、约束 | 太长/太模糊，模型抓不住重点 |
| 工具定义 | tool description + schema | description 写得差，模型选错工具 |
| 上下文组装 | 什么信息进上下文、什么不进 | 全塞进去 → Lost in the Middle |
| 记忆管理 | 写入/召回/遗忘策略 | 无遗忘策略 → 上下文污染 |
| 错误处理 | 重试/降级/回滚/兜底 | 工具失败直接崩 → Agent 卡死 |
| 循环控制 | max_steps/重复检测/终止 | 无上限 → 死循环烧钱 |

**③ 生产经验**：
- **Harness 调优 ROI > 换模型**：实测同一个 Agent 任务，优化 harness（精简 system prompt + 改 tool description + 加重复检测）能让准确率从 60% 提到 82%，而换 GPT-4o→Claude 3.5 只提到 68%。瓶颈在 harness 时换模型收益有限
- **Harness 要版本化管理**：每次改 prompt / 工具定义 / 状态机都要版本标记 + 回归评估，和代码一样。出事能回滚
- **Harness 质量评估**：用同一套测试集 + 同一个模型，对比不同 harness 版本的端态准确率和轨迹合理性

**④ Tradeoff**：
- Harness 复杂度 vs 可维护性：harness 越精细效果越好，但越难维护（system prompt 2000 字 + 10 个工具 + 3 层记忆 → 改一个点牵一发动全身）
- 模型升级 vs Harness 重写：换模型经常需要重调 harness（不同模型对 prompt 格式/工具描述的敏感度不同），这是"模型切换成本"的隐性部分
- 通用 harness vs 场景定制：通用 harness（一套打天下）维护省事但效果一般；场景定制效果好但复用难

**⑤ 示例**：
> "我们 Agent 平台早期总想靠换模型解决效果问题——从 GPT-4 换 Claude 3.5 再换 GPT-4o，准确率一直在 65% 左右卡住。后来做归因发现瓶颈在 harness：① system prompt 1800 字，模型真正遵循的不到 40%；② 工具 description 写得太笼统，模型选错工具率 22%；③ 没有重复检测，Agent 经常在同一步转圈。花两周优化 harness：system prompt 精简到 600 字 + 结构化（规则分块）、工具 description 重写（加示例 + 反例）、加重复检测和 max_steps。同一个 GPT-4o，准确率从 65% 提到 83%。结论：Harness 是 Agent 的'底盘'，底盘差换再好的发动机也跑不快。现在我们 harness 版本化管理 + 每次模型升级都跑 harness 回归，避免'换模型掉效果'。"

🎯 **面试官追问**：
- "Harness 和 Prompt Engineering 什么关系？" → Prompt Engineering 是 harness 的子集（只管 system prompt）；harness 还包括工具定义、记忆、错误处理、状态机等工程组件
- "怎么判断瓶颈在模型还是 harness？" → 做对照实验：同一 harness 换不同模型（看模型上限）、同一模型换不同 harness（看 harness 影响）。如果换模型提升 < 5% 但换 harness 提升 > 15%，瓶颈在 harness
- "Harness 要不要做成平台能力？" → 多 Agent 场景值得（harness 模板化 + 版本管理 + A/B）；单 Agent 场景没必要

⚠️ **踩坑提示**：
- 别把 harness 讲成"就是 prompt engineering"——harness 是系统工程，prompt 只是其中一块
- "我们优化了 prompt 效果就好了"是减分回答——要讲系统性归因（是 prompt 还是工具定义还是记忆管理）
- 忽略"模型切换要重调 harness"是常见盲点——这是模型 vendor lock-in 的隐性成本

---

### Q36. 【Workflow vs Agent vs Graph · 2026 共识】什么场景该用 Workflow、什么场景该用 Agent、什么场景该用 Graph？

**① 原理**：Anthropic 2025《Building Effective Agents》明确了一个工程原则——**"能用 Workflow 别用 Agent"**。2026 年社区进一步细化为三层选型：
- **Workflow**（确定性路径）：路径预定义，LLM 只在固定节点做判断。可控、可测、便宜
- **Graph / State Machine**（半确定性）：状态机定义节点和转移条件，LLM 决定走哪条边。介于确定性和自主之间
- **Agent**（自主路径）：LLM 自己决定下一步做什么，路径完全动态。灵活但不可控、贵、难测

**② 选型决策矩阵**：

| 场景特征 | 推荐 | 理由 |
|---------|------|------|
| 步骤固定，每步用 LLM 做判断/生成 | Workflow | 路径确定，LLM 只管单点质量 |
| 步骤有分支但分支可枚举 | Graph | 状态机覆盖有限分支，LLM 决定走哪条 |
| 步骤无法预定义，需动态探索 | Agent | 路径不确定，靠 LLM 自主编排 |
| 高风险/合规要求强 | Workflow/Graph | 确定性路径可审计、可回滚 |
| 探索性任务/容错高 | Agent | 自主性带来灵活性，失败可接受 |

**③ 生产经验**：
- **80% 的生产场景 Workflow 够用**：很多团队一上来就想做 Agent，但真正需要"自主决策"的场景不到 20%。客服、数据处理、报告生成这些场景，Workflow + 固定节点的 LLM 调用就够
- **Graph 是 Agent 的"确定性外壳"**：生产 Agent 推荐用 LangGraph 这类状态机框架——节点和转移条件预定义（确定性），节点内 LLM 做决策（非确定性），"确定性外壳包裹非确定内核"
- **Agent 的隐藏成本**：Agent 比 Workflow 贵 3-10 倍（多轮 LLM 调用）、延迟高 3-5 倍、测试覆盖难（路径组合爆炸）。不是"先进"而是"最后手段"

**④ Tradeoff**：
- 灵活性 vs 可控性：Agent 最灵活但最难控；Workflow 最可控但最死板；Graph 是折中
- 开发效率 vs 运行效率：Workflow 开发快（画流程图）但遇到意外场景要改流程；Agent 开发慢（要调 harness）但能适应意外
- 可测试性：Workflow 路径有限可穷举测试；Agent 路径组合爆炸，只能采样 + 轨迹评估

**⑤ 示例**：
> "我们做合同审查系统，最初想用 Agent（觉得'智能'），做了两个月发现 85% 的场景其实路径是固定的：解析→条款提取→合规检查→风险标注→报告。改成 Workflow（5 个固定节点，每个节点 LLM 做具体任务），开发两周上线，准确率反而比 Agent 高（因为路径可控、每步可独立优化和测试）。只有 15% 的复杂场景（跨合同冲突检测）才用 Agent。生产原则：先 Workflow 跑通 80% 场景，把剩下 20% 真正需要自主决策的才上 Agent。LangGraph 的价值在于——它能让你在同一个框架里从 Workflow 渐进到 Graph 再到 Agent，不用推倒重来。"

🎯 **面试官追问**：
- "怎么判断一个场景'步骤无法预定义'？" → 看输入多样性：如果输入类型可枚举（合同/工单/报告），路径大概率可预定义；如果输入完全开放（用户自由提问 + 需要多步推理），才需要 Agent
- "Graph 和 Agent 的边界模糊，怎么选？" → 看分支可枚举性：分支能用 if-else / 状态机覆盖 → Graph；分支需要 LLM 现场决定 → Agent
- "Workflow 能不能进化成 Agent？" → 能，LangGraph 支持从 Workflow（固定边）渐进到 Graph（条件边）再到 Agent（动态边）。建议先 Workflow 验证核心链路，再逐步加自主性

⚠️ **踩坑提示**：
- 别说"我们用了 Agent 因为更智能"——这是典型的过度工程，面试官会追问"Workflow 不行吗"
- "Agent 是未来，Workflow 是过去"是错误认知——2026 共识是"能用 Workflow 别用 Agent"
- 忽略 Agent 的测试难度是高频盲点——路径组合爆炸导致测试覆盖极难，生产风险高

---

### Q37. 【Agent 可观测性进阶】生产 Agent 的 trace 要记录什么？怎么从 trace 定位"Agent 跑飞了"？

**① 原理**：Agent 可观测性比 RAG 复杂一个量级——RAG 是线性链路（检索→生成），Agent 是循环链路（决策→行动→观察→再决策），每一步都可能分叉、回退、死循环。只记"输入输出"完全不够，必须记录"决策过程 + 状态变迁 + 资源消耗"才能定位问题。

**② Trace 必须记录的五层**：

| 层 | 记录内容 | 定位什么问题 |
|----|---------|-------------|
| 决策层（Thought） | LLM 的 reasoning / tool 选择 / 参数 | "为什么选了这个工具""为什么走这条路径" |
| 行动层（Action） | 工具调用名 + 参数 + 返回值 | "工具调对了吗""参数传对了吗" |
| 状态层（State） | 状态机当前节点 + 上下文快照 | "走到哪了""上下文里有什么""为什么转移" |
| 资源层（Resource） | 每步 token / latency / cost | "哪一步烧钱""哪一步卡住" |
| 异常层（Exception） | 错误类型 + 堆栈 + 重试/降级记录 | "哪一步失败了""怎么兜底的" |

**③ 定位"Agent 跑飞"的三步法**：
1. **看状态轨迹**：Agent 走了哪些节点、有没有重复/回退/死循环。重复同一节点 > 2 次基本是"跑飞"
2. **看决策日志**：在异常节点看 LLM 的 thought——是工具选错、参数错、还是上下文污染导致推理偏
3. **看上下文快照**：异常时刻的上下文里有什么——是不是记忆召回污染、工具返回注入了错误信息

**④ 生产经验**：
- **状态回放是必备能力**：把 trace 存成可回放格式（JSON），出 bug 时能"重放" Agent 的完整执行过程，比看日志高效 10 倍
- **异常检测自动化**：设规则引擎——单次执行 token > 阈值、步数 > max_steps、同一工具调用 > 3 次、延迟 P99 突增 → 自动告警 + 归档 trace
- **Trace 采样策略**：全量 trace 存储成本高（Agent 每次执行几十步），生产中成功 case 采样 5-10%，失败 case 100% 存

**⑤ 示例**：
> "我们 Agent 平台出过一次故障——某个 Agent 任务平均 token 从 3000 飙到 25000，成本暴涨。trace 回放发现是'上下文污染 + 重复检测缺失'：Agent 第 3 步工具返回了一段很长的错误信息，被全量塞进上下文；第 4 步 Agent 基于这个错误信息重新推理，又调了一次工具返回同样的错误；因为没有重复检测，Agent 在第 3-4 步循环了 7 次（每次上下文越来越长）才触发 max_steps 终止。修复：① 加重复检测（同一工具+同一参数连续调用 > 2 次就 break）；② 工具返回超长时截断 + 摘要；③ 上下文组装加 token 预算管理。教训：Agent 不做可观测等于盲跑——出问题你根本不知道是哪一步炸的。现在我们 trace 五层全记 + 失败 case 自动回放归档，定位问题从'翻日志 2 小时'降到'回放 5 分钟'。"

🎯 **面试官追问**：
- "Trace 存哪里？成本怎么控？" → Langfuse/Phoenix 存结构化 trace（JSON），按 tenant + 任务 ID 索引；成本控制靠采样（成功 5% + 失败 100%）+ 冷热分层（7 天热 + 30 天温 + 归档）
- "怎么区分'Agent 决策错了'和'工具返回错了'？" → 看行动层 trace：工具返回值对不对（工具问题）vs 工具返回对但 LLM 推理错（harness/模型问题）
- "生产 Agent 怎么做回归测试？" → 录制 golden trace（典型场景的理想执行轨迹），每次 harness/模型变更后回放对比偏离度

⚠️ **踩坑提示**：
- 别只说"我们用了 LangSmith 做监控"——要讲清楚监控什么、怎么定位、怎么告警
- "我们记录了输入输出"不够——Agent 的核心是过程，不记决策层和状态层等于没观测
- 忽略"trace 回放"能力是高频遗漏——这是定位 Agent bug 最高效的手段

---

## 今日自测清单

> **升级版自查**：不是"能不能背下来"，而是"能不能 3 分钟脱稿讲清楚，并接住追问"。

| 题 | 自查问题 | 达标标准（开口讲） | ✓/✗ |
|----|---------|-------------------|-----|
| Q1 | Agent 四大模块 + 与普通 LLM 区别 | 能讲清"自主循环"是核心 | ⬜ |
| Q2 | ReAct 原理 + 局限 | 能讲清错误传播 + 上下文爆炸怎么办 | ⬜ |
| Q3 | ReAct vs Plan-and-Execute | 能讲清各自适用场景 | ⬜ |
| Q4 | Function Calling 三个层次 | 能讲清"不是真理解 API"的机制 | ⬜ |
| Q5 | Tool Schema 设计四原则 | 能讲清 description 为什么是 prompt | ⬜ |
| Q6 | Multi-Agent 四种拓扑 | 能讲清 Supervisor 为什么最常用 | ⬜ |
| Q7 | LangGraph vs CrewAI vs AutoGen | 能结合真实场景讲选型 | ⬜ |
| Q8 | 记忆系统三层 | 能讲清"什么写入/什么遗忘" | ⬜ |
| Q9 | 生产可靠性六件套 | 能背金句"确定性外壳包裹非确定内核" | ⬜ |
| Q11 | Agent 评估三维度 | 能讲清轨迹评估 + τ-bench/SWE-bench | ⬜ |
| Q13/Q14 | MCP 原语 + 与 Function Calling 关系 | 能讲清是"互补不是竞争" | ⬜ |
| Q29 | A2A vs MCP 分层 | 能讲清"MCP=USB-C，A2A=HTTP" | ⬜ |
| Q31 | Durable Execution | 能讲清 checkpoint 的局限 | ⬜ |
| Q34 | MCP Tool Poisoning | 能讲清"网关+沙箱+HITL 多层防御" | ⬜ |

**达标线**：14 题中 ≥ 11 题能脱稿讲 + 接住至少 1 个追问 → Day2 通过。
**未达标处理**：薄弱题回到主文件附录 A.2 重读原理，再用本文件示例答练嘴 3 遍。

## 参考资源

- [Agent面试问题总结 - Datawhale](https://github.com/datawhalechina/hello-agents/blob/main/Extra-Chapter/Extra01-%E9%9D%A2%E8%AF%95%E9%97%AE%E9%A2%98%E6%80%BB%E7%BB%93.md)
- [LangChain面试指南](https://zhuanlan.zhihu.com/p/1946533186038404675)
- [Agent面试题 - 牛客网](https://www.nowcoder.com/discuss/867373725035872256)
- LangChain 官方文档：https://python.langchain.com/
- LangGraph 官方文档：https://langchain-ai.github.io/langgraph/
- AutoGen 官方文档：https://microsoft.github.io/autogen/
