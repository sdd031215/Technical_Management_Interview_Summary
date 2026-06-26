# Day 9 — LLM 部署运维与推理优化实战

> 目标：掌握推理框架选型、GPU 规划、分布式推理、成本优化的面试要点
> 全天技术深度，配合面试投递持续进行

---

## 🔗 与主文件《3 周版》的联动（避免重复，先读这里）

本文件是 Day9 的**深化执行手册**，技术原理的权威答案在主文件附录，本文件聚焦"面试场上怎么答"：

| 你要做的 | 去哪找 | 本文件补充什么 |
|---------|--------|---------------|
| 查技术原理深度 | 主文件 **附录 A.3**（大模型基础与推理部署） | ❌ 不重复 |
| 查答题框架 | 主文件 **附录 B.3**（Q13-Q16 推理部署题） | ✅ 补面试官追问 + 示例答 |
| 查成本优化策略 | 主文件 **附录 A.3.5** | ✅ 补口述话术 |
| 做半日时段练习 | 主文件 **第一周 Day 4**（已含本主题半日表） | ✅ 本文件做"题库深化" |

**阅读顺序建议**：先过主文件附录 A.3 建立知识框架 → 再用本文件做面试场景演练。

---

## 今日安排

| 时段 | 内容 | 产出 |
|------|------|------|
| 上午 9:00-12:00 | 推理框架对比 + GPU 选型 + 量化部署 | 技术笔记 |
| 下午 14:00-16:00 | 分布式推理 + 成本优化 + 服务化部署 | 架构决策表 |
| 下午 16:00-17:00 | 投递跟进 + 面试安排确认 | 投递记录 |
| 晚上 19:00-20:00 | 自测 + 薄弱点标记 | 自测记录 |

> **图标约定**（贯穿本文件每道题）：
> 🎯 **面试官追问**——这道题答完，面试官真正会接着问的连环问题
> ⚠️ **踩坑提示**——常见减分回答 / 误区
> 💬 **示例答**——口述稿风格，可直接练嘴，括号里替换成你的真实数据

---

## 核心面试题详解

### Q1. 五大推理框架（vLLM/TensorRT-LLM/llama.cpp/TGI/Ollama）如何选型？

**答案：**

| 维度 | vLLM | TensorRT-LLM | llama.cpp | TGI | Ollama |
|------|------|-------------|-----------|-----|--------|
| 核心技术 | PagedAttention + Continuous Batching | NVIDIA 编译器优化 | 纯 C/C++，GGUF 格式 | Rust + HPU 引擎 | 基于 llama.cpp 封装 |
| 硬件依赖 | NVIDIA GPU | 仅 NVIDIA GPU | CPU/GPU/Apple Silicon | NVIDIA GPU | CPU/GPU/Apple Silicon |
| 吞吐量 | 高 | **最高** | 中等 | 高 | 低 |
| 冷启动 | ~62 秒 | ~28 分钟 | 几秒 | 中等 | ~3-5 秒 |
| 部署复杂度 | 低 | 高 | 低 | 中 | 极低 |
| 适用场景 | **生产环境首选** | 极致性能 | 边缘/CPU | HuggingFace 生态 | 开发调试 |

**架构师视角**：实际生产往往组合使用——线上用 vLLM，边缘用 llama.cpp，开发用 Ollama，通过 AI Gateway 统一路由。

🎯 **面试官追问**：
- "你实际生产用过哪个？为什么没选 TensorRT-LLM？" → 考察"是否真上过线"
- "vLLM 的 Continuous Batching 和传统 static batching 具体差在哪？"（→ 见 Q2）
- "如果你们模型是 MoE 架构，框架选型会变吗？" → vLLM/SGLang 对 MoE 支持差异

⚠️ **踩坑提示**：
- 别只报菜名"vLLM 最好"——要讲**你的决策依据**（团队运维能力、硬件、SLA）
- 别贬低 TensorRT-LLM——它确实最快，只是部署重、编译慢，是 tradeoff 不是缺点
- 不要说"没用过其他框架，只看过文档"——会暴露缺乏实战

💬 **示例答**：
> "我们线上推理用 vLLM，主要三个原因：一是 PagedAttention 把显存利用率从 30% 提到 90% 多，同样 GPU 能扛更多并发；二是部署相对轻，不像 TensorRT-LLM 要编译半小时、调参痛苦，我们团队规模（X 人）hold 不住；三是社区活跃，遇到坑能找到解法。开发同学本地用 Ollama 起小模型，边缘场景用 llama.cpp。TensorRT-LLM 我们评估过，对延迟极致敏感的场景（比如实时语音）才会考虑，目前还没到那个量级。"

---

### Q2. PagedAttention 和 Continuous Batching 分别解决什么问题？

**答案：**

**PagedAttention**：解决 KV Cache 显存碎片化
- 将 KV Cache 切分为固定 block（如 16 tokens），通过 block table 映射
- 不同请求可共享公共前缀（如 system prompt）
- 效果：显存利用率从 20-40% 提升到 ~96%

**Continuous Batching**：解决请求调度效率
- token 级别动态调度：完成即移出、新请求即插入
- 支持 Preemption：显存不足时低优先级请求换出到 CPU
- 效果：高并发场景吞吐提升 2-4 倍

🎯 **面试官追问**：
- "PagedAttention 的 block size 怎么定？大了小了有什么影响？" → 小 block 灵活但元数据开销大，大 block 省元表但碎片化回升
- "Continuous Batching 下，长短请求混部会不会有队头阻塞？" → 会，所以有 priority queue 和 preemption 机制
- "Preemption 把 KV Cache 换出到 CPU，再换回来不会很慢吗？什么时候触发？" → 是兜底机制，正常调度不该频繁触发，监控换出率

⚠️ **踩坑提示**：
- 别把 PagedAttention（显存）和 Continuous Batching（调度）混为一谈，面试官常用来设陷阱
- "提升了 2-4 倍"这种数字要标"高并发场景下"，单并发其实没差别

💬 **示例答**：
> "这俩解决的是不同问题。PagedAttention 解决显存碎片——传统做法每个请求预分配一整块连续 KV Cache，但生成长度不定，要么浪费要么溢出；vLLM 借鉴 OS 虚拟内存，把 KV 切成固定 block 用 block table 映射，显存利用率从 30% 多直接拉到 90% 多。Continuous Batching 解决的是调度——传统 batch 要等一批都生成完才能接新请求，长短请求混部时 GPU 大量空转；continuous 是 token 级别动态拼批，谁完成谁走，新请求随时插队。两个加起来，我们高并发场景吞吐大概提了 3 倍。"

---

### Q3. A100、H100、L40S 三款 GPU 如何选型？

**答案：**

| 参数 | A100 80GB | H100 80GB SXM | L40S 48GB |
|------|-----------|---------------|-----------|
| 显存带宽 | 2,039 GB/s | **3,350 GB/s** | 864 GB/s |
| FP16 算力 | 312 TFLOPS | **989 TFLOPS** | 362 TFLOPS |
| FP8 支持 | 否 | **是** | 是 |
| NVLink | 是 | 是 | 否 |
| 参考价 | ~$10K | ~$25-30K | ~$8-10K |

**显存计算公式**：
```
总显存 = 模型权重 + KV Cache + 激活值 + 框架开销(15-20%)

权重: FP16 = 参数量×2B, FP8 = 参数量×1B, INT4 = 参数量×0.5B
KV Cache/请求: 2 × layers × kv_heads × head_dim × seq_len × 2B
```

**实例（Llama 3.1 70B）**：
- FP16: 140GB → 2× A100/H100
- FP8: 70GB → 1× H100
- INT4: 35GB → 1× L40S

**选型建议**：H100 大模型首选（FP8 单卡跑 70B）；A100 性价比之选；L40S 推理性价比最优。

🎯 **面试官追问**：
- "现场算一下：Qwen 72B FP8，batch=8，seq_len=4096，要多少 H100？" → 考察你能否现场用公式算（权重 72GB + 8 路并发 KV Cache，单 H100 80GB 放不下，要 2 卡 TP）
- "KV Cache 为什么占显存这么大？和 batch、seq_len 是什么关系？" → 线性增长，是高并发主要瓶颈
- "为什么不用消费级卡（4090）？" → 显存 24G 太小、无 NVLink、ECC 缺失、产能/合规问题

⚠️ **踩坑提示**：
- 显存公式别只背"参数量×2B"，**漏掉 KV Cache 是最常见的扣分点**——大模型推理 KV Cache 往往比权重还大
- 别说"70B 用一张 A100 就行"——FP16 140GB 根本放不下 80GB 卡，必须 2 卡或量化
- 价格数字记不准没关系，但**趋势要对**：H100 > A100 > L40S

💬 **示例答**：
> "推理显存主要三块：权重 + KV Cache + 激活值。权重是固定开销，FP16 下 70B 大概 140GB；KV Cache 是随并发和上下文长度线性增长的，公式是 2 × layers × kv_heads × head_dim × seq_len × batch × 2B，这块往往被低估——长上下文高并发场景，KV Cache 能是权重的 1-2 倍。所以我们部署 Qwen 72B 用 FP8 + H100，权重压到 72GB 单卡能放，再把 KV Cache 用 PagedAttention 管理，单卡能扛 X 路并发。"

---

### Q4. 如何设计生产级 LLM API Gateway？

**答案：**

五层架构：
```
接入控制层（认证/限流/请求校验）
  → 智能路由层（任务类型/成本/延迟路由）
    → 模型容错层（重试/超时切换/熔断器）
      → 内容治理层（PII 脱敏/审核/缓存）
        → 可观测性层（Trace/指标/审计）
```

**核心组件**：
- **限流**：Token Bucket / Sliding Window，按用户/模型/时间段
- **路由**：简单→小模型，复杂→大模型；实时→就近，批量→成本最优
- **容错**：主模型超时→自动 fallback 备用模型；熔断器模式
- **缓存**：语义缓存降低重复查询成本

**主流方案**：LiteLLM、Portkey、Kong AI Gateway

🎯 **面试官追问**：
- "语义缓存怎么避免误命中？"（query 相似但答案不同）→ 加业务维度过滤 + 相似度阈值校准 + 关键字段精确匹配前置
- "模型 fallback 时，prompt 格式不兼容怎么办？"（GPT→Claude 函数调用格式不同）→ Gateway 做协议转换层
- "多租户场景下，怎么隔离不同租户的配额和成本？" → 按 tenant_id 限流 + token 计费独立 + 缓存 namespace 隔离

⚠️ **踩坑提示**：
- 别只讲"五层架构"就结束——面试官想听**每层的关键决策**（限流用什么算法、缓存命中率多少、fallback 策略）
- "语义缓存命中率 15-30%"要标前提（FAQ 类场景高，开放对话类低），别当万能数字
- 这题是 LLM Gateway 系统设计题的预热，深度可参考主文件附录 **B.19**（LLM 网关/模型路由层）

💬 **示例答**：
> "我们把 LLM 当不稳定依赖来工程化，自建了一层 Gateway，核心五件事：一是统一鉴权和配额，按租户和模型分别限流；二是路由，简单 query 走小模型、复杂的走大模型，用分类器预判难度；三是缓存，精确缓存加语义缓存，FAQ 类场景命中 30%+；四是容错，主模型超时自动 fallback 到备用 provider，熔断器防雪崩；五是全链路 trace，按接口/用户/场景统计 token 和成本，做成本看板。这层让我们后面换模型、调配比都不用动业务代码。"

---

### Q5. LLM 推理的负载均衡与传统 Web 服务有何不同？

**答案：**

| 维度 | 传统 Web | LLM 推理 |
|------|---------|---------|
| 请求特征 | 短请求、无状态 | 长请求、有状态（KV Cache） |
| 冷启动 | 秒级 | 分钟级 |
| 请求成本差异 | 小 | 巨大（10 token vs 10K token） |

**负载均衡策略**：
1. **Least-Connections**：路由到活跃请求最少的实例
2. **KV Cache-Aware Routing**：共享前缀的请求路由到同一实例（SGLang RadixAttention）
3. **Priority-Based Queue**：付费用户优先、实时优先于批量
4. **Prefill-Decode 分离**：计算密集和显存密集阶段分到不同 GPU

**弹性伸缩**：
- GPU 利用率 >70% 扩容，<30% 缩容
- 预留实例（基线）+ Spot 实例（弹性）+ 渐进式扩缩

🎯 **面试官追问**：
- "Prefill-Decode 分离具体怎么做的？为什么能提性能？" → Prefill 计算密集（吃算力），Decode 显存密集（吃带宽），分开调度让两类 GPU 都打满
- "SGLang 的 RadixAttention 和普通 Prefix Cache 区别？" → 用 Radix Tree 复用共享前缀的 KV，多用户共享 system prompt 时收益大
- "GPU 扩容这么慢（拉镜像+加载权重几分钟），怎么应对突发流量？" → 预热实例池 + 流量预测 + 排队机制

⚠️ **踩坑提示**：
- 别忽略"有状态"这个核心差异——传统 LB 轮询会把同一会话打到不同实例，KV Cache 全失效
- Least-Connections 在 LLM 场景不一定对——长请求会把某实例占满，要用 KV-aware routing
- 扩缩容阈值（70%/30%）是经验值，要说"会根据 P99 延迟动态调"，体现工程判断

💬 **示例答**：
> "LLM 推理负载均衡和传统 Web 服务最本质的区别是请求有状态——KV Cache 绑在某个实例上。传统轮询或最少连接会把同一会话的续接请求打到别的实例，KV Cache 全失效，等于每次都从头算。所以我们用 KV-aware routing，相同前缀（比如同一个 system prompt 或同一会话）尽量路由到同一实例；对延迟敏感的 prefill 和显存敏感的 decode 做分离调度；扩容不是简单看 CPU，而是看 KV 占用率和排队深度，而且因为拉权重慢，我们维护一个预热实例池应对突发。"

---

### Q6. AWQ 和 GPTQ 量化的原理和精度对比？

**答案：**

| 维度 | GPTQ | AWQ |
|------|------|-----|
| 原理 | 二阶信息（Hessian），逐层补偿 | 激活值分布识别重要权重 |
| 4-bit 精度损失 | 1-3% | **0.5-2%** |
| 推理速度提升 | 1.5-2x | **2-3x** |
| 显存节省 | ~75% | ~75% |
| 量化速度 | 较慢 | 中等 |

**选型建议**：追求速度→AWQ，追求稳定性→GPTQ，H100→FP8 原生，CPU/边缘→GGUF

🎯 **面试官追问**：
- "AWQ 为什么比 GPTQ 精度更好？原理上的差别？" → AWQ 基于激活值分布识别"重要权重"保持高精度，GPTQ 是逐层 Hessian 补偿
- "量化后业务指标掉了怎么办？" → 别只看 perplexity，要在业务 golden set 上测；掉了就换高 bit 或混合精度（敏感层不量化）
- "FP8 和 INT8/INT4 是一回事吗？" → 不是，FP8 是 H100 硬件原生支持的浮点格式，精度好于 INT8；INT4 是整数

⚠️ **踩坑提示**：
- 别说"量化精度损失 1-3% 可以忽略"——**业务指标可能放大**，比如 RAG 召回率掉 1%，端到端答案准确率可能掉 5%
- "INT4 显存降 75%"要补一句"但不是所有模型都扛得住 4bit，小模型（7B 以下）量化损失明显"
- 别忘了**评估方法**：用业务 golden set 测，而不是看论文的 perplexity

💬 **示例答**：
> "量化我们主要用 AWQ 4bit，相比 GPTQ 推理更快、精度损失更小——原理是 AWQ 看激活值分布识别哪些权重重要、保持高精度，不重要的压到 4bit。但量化不是无脑上，我们有一套业务 golden set 做回归——量化前后跑一遍，如果关键指标掉了超过 X% 就回退，或者对敏感层用混合精度。FP8 我们在新上的 H100 上试，硬件原生支持精度更好，但要求新卡，老 A100 用不了。"

---

### Q7. Tensor Parallelism、Pipeline Parallelism、Data Parallelism 如何选择？

**答案：**

| 策略 | 原理 | 优势 | 劣势 | 适用 |
|------|------|------|------|------|
| **TP（层内切分）** | 每层权重矩阵切分到多 GPU | 延迟最低 | 通信量大，需 NVLink | 单机多卡+低延迟 |
| **PP（层间切分）** | 不同层分配到不同 GPU | 通信量小，可跨机 | pipeline bubble | 多机部署 |
| **DP（实例复制）** | 每 GPU 完整模型副本 | 线性扩展吞吐 | 每卡需放下完整模型 | 模型能放进单卡 |

**3D 并行组合（70B 模型，2 台机器×8 GPU）**：
- TP=8（单机 NVLink）+ PP=2（跨机 RDMA）+ DP=1

**决策框架**：单卡放得下→DP；单机放得下→TP；单机放不下→TP+PP

**Expert Parallelism（MoE 专用）**：不同 Expert 分布到不同 GPU

🎯 **面试官追问**：
- "TP 通信量为什么大？每层都要通信吗？" → 是，每层 attention/FFN 后要做 all-reduce，所以强依赖 NVLink 带宽，跨机 TP 基本不可行
- "Pipeline 的 bubble 怎么缓解？" → micro-batching（把一个 batch 切成多份流水）+ 1F1B 调度
- "如果只有普通以太网（无 RDMA），你怎么部署大模型？" → 只能 PP + DP，TP 退化到单机内

⚠️ **踩坑提示**：
- 决策框架"单卡放得下→DP；单机放得下→TP；单机放不下→TP+PP"要背熟，这是高频考点
- 别忽略**通信带宽**这个隐藏约束：TP 没 NVLink 等于没 TP
- MoE 模型的 EP（Expert Parallel）是新趋势，DeepSeek/Mixtral 类模型常考，体现前沿敏感度

💬 **示例答**：
> "并行策略核心看模型放哪。单卡放得下就 DP，复制多份线性扩吞吐，最简单；单机多卡放得下（比如 8 卡）就 TP，把每层权重矩阵切开，延迟最低但每层都要 all-reduce，强依赖 NVLink；单机放不下才上 PP，按层切到不同机器，通信少但有 pipeline bubble，要用 micro-batching 缓解。我们 70B 模型 2 台 8 卡机器，用 TP=8 单机内 + PP=2 跨机 + DP=1，跨机走 RDMA。MoE 模型还会加 EP，把不同 expert 分布到不同卡。"

---

### Q8. LLM 推理成本优化的三层策略？

**答案：**

**基础设施层**：
- Spot 实例（1/3~1/2 价格）+ 预留实例（基线）混合部署
- 硬件选型：L40S 推理性价比优于 A100（成本低 30-50%）
- FP8 + H100 单卡 > FP16 + 2×A100（70B 模型成本减半）

**模型层**：
- 模型蒸馏：大模型生成数据训练小模型（10K-50K 样本即可）
- 量化部署：INT4 显存降 75%，允许更便宜的 GPU
- Speculative Decoding：小模型 draft + 大模型验证，加速 2-3 倍

**应用层**：
- 请求分级路由：80% 简单请求→小模型，20% 复杂→大模型
- 语义缓存：命中率 15-30%
- Prompt 优化：精简 system prompt，合理 max_tokens
- 批处理异步化：Batch API 成本通常是实时的 50%

🎯 **面试官追问**：
- "模型路由用什么判断简单/复杂？" → 轻量分类器（BERT 级）或 LLM 路由（小模型先判），tradeoff 是延迟和准确率
- "Speculative Decoding 在什么场景收益最大？有副作用吗？" → 确定性高、可预测性强的任务收益大；副作用是 draft 模型选不好反而拖慢
- "成本优化的天花板在哪？" → 在效果不降的前提下，缓存 + 路由 + 批处理能砍 50-70%，再往下要动模型本身（蒸馏/量化）

⚠️ **踩坑提示**：
- 别只讲技术手段，**要讲你怎么量化效果**（"成本看板按场景归因，每月省 X 万"）——这是 Tech Lead 视角
- "语义缓存命中 30%"是 FAQ 场景，开放对话可能 5% 都没有，要说清前提
- 成本优化是主文件附录 **A.3.5** 和 **B.13** 的核心，深度答案去那里看

💬 **示例答**：
> "降本是系统工程，我们从三层下手。基础设施层：Spot + 预留混合，FP8+H100 替代 FP16+A100 把 70B 单卡跑下来，硬件成本砍一半。模型层：蒸馏小模型接管简单任务，INT4 量化让便宜卡也能跑。应用层收益最大——做模型路由，用轻量分类器把 80% 简单 query 导到小模型；语义缓存对 FAQ 类命中 30%；非实时的走 batch API 省 50%。这套组合下来，我们 per-query 成本从 X 降到 Y，同时效果指标没掉。关键是建了成本看板，按接口/场景归因，不然优化无从下手。"

---

### Q9. 如何设计高可用 LLM 推理服务？

**答案：**

**降级策略分级**：
- Level 1：切换备用同型号模型（用户无感知）
- Level 2：切换小模型（质量略降但可用）
- Level 3：返回语义缓存历史回答（标注"非实时"）
- Level 4：排队模式，告知预计等待时间

**SLA 保障**：
| 指标 | 目标 | 手段 |
|------|------|------|
| 可用性 | 99.95% | 多活部署+自动故障转移 |
| TTFT P95 | <2s | 就近路由+预热+Prefix Caching |
| 降级 | 分级降级 | 主→备→缓存→排队 |

🎯 **面试官追问**：
- "降级到小模型时，效果掉了业务方投诉怎么办？" → 提前和业务方约定降级 SLA + 降级时打标 + 快速恢复机制
- "99.95% 可用性，单实例肯定做不到，多活怎么解决模型一致性？" → 模型只读、无状态，多活主要是流量切换；配比/灰度配置走配置中心统一推
- "LLM 服务的事务一致性怎么保证？比如工具调用中途失败？" → 幂等设计 + 持久化中间状态（state machine）+ 补偿/重试，别追求 ACID

⚠️ **踩坑提示**：
- 别把"分级降级"讲成简单的 fallback——**关键是事先和业务方约定降级契约**（什么场景降、降级后质量底线）
- LLM 没有传统意义的"事务"，别套用数据库那套，面试官想看你理解 LLM 的状态模型
- 99.95% 是月度 SLA，对应月宕机 ~22 分钟，要说清统计口径

💬 **示例答**：
> "LLM 高可用的难点是模型大、启动慢，不能像 web 服务那样随便重启。我们做四级降级：L1 切备用同型号模型（用户无感）；L2 切小模型（质量降但可用，事先和业务方约好降级时的质量底线）；L3 返回语义缓存历史答案（标注'非实时'）；L4 排队告知等待时间。这套契约提前和业务对齐，不是出事才商量。多活部署保证 99.95%，关键是模型只读无状态，故障转移就是切流量。工具调用这种有状态的，我们用状态机持久化中间步骤，失败能从断点续。"

---

### Q10. 如何评估推理框架性能？关键指标有哪些？

**答案：**

| 指标 | 含义 | 重要性 |
|------|------|--------|
| TTFT | 首 token 延迟 | 用户体感速度 |
| TPS | 每秒生成 token 数 | 长文本等待时间 |
| Throughput | 系统总吞吐 | 并发承载能力 |
| Cost per 1M tokens | 每百万 token 成本 | 商业可行性 |
| GPU Utilization | GPU 利用率 | 资源浪费程度 |

**Benchmark 方法**：选代表性 prompt → 固定模型精度 → 变化并发数 → 测量各指标 → 长稳测试（>1 小时）

**常见陷阱**：不要只看 batch=1 的 TPS；注意冷启动对 P99 影响

🎯 **面试官追问**：
- "TTFT 和 TPS 哪个对用户体验更重要？" → 在线 chat 场景 TTFT 决定首屏体感，长文本生成 TPS 决定总等待；不同场景权重不同
- "Throughput 和单请求延迟是矛盾的吧？怎么平衡？" → 是，批处理提吞吐但增单请求延迟；按业务取舍（在线重 P99，离线重吞吐）
- "怎么测才不会被供应商的 benchmark 骗？" → 用自己的真实 prompt 分布测，别用供应商的公开数据集

⚠️ **踩坑提示**：
- 别只背指标定义——面试官想听**每个指标怎么影响业务决策**（比如 TTFT 高了用户流失率怎么变）
- "batch=1 的 TPS"是供应商最爱报的漂亮数字，生产场景要看真实并发下的吞吐和 P99
- benchmark 的 prompt 分布很关键，长 prompt 和短 prompt 结果差几倍

💬 **示例答**：
> "我们评估推理框架看 5 个指标：TTFT（首 token 延迟，决定首屏体感）、TPS（生成速度）、Throughput（总吞吐，决定并发承载）、Cost per 1M tokens（商业可行性）、GPU 利用率。测法上我们吃过亏——早期看供应商报的 batch=1 TPS 觉得很快，上线发现并发一上来 P99 炸了。后来我们用自己的真实 prompt 分布做 benchmark，固定模型精度，从并发 1 拉到并发 64，每个档位测各指标，再跑 1 小时长稳测试看会不会退化。重点是别信公开 benchmark，用自己的业务数据测。"

---

## 深度问题补充（高频难题 + 前沿趋势）

### Q11. 【底层原理】vLLM 的 PagedAttention 源码级原理？block table 怎么工作？

**① 原理**：借鉴 OS 虚拟内存分页——把 KV Cache 切成固定大小 block（默认 16 tokens），用 block table 映射逻辑 block_id → 物理 block 地址。KV 不需物理连续，按需分配，消除内部碎片。

**② 数据结构**：
```
逻辑视图（请求看到的）: [b0][b1][b2][b3]...（连续逻辑地址）
block table: [0→物理p2][1→p7][2→p1][3→p5]（逻辑到物理映射）
物理显存（实际存储）: p0,p1,p2,...,pN（按需分配，可不连续）
```

**③ 生产经验**：
- block size 默认 16 token，是显存碎片 vs 元数据开销的平衡点
- Copy-on-Write：多请求共享 system prompt 的 KV，某请求改了某 token 才复制该 block
- 内存利用率从传统 20-40% 提升到 90%+

**④ Tradeoff**：
- 灵活性 vs 访问开销：block table 间接寻址略增开销，但省的显存远大于此
- block size：太小元数据膨胀，太大碎片回升

**⑤ 示例**：
> "PagedAttention 的精妙是把 OS 的虚拟内存思想搬到 GPU。我们 70B 部署，传统方式预分配 max_seq_len KV 浪费 60% 显存；PagedAttention 按需分配，同样显存多扛 3 倍并发。block table 是核心——逻辑上请求看到连续 KV，物理上散落在各处，attention 计算时通过 table 间接寻址。Copy-on-Write 让共享 system prompt 的请求复用 KV，进一步省。"

---

### Q12. 【生产故障】TP 并行训练/推理时，某张卡慢导致整体被拖累（straggler），怎么治？

**① 原理**：TP（Tensor Parallelism）每层都要 all-reduce 同步，任一卡慢（straggler）会拖累全链路——木桶效应。PP 和 DP 类似。

**② 定位方法**：
- NCCL benchmark：测卡间带宽，找异常链路
- per-rank trace：每卡的 compute/communication 时间，找慢卡
- 常见根因：①硬件故障（某卡降频）②NVLink 拓扑不均（非全互联）③热 throttling ④其他进程抢占

**③ 生产经验**：
- 部署前做 NCCL test，剔除有问题的卡/链路
- 监控 GPU 温度和频率，throttling 是隐形杀手
- TP 尽量在单机 NVLink 内（8 卡全互联），跨机用 PP
- straggler 检测：设延迟阈值，超限告警 + 自动迁移

**④ Tradeoff**：
- 性能 vs 容错：加冗余卡可容错但浪费；严格剔除坏卡提升整体
- TP vs PP：TP 通信频繁但延迟低，PP 通信少但有 bubble

**⑤ 示例**：
> "我们 8 卡 TP 部署遇到过：某张卡温度过高降频 20%，整个推理延迟翻倍。排查靠 per-rank trace 一眼看到那张卡 compute 时间明显长。根治：①机柜风道改造降温 ②监控加 GPU 频率告警 ③关键服务用 9 卡部署（8 工作 + 1 热备），坏卡自动切换。教训：TP 的木桶效应极强，一张卡慢全盘慢，必须有 per-rank 可观测。"

---

### Q13. 【前沿趋势】推理模型（o1/R1）的部署和普通模型有什么不同？怎么优化？

**① 原理**：推理模型生成大量隐藏"思考 token"（几千到几万），推理特征和普通模型差异大——①总 token 数高 5-20 倍 ②思考阶段不流式输出（用户看不到）③输出阶段才开始流式。

**② 部署挑战与解法**：

| 挑战 | 普通模型 | 推理模型 | 解法 |
|------|---------|---------|------|
| 总 token | 几百 | 几千-几万 | KV Cache 容量翻倍 |
| 延迟 | 首 token 快 | 思考阶段长 | 前端"思考中"动画 |
| 成本 | 低 | 高 5-20 倍 | 严格模型路由，只给难题 |
| 流式 | 全程流式 | 思考不流式 + 答案流式 | 分阶段流式协议 |

**③ 生产经验**：
- **模型路由必须**：90% query 走普通模型，10% 难题走推理模型，否则成本爆炸
- **思考预算**：设 max_thinking_tokens 上限，防过度思考
- **前端体验**：思考阶段显示"正在思考 X 秒"+可选展开思考过程，缓解用户焦虑
- **缓存思考**：相似问题的思考过程可缓存复用

**④ Tradeoff**：
- 能力 vs 成本：推理模型强但贵，必须路由分发
- 透明 vs 简洁：暴露思考过程增加信任但可能泄露推理模式

**⑤ 示例**：
> "部署 DeepSeek-R1 做复杂工单，和普通模型分开部署。关键设计：①模型路由——轻量分类器预判难度，只 8% 难题走 R1 ②思考预算上限 8K token，防烧钱 ③前端分阶段流式：思考阶段显示'正在分析...'进度条，思考完才开始流式答案 ④KV Cache 容量按 4 倍预估。这样 R1 单卡成本虽高，但只处理 8% 流量，整体成本可控，难题解决率提升 30%。"

---

### Q14. 【权衡决策】自建推理集群 vs 用 API（GPT/Claude），什么时候自建才划算？

**① 原理**：核心公式——**盈亏平衡点 = 自建固定成本 ÷ (API 单价 - 自建边际成本)**。低于这个量用 API 划算，高于则自建。

**② 量化对比（以 70B 模型、年查询量为例）**：

| 维度 | API（GPT-4o 级） | 自建（70B + vLLM） |
|------|-----------------|------------------|
| 起步成本 | 0 | GPU + 运维（高） |
| 边际成本 | $2-5/1M token | $0.3-0.8/1M token |
| 盈亏平衡点 | — | 约 5000 万-1 亿 token/月 |
| 数据合规 | 数据出境 | 私有化 |
| 能力上限 | 顶级 | 受开源模型限制 |
| 运维负担 | 0 | 高（需专职团队） |

**③ 生产经验决策**：
- **量小 + 要顶级能力** → API（省心）
- **量大（> 1 亿 token/月）+ 可接受开源能力** → 自建（省钱）
- **数据合规强制** → 必须自建（无选）
- **混合最优**：敏感数据自建 + 非敏感/难题走 API

**④ Tradeoff**：
- 成本 vs 能力：自建便宜但开源能力有上限；API 贵但能力顶
- 灵活 vs 稳定：API 随时换模型；自建要自己升级
- 可控 vs 运维：自建完全可控但运维重

**⑤ 示例**：
> "我们算过账：月 3 亿 token，用 GPT-4o 约 60 万/月；自建 70B（4 卡 H100 + 运维）固定 30 万/月 + 边际 8 万 = 38 万。自建年省 264 万，且数据合规。但前提是量大——如果月只有 1000 万 token，API 2 万/月远低于自建 30 万固定成本。所以我们的策略：核心高频流量自建（3 亿），长尾复杂流量走 API（2000 万），混合最优。"

---

### Q15. 【生产实战】TTFT（首 token 延迟）怎么优化？从 5 秒压到 1 秒要做哪些事？

**① 原理**：TTFT = 网络 + 鉴权 + 队列 + Prefill（首 token 计算）。Prefill 是大头——处理 prompt 的计算量，长 prompt 慢。

**② 五层优化**：

| 层 | 手段 | 收益 |
|----|------|------|
| 网络 | 就近接入 + 连接复用 | 省 100-300ms |
| 队列 | 优先级调度 + 弹性扩容 | 队列等待从秒级到 100ms |
| Prefill | Prefix Caching（复用 system prompt KV） | 长 system prompt 省 50-80% |
| Prefill | Chunked Prefill（切分与 decode 交错） | 避免长 prefill 阻塞 |
| 模型 | 蒸馏小模型处理简单 query | prefill 快 3-5 倍 |

**③ 生产经验优化顺序**（按 ROI）：
1. 先开 Prefix Caching（最便宜，system prompt 长 when 收益巨大）
2. 再做模型路由（简单 query 走小模型）
3. 然后 Chunked Prefill（需框架支持，如 vLLM/SGLang）
4. 最后就近部署 + 弹性扩容（基础设施投入）
5. Speculative Decoding 是最后大招（复杂但提 TTFT 明显）

**④ Tradeoff**：
- TTFT vs 吞吐：优化 TTFT 的某些手段（如减小 batch）会降吞吐
- 成本 vs 延迟：小模型快但能力弱；多区域部署快但贵

**⑤ 示例**：
> "我们把 TTFT 从 4.2s 压到 0.9s。第一步 Prefix Caching（system prompt 2K token 复用 KV）省 1.5s；第二步模型路由（70% 简单 query 走 7B）省 1s；第三步 Chunked Prefill（长 prefill 切分不阻塞）省 0.5s；第四步同城多活 + 队列优先级省 0.3s。关键是按 ROI 排序——Prefix Caching 零成本收益最大，先做。Speculative Decoding 我们评估了但没上，因为实现复杂且当前 TTFT 已达标。"

---

### Q16. 【前沿趋势】SGLang 的 RadixAttention 和 vLLM 的 Prefix Cache 有什么区别？

**① 原理**：
- **vLLM Prefix Cache**：自动检测相同前缀（如 system prompt），复用其 KV block，是 PagedAttention 的扩展
- **SGLang RadixAttention**：用 Radix Tree（基数树）管理所有历史 KV 前缀，多用户/多会话共享前缀时复用率更高

**② 量化对比**：
- 单用户重复前缀：两者效果相近
- 多用户共享 system prompt：RadixAttention 复用率更高（树结构高效匹配）
- 多轮对话（前缀逐轮增长）：RadixAttention 增量复用更优

**③ 生产经验**：
- 通用场景（单轮 + 固定 system prompt）→ vLLM Prefix Cache 够用
- 多轮对话密集 / 多 Agent 共享上下文 → SGLang RadixAttention 收益大
- 结构化输出（JSON Schema）→ SGLang 的压缩有限状态机也加分

**④ Tradeoff**：
- 通用性 vs 专用优化：vLLM 生态更广；SGLang 在特定场景更快
- 成熟度：vLLM 更成熟稳定；SGLang 较新但迭代快

**⑤ 示例**：
> "我们多轮 Agent 场景从 vLLM 切到 SGLang，TTFT 降 30%。原因是 RadixAttention 对增量前缀复用更好——多轮对话每轮前缀增长，SGLang 能复用历史轮次的 KV，vLLM 的 Prefix Cache 只复用固定前缀。但通用 RAG 场景两者差不多。判断：多轮/共享上下文场景选 SGLang，通用单轮选 vLLM（生态更稳）。"

---

### Q17. 【前沿趋势·2026高频】PD 分离（Prefill-Decode 分离）架构是什么？为什么能降 60% 成本？

**① 原理**：LLM 推理分两阶段——Prefill（处理 prompt，计算密集，吃算力）和 Decode（生成 token，访存密集，吃带宽）。两者资源特征完全不同，传统合并部署会导致**资源错配**（要么 Prefill 算力闲置，要么 Decode 带宽闲置）。PD 分离把两阶段拆到不同实例，各自最优配置。

**② 量化对比**：

| 维度 | 合并部署 | PD 分离 |
|------|---------|---------|
| Prefill 实例 | 共享 | 计算型 GPU（H100 高算力） |
| Decode 实例 | 共享 | 访存型 GPU（A100 高带宽） |
| 资源利用率 | 30-50% | 70-90% |
| 成本 | 基准 | 降 40-60% |
| 复杂度 | 低 | 高（需跨实例传 KV Cache） |

**③ 核心挑战与解法**：
- **KV Cache 传输**：Prefill 完成后要把 KV Cache 传给 Decode 实例（大！）→ 用 RDMA/NIXL 高速传输，或 KV Cache 量化压缩
- **调度路由**：哪个请求分给哪个 Prefill/Decode 实例 → 中心调度器 + 负载均衡
- **弹性伸缩**：Prefill 和 Decode 独立扩缩（ Prefill 突增算力，Decode 突增并发）

**④ 生产经验**：
- **适合高并发 + 长上下文**：长 prompt 场景 Prefill 重，分离收益大
- **不适合低 QPS**：分离增复杂度，低 QPS 收益覆盖不了运维成本
- **实现**：llm-d、DeepSeek、Mooncake（Moonshot）已有生产实践
- **混合部署**：部分场景分离（长 prompt），部分合并（短 prompt）

**⑤ 示例**：
> "我们长上下文 RAG（prompt 平均 8K）切 PD 分离后成本降 55%。原来合并部署：H100 既算 Prefill 又算 Decode，Prefill 时算力打满但带宽闲置，Decode 时带宽打满但算力闲置，整体利用率 40%。分离后：Prefill 用 4 卡 H100（算力型），Decode 用 8 卡 A100（带宽型），各自利用率 80%+。挑战是 KV Cache 传输——我们用 RDMA + 量化压缩，传输延迟从 200ms 降到 30ms。判断：长上下文 + 高并发场景 PD 分离是 2026 的标配优化。"

---

### Q18. 【底层原理·2026高频】面试官让你手写一个带 KV Cache 的 Attention，怎么写？

**① 原理**：这是 2026 高频代码题，考察对 Attention + KV Cache 原理的深度理解。核心是：缓存历史 K/V，每步只算新 token 的 Q/K/V 并 append。

**② 手写实现（PyTorch 伪代码，面试可默写）**：

```python
import torch
import torch.nn as nn
import math

class AttentionWithKVCache(nn.Module):
    def __init__(self, dim, n_heads):
        super().__init__()
        self.n_heads = n_heads
        self.head_dim = dim // n_heads
        self.W_Q = nn.Linear(dim, dim)
        self.W_K = nn.Linear(dim, dim)
        self.W_V = nn.Linear(dim, dim)
        self.W_O = nn.Linear(dim, dim)

    def forward(self, x, kv_cache=None):
        B, T, D = x.shape  # T=1 for decode step

        Q = self.W_Q(x).view(B, T, self.n_heads, self.head_dim).transpose(1, 2)
        K_new = self.W_K(x).view(B, T, self.n_heads, self.head_dim).transpose(1, 2)
        V_new = self.W_V(x).view(B, T, self.n_heads, self.head_dim).transpose(1, 2)

        # 拼接历史 KV Cache
        if kv_cache is not None:
            K_prev, V_prev = kv_cache
            K = torch.cat([K_prev, K_new], dim=2)  # 沿 seq 维拼接
            V = torch.cat([V_prev, V_new], dim=2)
        else:
            K, V = K_new, V_new

        new_kv_cache = (K, V)  # 更新缓存供下一步用

        # Attention: Q @ K^T / sqrt(d)
        attn = Q @ K.transpose(-2, -1) / math.sqrt(self.head_dim)
        attn = torch.softmax(attn, dim=-1)
        out = attn @ V  # [B, n_heads, T, head_dim]

        out = out.transpose(1, 2).contiguous().view(B, T, D)
        return self.W_O(out), new_kv_cache
```

**③ 关键考点（面试官追问）**：
- "为什么 K/V 能缓存而 Q 不能？" → 见 Day3 Q26（Q 每步新，K/V 历史不变）
- "Prefill 和 Decode 阶段调用有什么区别？" → Prefill 时 kv_cache=None（全量算），Decode 时 T=1 且传入 kv_cache（增量）
- "GQA 怎么改？" → K/V 的 head 数少于 Q（如 Q 32 头，K/V 8 头），repeat_interpolate 后再 attention

**④ 生产经验**：
- **Prefill 优化**：用 Flash Attention（分块计算）
- **Decode 优化**：用 PagedAttention（分页管理 KV Cache 显存）
- **KV Cache 量化**：FP16 → INT8 省一半显存
- 实际生产别手写，用 vLLM/SGLang 已优化好的

**⑤ 示例**：
> "面试手写 KV Cache Attention 的关键：① 维护 kv_cache 张量（K/V 各一份）② 每步新 token 算 Q/K/V，K/V 与缓存拼接 ③ attention 用拼接后的 K/V。常见追问 Prefill vs Decode——Prefill kv_cache=None 全量算，Decode 传 T=1 的 x 和历史 kv_cache 增量算。手写能过基本说明真理解了 Attention，不是只会调 API。生产当然别手写，vLLM 的 PagedAttention 实现复杂得多（分页管理 + COW）。"

---

### Q19. 【生产实战·2026高频】线上 LLM 推理 P99 延迟突然飙高，怎么排查？

**① 原理**：LLM 推理延迟 = 队列 + Prefill + Decode + 网络。P99 飙高通常是某一环节阻塞，需**全链路 trace 分段定位**，不能猜。

**② 分段定位法**：
```
端到端延迟 = 网络入 + 队列等待 + Prefill + Decode + 网络出
```
每段要有监控，P99 飙高时先看哪段炸：
- **队列等待炸** → 容量不足，扩容或限流
- **Prefill 炸** → 长 prompt 集中（ batching 时长 prompt 拖累），用 Chunked Prefill
- **Decode 炸** → batch 太大 / KV Cache 换出（preemption）/ GPU 抢占
- **网络炸** → 跨区域 / 带宽不足

**③ 生产经验排查链**：
1. 看 trace 分段，定位到炸的环节（不是猜）
2. 队列炸 → 看 QPS 是否突增，限流 + 弹性扩容
3. Prefill 炸 → 看 prompt 长度分布，长 prompt 用 Chunked Prefill 或独立 Prefill 实例（PD 分离）
4. Decode 炸 → 看 batch size、preemption 次数、GPU 利用率
5. 排查"邻居效应"：同一 GPU 其他任务抢占

**④ Tradeoff**：
- 限流 vs 体验：P99 飙高时限流降级保 SLA，但部分请求被拒
- 成本 vs 延迟：扩容降延迟但增成本；用 Spot 实例弹性平衡

**⑤ 示例**：
> "我们 P99 从 2s 飙到 15s。trace 分段显示 Prefill 从 200ms 涨到 8s——查 prompt 分布，发现某客户发了大量 30K 长上下文请求，batching 时拖累整批。解法：①Chunked Prefill（长 prompt 切分不阻塞）②PD 分离（长 prompt 走专用 Prefill 实例）③按 prompt 长度分桶 batch（长短不混）。修复后 P99 回到 2.5s。关键认知：P99 排查必须分段 trace，'P99 飙了'是症状不是原因。"

---

## 2026 新增高频题（推理优化前沿）

> 本节补 4 题。PD 分离（Q17）、推理模型部署（Q13）、SGLang vs vLLM Prefix Cache（Q16）已覆盖；这里补 Speculative Decoding 2026 升级、新低比特量化（MXFP4/NVFP4）、Multi-LoRA serving、2026 框架选型更新。

### Q20. 【Speculative Decoding · 2026】EAGLE-3 / Medusa / DeepSeek MTP 有什么区别？框架支持怎样？

> 注：Day3 Q20 已讲 speculative decoding 基础原理；本题补 2026 各变体对比和框架支持。

**① 三大变体原理**：
- **EAGLE-3**：独立的轻量 draft 模型，用 hidden state（隐状态）预测下一 token，最准；vLLM 支持，2.5-4× 加速
- **Medusa**：在 target 模型上加并行头，预测位置 +1/+2/+3，无需独立 draft 模型
- **DeepSeek MTP（Multi-Token Prediction）**：原生多 token 预测，DeepSeek 模型自带

**② 量化效果**：接受率高时 2-3× 加速，无损精度；接受率取决于 draft 与 target 分布相似度

**③ 框架支持对比（2026）**：
- **vLLM 0.4+**：支持最广（EAGLE/EAGLE-2/3、Medusa、n-gram、prompt-lookup）
- **SGLang**：集成 EAGLE-3 中
- **LMDeploy**：支持中
- **P-EAGLE**：并行变体

**④ 生产经验**：
- draft 模型选小但同家族（Llama 70B + 8B draft），分布相似接受率高
- **GPU 不能太闲**：batch 已打满时 speculative 反增成本（draft 也要算）——只在低峰/低延迟场景开

**⑤ 示例**：
> "我们在线 chat 用 EAGLE-3：Llama 70B + 小 draft 模型，K=4，接受率 75%。首字延迟降 40%。关键是只在低并发时段开——高峰 batch 打满时 speculative 反而增 FLOPs。判断：speculative 是'用多余算力换延迟'，适合延迟敏感型在线场景，不适合吞吐型离线。"

---

### Q21. 【低比特量化 · 2026】MXFP4 / NVFP4 是什么？比 GPTQ/AWQ 强在哪？

> 注：Q6 已讲 GPTQ/AWQ/GGUF；本题补 2026 硬件原生 4bit 新格式。

**① 原理**：Blackwell（B200）+ AMD MI355 让硬件原生 4bit 成为现实。MXFP4 和 NVFP4 都是 E2M1 编码（2 位指数+1 位尾数），区别在 scaling 粒度：
- **MXFP4**：block 级 scaling（32 元素/block，约 4096 个值共享一个 scale）
- **NVFP4**：更细的 per-tile scaling，FP8→FP4 精度损失极小

**② 为什么 2026 重要**：
- GPT-OSS 120B 用 MXFP4 MoE 权重量化，能跑在单张 80GB GPU 上
- 硬件原生支持 → 比软件量化的 GPTQ/AWQ 推理更快、精度更好

**③ 生产经验**：
- **逐层敏感度分析**：不是所有层都能 4bit，有的层敏感（量化掉点），用混合精度（敏感层保高 bit）
- KV cache 可量化到 FP8，精度损失极小（详见 Day3 Q32）
- INT2/极低比特（AQLM/BIQA）仍是研究向，sub-4bit 是 2026 实际前沿

**④ Tradeoff**：
- 硬件绑定：MXFP4/NVFP4 要新卡（Blackwell/MI355），老卡用不了
- 精度 vs 成本：4bit 省一半显存，但小模型（7B 以下）量化损失明显

**⑤ 示例**：
> "我们上新 Blackwell 后测 MXFP4：70B 模型权重从 140GB(FP16) 压到 35GB，单卡 80GB 放得下还能留 KV 空间。关键是先做逐层敏感度分析——FFN 层大多能 4bit，attention 的 Q/K/V 保 FP8。混合精度后业务 golden set 几乎不掉。判断：硬件原生 4bit 是 2026 推理降本的主力，但别无脑全量化，敏感层要留高 bit。"

---

### Q22. 【Multi-LoRA Serving】怎么在一个 base 模型上高效服务几百个 LoRA adapter？

**① 原理**：多租户 LLM 服务（每个租户/用户一个 LoRA 微调版本）2026 成主流。核心思路：**一个 base 模型副本 + LRU 缓存的 adapter，按请求切换**——而不是每个 LoRA 部署一份完整模型。

**② 工程方案**：
- **base 共享 + adapter 热加载**：Ray Serve、vLLM、Friendli、TensorRT-LLM 都支持
- **InfiniLoRA**：disaggregated 多 LoRA
- **LoRA-Inlaid**：一次 forward batch 多个 adapter（同 adapter 的请求合批最大化吞吐）

**③ 生产经验**：
- **adapter 缓存**：热门 adapter 常驻显存，冷门按 LRU 淘汰
- **路由挑战**：把用同一 adapter 的请求合批，提升吞吐
- "服务几百个 LoRA 的成本约等于一个 base 模型"（MLSys 2024 参考结论）

**④ Tradeoff**：
- 显存 vs 切换延迟：adapter 全常驻省切换延迟但吃显存；LRU 缓存省显存但冷启动有延迟
- 隔离 vs 共享：base 共享成本最低，但要确保 adapter 间不互相干扰

**⑤ 示例**：
> "我们做 ToB 客服，每个企业客户一个 LoRA（行业话术/知识）。最初每个 LoRA 部署一份 7B，100 个客户要 100 个模型实例，GPU 成本爆炸。改成 Multi-LoRA：一个 Qwen-7B base + 100 个 LoRA adapter（每个几十 MB）LRU 缓存，请求带 tenant_id 路由到对应 adapter。GPU 成本降 95%，热 adapter 切换 <10ms。判断：多租户定制场景，Multi-LoRA 是必选项——base 共享 + adapter 热加载是性价比之王。"

---

### Q23. 【框架选型 · 2026 更新】vLLM 0.22 / SGLang / LMDeploy 2026 怎么选？

> 注：Q1 是老版框架对比；本题补 2026 各框架新进展。

**① 2026 各框架强项**：
- **vLLM 0.22**：PagedAttention + continuous batching 生态最广；新加 KV cache 极致压缩、batch invariance（延迟改善 ~28.9%）、speculative 支持最全
- **SGLang**：RadixAttention（Radix Tree 复用共享前缀 KV），多轮/结构化/前缀共享场景强
- **LMDeploy**（InternLM 出品）：高效，国内生态流行

**② 选型决策**：
- 通用在线推理、生态广 → vLLM
- 多轮对话 / 大量共享 system prompt / 结构化输出 → SGLang（前缀复用收益大）
- 国内团队、InternLM 系模型 → LMDeploy
- 极致延迟（实时语音/多模态）→ TensorRT-LLM（编译重但最快）

**③ 共同趋势**：都支持 PD 分离（via llm-d / Ray Serve）、都集成 speculative decoding、KV 优化成标配

**④ Tradeoff**：vLLM 起步快生态广；SGLang 在前缀共享 workload 领先；选型看你的 prompt 分布（是否大量共享前缀）

🎯 **面试官追问**：
- "vLLM 和 SGLang 怎么选，给个具体场景？" → 客服多轮对话（共享 system prompt）选 SGLang 前缀复用收益大；通用 API 服务选 vLLM 生态
- "为什么 vLLM 不直接抄 RadixAttention？" → 架构差异，vLLM 的 Prefix Cache 是另一套实现；两者在融合

---

### Q24. 【推理成本经济学 · 2026】硬件 + 开源模型 + SLM 三重降本，ROI 怎么重估？

**① 三重降本力量（2026）**：
- **硬件**：Blackwell/Rubin（NVIDIA，宣称推理再降 10×）、Groq LPU（极致低延迟，几百 token/s）、Cerebras（wafer-scale）
- **开源模型**：DeepSeek/Qwen/GLM 接近闭源，自部署边际成本低
- **SLM + 蒸馏**：Phi-4/Qwen-small 端侧近零边际成本

**② 对应用设计的影响**：
- **延迟预算放宽**：硬件快了，可以"多扔点推理"（更长 agent 循环、更多 rerank）
- **ROI 重估**：之前不经济的场景（每用户个性化、always-on agent）变可行
- **价值迁移**：从"成本套利"转向"质量/差异化"

**③ Jevons 悖论（警惕）**：成本降 → 用量涨 → 总花费可能不降反升。要做 GPU FinOps（按租户/团队预算、chargeback、看板）

**④ Tradeoff**：
- 锁定 vs 灵活：Groq/Cerebras 快但锁定；通用 GPU 灵活
- benchmark vs 真实：厂商 benchmark 和真实 workload 吞吐有差距，要自测

**⑤ 示例**：
> "我们 2026 重估了推理 ROI。2024 年'每个用户个性化 agent'因成本不可行；现在 SLM 端侧 + 蒸馏让 80% 流量近零成本，可行了。但我们发现总 GPU 花费没降——因为用量涨了 5 倍（Jevons 悖论）。所以上了 GPU FinOps：按业务线预算 + chargeback + 成本看板，把'推理当资源管理'。判断：成本下降不是'省钱'，是'解锁新场景 + 要更精细治理'。"

---

## 2026 进阶高频题（分水岭级）

> 以下 3 题面向推理基础设施深问，考的是"你是否理解 2026 年大规模 LLM 服务的架构演进"。PD 分离、Multi-LoRA Serving、推理服务高可用，都是大厂 Infra / Tech Lead 面的高频题。

### Q25. 【PD 分离工程化 · 2026 高频】Prefill-Decode 分离什么时候该上？KV 传输层怎么设计？

**① 原理**：Prefill（处理输入 prompt，compute-bound）和 Decode（逐 token 生成，memory-bound）的资源 profile 完全相反——Prefill 要算力、Decode 要显存带宽。单体 vLLM 把两者混跑时，长 prompt 的 Prefill 会阻塞 Decode（反之亦然），导致 GPU 利用率和延迟都差。PD 分离把 Prefill 和 Decode 拆到独立 GPU 池，各自按 profile 扩缩。

**② 什么时候该上 PD 分离**：
- **单体扛不住的信号**：① 高并发 + 长短请求混合（长 prompt 的 Prefill 拖慢短请求的 Decode）；② P99 延迟不稳定（Prefill 尖刺导致 Decode 排队）；③ GPU 利用率低（Prefill 算力闲置时 Decode 显存带宽也不够用）
- **不该上的场景**：低并发 / 请求长度均匀 / 延迟要求宽松——单体 vLLM + Continuous Batching 够用，PD 分离的架构复杂度不值得

**③ 核心工程难点——KV 传输**：
- Prefill 算完后要把 KV Cache 传给 Decode 节点，这是 PD 分离的关键瓶颈
- **KV 传输量巨大**：128K 上下文的 KV Cache 达几十 GB，跨节点传输延迟高
- **方案演进**：
  - Mooncake（Moonshot）：RDMA + KVCache 池化，Prefill 节点写 KV 到共享池，Decode 节点从池读
  - DeepSeek 3FS：分布式文件系统存 KV，利用高速 NVMe 网络
  - Splitwise：同节点 Prefill/Decode 分时复用 + KV 本地传递（不跨节点）

**④ Tradeoff**：
- 延迟 vs 吞吐：PD 分离提吞吐（各池独立优化）但增延迟（KV 传输 + 跨节点调度）；高吞吐低延迟都要时要权衡
- 架构复杂度 vs 收益：PD 分离要管两套 GPU 池 + KV 传输层 + 调度器，运维复杂度翻倍；只有规模够大才值得
- KV 一致性：Prefill 和 Decode 用同一模型但不同副本，模型版本要对齐；KV 传输要保证完整性

**⑤ 示例**：
> "我们 LLM 服务 QPS 从 50 涨到 500 后，单体 vLLM 扛不住——长 prompt（8K+）的 Prefill 占满 GPU 算力，短请求的 Decode 排队 P99 从 2s 飙到 15s。评估后上 PD 分离：① Prefill 池用 A100（算力强）跑长 prompt；② Decode 池用 L40S（显存带宽够、便宜）跑生成；③ KV 传输用 RDMA 共享池（Mooncake 思路），128K KV 传输 ~200ms。效果：P99 从 15s 降到 3s，GPU 利用率从 45% 提到 78%。踩的坑：KV 传输延迟在高并发时会成为新瓶颈（多个 Prefill 同时传 KV 撞带宽），要限流 + 优先级队列。判断：PD 分离是高并发场景的'必经之路'，但 QPS < 100 时单体 vLLM 更省心，别过度工程。"

🎯 **面试官追问**：
- "KV 传输用 RDMA 还是普通网络？" → RDMA（低延迟、高带宽）是首选；普通网络延迟太高（128K KV 传 1-2s）不实用。没有 RDMA 的环境可考虑同节点 PD 分时复用
- "Prefill 和 Decode 的 GPU 配比怎么定？" → 看请求特征：长 prompt 多 → Prefill 池大；生成长（多轮对话）→ Decode 池大。生产中按流量动态调
- "PD 分离后 Continuous Batching 还用吗？" → 用，但各池独立做——Prefill 池做 Prefill batching，Decode 池做 Decode batching，各自优化

⚠️ **踩坑提示**：
- 别说"PD 分离一定更好"——它是高并发场景的优化，低并发时架构复杂度不值得
- 忽略"KV 传输瓶颈"是高频盲点——很多人只想着拆开就好，没想过 KV 怎么传
- "我们上了 PD 分离"如果讲不出"为什么该上"（单体扛不住的信号），会被认为盲目追新

---

### Q26. 【Multi-LoRA Serving · 2026 高频】多租户场景怎么低成本做模型定制？Multi-LoRA 热加载怎么实现？

**① 原理**：企业场景多个租户/业务线要"同一个 base 模型 + 不同定制"（不同语气、不同领域知识、不同输出格式）。全量微调每个租户一个模型 = N 倍显存 + N 倍部署成本。Multi-LoRA Serving 的思路：**base 模型共享一份 + 多个 LoRA adapter 热加载**，按请求动态切换 adapter，显存省 N 倍、定制成本极低。

**② 架构要点**：
- **Base 模型常驻**：一份 base 模型权重常驻显存（最大开销）
- **LoRA adapter 池**：每个 adapter 只有几 MB-几十 MB（rank=8 时约 10-50MB），几十个 adapter 占用远小于一个 base
- **请求路由**：请求带 `lora_id`，推理引擎按 `lora_id` 加载对应 adapter
- **热加载 vs 预加载**：adapter 预加载到显存（快但占显存）vs 按需从磁盘加载（省显存但首次请求慢）。vLLM/Punica 支持混合策略（热点 adapter 预加载 + 冷门按需加载）

**③ 生产经验**：
- **rank 选择**：rank=8-16 覆盖 80% 定制需求（语气/格式/轻量领域适配）；rank=64+ 才用于强领域定制（但 adapter 变大，热加载慢）
- **adapter 版本管理**：adapter 要版本化（v1/v2），灰度切换（先 10% 流量切 v2 验证再全量）；回滚要快（adapter 小，秒级切回）
- **多 adapter 并发**：一个 batch 里可能有不同 `lora_id` 的请求，推理引擎要支持"同一 batch 内多 adapter 计算"（Punica/SGLang 支持，vLLM 2024 后支持）

**④ Tradeoff**：
- 定制深度 vs 灵活性：LoRA 适合轻量定制（格式/语气/轻量领域）；强领域知识/能力提升要全量 SFT，Multi-LoRA 不够
- adapter 数量 vs 加载延迟：adapter 越多显存占用越大 + 热加载延迟越高；几十个以内用预加载，上百个要分层（热点预加载 + 冷门按需）
- 共享 base vs 隔离：共享 base 省资源但租户间可能相互影响（某租户大流量挤占）；大租户可独立 base + 小租户共享

**⑤ 示例**：
> "我们 SaaS 平台 50 个企业租户，每个要定制客服 agent 的语气和领域知识。最初每个租户全量微调一个模型，50 个模型 50 张卡，成本爆炸。改 Multi-LoRA：① base 模型用 Qwen-72B（1 份常驻 2 张 H100）；② 每个租户一个 LoRA adapter（rank=16，约 80MB），50 个 adapter 共 4GB 全预加载到显存；③ 请求带 tenant_id → 路由到对应 adapter。成本从 50 卡降到 2 卡，定制效果用 golden set 验证（语气匹配率 92%，比全量微调的 95% 只差 3 个点，但成本省 96%）。踩的坑：同 batch 多 adapter 计算要用 Punica（vLLM 早期版本不支持），否则只能串行处理不同租户请求。判断：Multi-LoRA 是多租户定制的性价比之王，只要定制需求是'轻量'的。"

🎯 **面试官追问**：
- "LoRA rank 怎么选？" → 从 rank=8 开始，用 golden set 跑准确率，不够再加；rank=64 以上收益递减且 adapter 变大影响热加载
- "adapter 之间会冲突吗？" → 不会，每个 adapter 独立训练、独立加载，请求间无状态污染；但同 batch 多 adapter 计算要引擎支持
- "Multi-LoRA 能替代 RAG 吗？" → 不能。LoRA 定制的是"能力/风格"，RAG 提供的是"实时事实"。两者互补：LoRA 让模型懂领域术语，RAG 给最新知识

⚠️ **踩坑提示**：
- 别把 Multi-LoRA 说成"万能定制"——它只适合轻量定制，强领域知识要全量 SFT
- "我们每个租户微调了一个模型"在大规模租户场景是减分回答——说明没考虑成本
- 忽略"同 batch 多 adapter 计算"是工程盲点——这是 Multi-LoRA Serving 的核心技术难点

---

### Q27. 【推理服务高可用】LLM 推理服务怎么做高可用和容灾？模型供应商挂了怎么办？

**① 原理**：LLM 推理服务的高可用比传统 Web 服务难——① 模型是有状态的（KV Cache、session）；② 故障模式多（GPU OOM、模型崩溃、供应商限流/宕机）；③ 降级复杂（不能简单返回 502，要 fallback 到小模型/缓存/兜底话术）。核心思路：**把 LLM 当不稳定依赖来工程化**，多层兜底。

**② 高可用架构四层**：

| 层 | 策略 | 应对故障 |
|----|------|---------|
| 模型层 | 多供应商 fallback（GPT→Claude→Qwen）、自部署兜底 | 供应商宕机/限流 |
| 推理层 | 多副本 + 负载均衡 + 健康检查 + 自动重启 | GPU OOM、进程崩溃 |
| 缓存层 | 语义缓存兜底（相似 query 命中历史回复） | 推理服务过载时降级返回 |
| 业务层 | 降级话术（"服务繁忙稍后重试"）、异步重试、人工兜底 | 全链路故障 |

**③ 生产经验**：
- **多供应商 fallback 的坑**：不同供应商的模型能力/格式/延迟差异大，fallback 后效果可能掉。要预先做"模型等价性评估"——A 模型 fallback 到 B 模型，B 的输出格式/质量是否可接受
- **语义缓存兜底**：推理服务过载时，用语义缓存（embedding 相似度）命中历史高质量回复，比"返回错误"体验好 100 倍。但要注意误命中风险（相似但不等于的问题）
- **GPU 故障自愈**：GPU OOM / ECC 错误是常态，推理进程要能自动重启（k8s liveness + 自动恢复）；KV Cache 丢失要能从 checkpoint 恢复或优雅降级
- **跨 AZ / 跨 region**：大厂级高可用要跨 AZ 部署推理集群 + 跨 region 流量切换；但 LLM 推理成本高，通常只在"核心场景"做跨 region，非核心单 AZ 容忍

**④ Tradeoff**：
- 成本 vs 可用性：多供应商 + 多副本 + 跨 region = 高可用但成本翻倍；按场景分级——核心场景（客服/交易）高可用，非核心（内部工具）容忍降级
- 一致性 vs 可用性：多副本间 KV Cache / session 不共享，故障切换会丢失对话上下文；要权衡"切走保可用但丢上下文" vs "等恢复保上下文但用户等待"
- fallback 质量 vs 速度：fallback 到小模型质量掉但快；fallback 到大模型质量好但贵。按场景配

**⑤ 示例**：
> "我们 LLM 网关的高可用分四层：① 模型层——主用 GPT-4o，fallback 到 Claude 3.5（供应商限流时自动切），极端情况 fallback 到自部署 Qwen-72B（供应商全挂时兜底）；② 推理层——自部署 Qwen 多副本 + 健康检查，GPU OOM 自动重启 30s 内恢复；③ 缓存层——语义缓存存近 7 天高质量回复，推理过载时 60% 流量命中缓存降级返回；④ 业务层——全挂时返回'服务繁忙'话术 + 异步队列重试。踩的坑：GPT-4o fallback 到 Claude 时输出格式不一致（JSON 结构差异），下游解析失败。修复：网关层做格式适配（统一 JSON schema）+ fallback 模型预先做格式回归测试。教训：LLM 高可用不能只做'切供应商'，还要做'切后质量保证'——格式一致性、能力等价性都要预先验证。"

🎯 **面试官追问**：
- "语义缓存误命中怎么办？" → 设相似度阈值（> 0.95 才命中）+ 关键实体校验（query 里的实体要匹配）+ 低置信度走真推理兜底
- "多供应商的模型等价性怎么评估？" → 用 golden set 在各模型跑一遍，对比准确率/格式一致性/延迟；fallback 模型要"够用"不要求"等价"
- "KV Cache 丢失怎么办？" → 对话场景从历史消息重新 prefill（增加延迟但不丢上下文）；长任务用 checkpoint 定期存 KV

⚠️ **踩坑提示**：
- 别只说"我们做了多供应商 fallback"——要讲 fallback 后的质量保证（格式一致性/能力等价性）
- 忽略"语义缓存兜底"是高频遗漏——这是过载降级体验最好的手段
- "LLM 高可用 = 多副本"是片面的——LLM 的故障模式（GPU OOM、供应商限流）和传统 Web 不一样，要针对性设计

---

## 今日自测清单

> **升级版自查**：不是"能不能背下来"，而是"能不能 3 分钟脱稿讲清楚，并接住追问"。

| 题 | 自查问题 | 达标标准（开口讲） | ✓/✗ |
|----|---------|-------------------|-----|
| Q1 | 五大推理框架怎么选？ | 能讲清"为什么选 vLLM 而非 TensorRT-LLM"的 3 个理由 | ⬜ |
| Q2 | PagedAttention + Continuous Batching | 能区分两者解决的不同问题，不混淆 | ⬜ |
| Q3 | 显存计算 + GPU 选型 | 能现场口算 70B FP8 单卡放不放得下 | ⬜ |
| Q4 | API Gateway 五层 | 能讲清每层的关键决策（不只报菜名） | ⬜ |
| Q5 | LLM 负载均衡 vs Web 服务 | 能讲清"有状态"这个本质差异 + KV-aware routing | ⬜ |
| Q6 | AWQ vs GPTQ | 能讲清原理差异 + 量化后业务评估方法 | ⬜ |
| Q7 | TP/PP/DP 决策 | 能背熟决策框架 + 解释 TP 为何依赖 NVLink | ⬜ |
| Q8 | 成本优化三层 | 能讲清每层具体手段 + 怎么量化效果（成本看板） | ⬜ |
| Q9 | 高可用降级 | 能讲清四级降级 + 与业务方约定降级契约 | ⬜ |
| Q10 | 推理性能指标 | 能讲清 TTFT/TPS 在不同业务的权重 + 怎么 benchmark | ⬜ |
| Q21 | MXFP4/NVFP4 新量化 | 能讲清"硬件原生 4bit + 逐层敏感度" | ⬜ |
| Q22 | Multi-LoRA Serving | 能讲清"base 共享 + adapter 热加载" | ⬜ |
| Q24 | 推理成本经济学 2026 | 能讲清三重降本 + Jevons 悖论 | ⬜ |

**达标线**：13 题中 ≥ 10 题能脱稿讲 + 接住至少 1 个追问 → Day9 通过。
**未达标处理**：薄弱题回到主文件附录 A.3 重读原理，再用本文件示例答练嘴 3 遍。

## 参考资源

- [vLLM 官方文档](https://docs.vllm.ai/)
- [TensorRT-LLM GitHub](https://github.com/NVIDIA/TensorRT-LLM)
- [SGLang RadixAttention](https://github.com/sgl-project/sglang)
- [LLM Load Balancing - Portkey](https://portkey.ai/blog/llm-load-balancing)
