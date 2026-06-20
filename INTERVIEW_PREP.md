# 面试备战手册：电商 AI Agent 架构师 / 核心开发师

> 基于 eCan.ai 项目（v0.8.24）定制，包含自我介绍、STAR 故事、高频技术题、系统设计题、反问问题、差异化表达、弱点应对、代码讲解八大模块。

---

## 目录

1. [3 分钟自我介绍脚本](#1-3-分钟自我介绍脚本)
2. [STAR 故事库](#2-star-故事库)
3. [10 道高频技术题 + 参考答案](#3-10-道高频技术题--参考答案)
4. [3 道系统设计题 + 架构方案](#4-3-道系统设计题--架构方案)
5. [5 个高质量反问问题](#5-5-个高质量反问问题)
6. [技术亮点差异化表达](#6-技术亮点差异化表达)
7. [潜在弱点及应对策略](#7-潜在弱点及应对策略)
8. [代码 Walk-through 准备](#8-代码-walk-through-准备)
9. [面试前自测清单](#9-面试前自测清单)
10. [关键代码路径速查](#10-关键代码路径速查)

---

## 1. 3 分钟自我介绍脚本

> **使用方式**：记住以下三个结构层次和关键词，口语化表达，**不要逐字背诵**。语速约 130 字/分钟。

---

### 开场：定位自己（20 秒）

"我在 AI Agent 工程领域深耕了 X 年，主要聚焦多智能体系统架构、LLM 工程和电商自动化三个方向。"

---

### 主体：项目三个核心贡献（2 分钟）

"过去这段时间，我作为核心开发者参与了 eCan.ai 的从零到 v0.8 版本的构建。这是一个 AI 原生、隐私优先的跨平台电商智能体平台，已覆盖 RPA 自动化、客服、销售、采购、营销等 9 种垂直场景，支持 Windows/macOS/Linux 桌面端和 Docker Web 部署。

我的核心贡献集中在三个层面：

**第一，智能体内核。** 我们没有简单套壳 LangChain，而是在 LangGraph StateGraph 之上构建了自研的 `node_builder()` 节点包装层。它作为高阶函数，为每个计算节点透明地封装了指数退避重试、断点单步调试、节点间状态转移 DSL、8 阶段性能打点和终止状态检测（blocked/timeout/failed/completed）——前端的 Flowgram 可视化编辑器可以实时高亮每个节点的执行状态。

**第二，混合云执行架构。** 我们设计了三层执行模式——`cloud_only`、`local_reactive`、`local_extract`：云端 LLM 做决策，通过 AWS AppSync 把命令下发给本地浏览器执行；`PrivacyAgent` 在截图上传前做 PII 脱敏（正则替换手机/邮箱/信用卡为合成数据），保证用户数据原则上不出设备边界。

**第三，A2A 多智能体通信。** 我们实现了 LAN/WAN 双路由的统一消息层——局域网内走 Zeroconf mDNS 直连，跨网络走 AppSync WebSocket 中继，`unified_send_chat_message()` 自动判断路由，并用异步 fire-and-forget 解决了 A2A 互发的死锁问题。"

---

### 收尾：为什么在这里（20 秒）

"这些不是 POC，是跑在真实用户设备上的生产代码。我带着这些实战经验，想在贵司把多智能体技术深度应用到电商核心业务链路上。"

---

## 2. STAR 故事库

准备以下 3 个故事，根据面试官问题选用最匹配的一个，每个控制在 3 分钟内。

---

### 故事 A：`node_builder()` 的诞生——从"节点失败就宕机"到"可观测的健壮执行"

**Situation（背景）**

早期 LangGraph 节点直接调用用户代码，某节点抛异常后整个工作流就崩溃，前端一片空白无法 debug；重试也没有退避策略，高并发时会把 LLM API 打挂。

**Task（任务）**

设计一个统一的节点包装层，在不侵入用户业务逻辑的前提下，加上重试、可观测性、断点调试和状态机管理。

**Action（行动）**

在 `agent/ec_skill.py:430` 实现了 `node_builder()` 高阶函数。它接收原始节点函数，返回一个 LangGraph 兼容的 wrapper，核心机制有五个：

- **重试**：指数退避 `base_delay * 2^attempts + jitter`，遇到 `[NON_RETRYABLE]` 标签立即停止——这是给业务代码的 escape hatch。
- **断点**：`bp_manager.has_breakpoint(node_name)` 命中时调用 LangGraph 的 `interrupt()`，把经 `_safe_state_view()` 安全化的 state 快照发给前端；`skip_bp_once` 机制防止 resume 后重复命中同一断点。
- **单步**：`step_once` 标志实现"执行完当前节点后，下个节点自动 interrupt"，即 IDE 里的 Step Over 语义。
- **终止态检测**：扫描 `tool_result[node_name].status`，识别 blocked/timeout/failed/completed，桌面模式走 IPC，云模式走 cloud_logger，前端实时更新节点颜色。
- **性能打点**：`__node_timings__` 写入 state，全链路延迟可追溯。

**Result（结果）**

节点可靠性从"随机崩溃"变为可恢复，Flowgram 编辑器实现了 IDE 级别的实时调试体验，线上 debug 效率提升约 70%。

---

### 故事 B：MCP 并发序列化瓶颈——生产环境 `rag_query` 超时

**Situation**

在多客户并发压测时，3 个并发的 `rag_query` 调用中有 2 个会超时（60s 全部耗尽），然后诡异地在 ~2s 内完成。

**Task**

定位根因，修复高频并发 MCP 工具的性能问题。

**Action**

通过分析日志发现，持久化 streamable-HTTP `ClientSession` 会序列化所有 in-flight 请求——一个慢请求会阻塞后面所有请求。解决方案分两步：

1. 在 `agent/mcp/local_client.py:72` 引入 `_NO_PERSISTENT_SESSION = {"send_chat", "rag_query"}` 黑名单——这些高频并发工具改走独立临时 Session（ephemeral session），每次调用独立建连，彼此不阻塞；
2. 增加 8 阶段 PERF 打点：`streams_open_ms / session_ctx_ms / initialize_ms / call_tool_ms / post_sleep_ms / session_close_ms / streams_close_ms / total_ms`，精确定位每一阶段耗时，输出到 INFO 日志。

**Result**

并发 `rag_query` 的 P99 从 60s 超时降至 ~2.5s。低频工具继续用 persistent session 以降低握手开销。

---

### 故事 C：A2A 死锁——智能体互相等待

**Situation**

智能体 A 发消息给 B 并阻塞等待响应，B 处理完想把结果发回 A，但 A 此时被自己的同步等待阻塞，B 的回调永远得不到处理——经典互等死锁。

**Task**

在不改变现有同步调用接口的前提下，解决 A2A 互发死锁。

**Action**

在 `agent/ec_agent.py:609` 采用**非对称设计**：

- A→B 用 `a2a_send_chat_message_sync()`（需要确认 B 已收到任务，同步等待合理）；
- B→A 用 `a2a_send_chat_message_async()`（fire-and-forget，提交给专用 4 线程 `_a2a_send_executor`，立即返回）。

B 的主线程提交异步回复后立即继续处理下一条消息，不再等 A 确认。A 的接收端通过独立的 `a2a_msg_queue` 消费 B 的回复，两者解耦。

**Result**

死锁完全消除，A2A 双向通信延迟降至毫秒级。已知 limitation：fire-and-forget 不确认 A 是否成功接收，目前靠 AppSync 持久化兜底——这是诚实的工程权衡，面试时主动说出能展示技术成熟度。

---

## 3. 10 道高频技术题 + 参考答案

---

### Q1：你们的多智能体架构是怎么设计的？智能体之间如何协调？

**核心答案：**

三层层级结构：**Commander**（指挥官，全局调度 + 任务分解）→ **Platoon**（排级，领域内协调）→ **Operator**（操作员，具体执行——如 RPA Supervisor → RPA Operator）。每个智能体是 `EC_Agent` 实例，继承自 browser-use 的 `Agent`，内部维护 `skills`（能力列表）和 `tasks`（运行中任务列表）。

通信层面构建了 `UnifiedMessenger`：
- 局域网内：Zeroconf/mDNS 发现 `AgentEndpoint`（含 `lan_host`、`lan_port`），走直接 HTTP A2A；
- 跨网络：走 AWS AppSync WebSocket 中继（`wan_relay_channel`）；
- `AgentEndpoint.is_lan_reachable` 属性检查最近 180s 内是否有心跳，自动选路。

核心工程问题——死锁：用**非对称模式**解决，发起方同步等待 B 接收确认，回复方用异步 fire-and-forget，互不阻塞。

---

### Q2：LangGraph 在你们系统里承担什么角色？做了哪些扩展？

**核心答案：**

LangGraph 是技能工作流的**执行引擎**，每个 `EC_Skill` 持有一个 `CompiledStateGraph`（`runnable`）。

四大扩展方向：
1. **`node_builder()` 包装层**：LangGraph 原生不提供跨节点统一重试、断点、可观测性，用高阶函数封装，保持 API 兼容（节点函数签名不变）；
2. **`NodeState` 扩展**：比标准 `MessagesState` 增加了 `events`、`attributes`、`tool_result`、`breakpoint`、`condition_vars`、`metadata` 等 30+ 字段，适配电商业务状态；
3. **Flowgram→LangGraph 编译器**：可视化流程图（12 种节点类型）运行时编译成 StateGraph，用户不需要写代码；
4. **`mapping_rules` DSL**：声明式地描述事件到状态的投影规则，解耦数据绑定与业务逻辑。

---

### Q3：描述你们的 RAG 架构，如何处理多租户和不同文档类型？

**核心答案：**

使用 **LightRAG** 作为 RAG 引擎，支持**向量检索 + 知识图谱**双引擎：向量检索适合语义相似查询，知识图谱适合关系推理（如"哪些产品属于同一品牌"）。

**多租户隔离**：HTTP 请求头 `LIGHTRAG-WORKSPACE` 传递工作区标识，值使用 `urllib.parse.quote()` percent-encoding，支持中文工作区名称，服务端根据此头切换到对应目录。

**文档分块路由**（`UniversalTableChunker`）：优先检测 Markdown 表格 → Excel → CSV/TSV → JSON 表格 → 降级为纯文本分块，不同格式走不同切分策略。

**质量保障**：`provider_limits_validator` 做速率限制熔断，`rerank_score_normalizer` 做归一化重排序，`lightrag_confidence_scorer` 给每条结果打置信度分。

---

### Q4：Flowgram 是什么？是如何编译成 LangGraph 的？

**核心答案：**

Flowgram 是平台自研的**可视化工作流表示格式**，用 JSON 描述节点和有向边，类似 n8n/Dify 的 Flow 格式，但面向 Agent 任务场景。

编译器 `flowgram2langgraph.py` 的 `function_registry` 映射 **12 种节点类型**到对应 Python builder：

| 节点类型 | Builder 函数 |
|---|---|
| `llm` | `build_llm_node`（LLM 调用） |
| `basic` / `code` | `build_basic_node`（用户自定义 Python） |
| `mcp` | `build_mcp_tool_calling_node`（调用 MCP 工具） |
| `condition` | `build_condition_node`（条件分支） |
| `loop` | `build_loop_node`（迭代） |
| `browser-automation` | `build_browser_automation_node` |
| `pend_event_node` | `build_pend_event_node`（等待外部事件） |
| `chat_node` | `build_chat_node` |
| `rag_node` | `build_rag_node` |
| `task` | `build_task_node`（嵌套任务） |
| `tool-picker` | `build_tool_picker_node` |
| 结构节点 | `event`/`comment`/`variable`/`sheet-call` |

**编译流程**：解析 JSON → 实例化 builder → `node_builder()` 包装 → 加入 `StateGraph` → 按边关系添加条件/无条件跳转 → `graph.compile(checkpointer=InMemorySaver())`。

---

### Q5：`mapping_rules` DSL 解决了什么问题？设计上有什么取舍？

**核心答案：**

**问题**：同一份"用户填写的表单"，在不同事件来源（HTTP 推送 / GUI 事件 / A2A 消息）中字段路径不同，但最终需要写入 LangGraph state 的同一位置。如果每个节点各自 hardcode `if event_type == ...: state["x"] = event["data"]["y"]`，代码量爆炸且难以维护。

**DSL 结构**：
```json
{
    "from": ["event.data.qa_form_to_agent", "event.data.qa_form"],
    "to": [
        {"target": "state.attributes.forms.qa_form"},
        {"target": "resume.qa_form_to_agent"}
    ],
    "transform": "to_string",
    "on_conflict": "merge_deep"
}
```
- `from`：多源字段路径，取第一个非空值；
- `to`：多目标，可同时写 state 和 resume；
- `on_conflict`：`merge_deep`（递归合并）/ `merge_shallow` / `overwrite` / `skip` / `append`。

**取舍**：维护 `developing` 和 `released` 两套模式：开发模式 `strict: false`，mapping 失败静默降级方便调试；发布模式 `strict: true`，保证生产数据完整性。`_write()` 函数统一实现所有 conflict 策略，保护嵌套 dict 不被整体覆盖。

---

### Q6：你们的混合云架构如何保证用户隐私？云端和本地如何分工？

**核心答案：**

核心设计原则：**"云决策、本地执行"**。`EXECUTION_TIER` 分三种：

| 模式 | LLM 位置 | 浏览器操作 | 适用场景 |
|---|---|---|---|
| `cloud_only` | 云端 | 云端 → AppSync → 本地执行 | 普通 SaaS 部署 |
| `local_reactive` | 本地 | 本地直接 | 隐私优先/本地 Ollama |
| `local_extract` | 本地 | 本地，仅提取数据 | 数据采集场景 |

**隐私保障机制**：
- `PrivacyAgent` 拦截浏览器截图，`RegexMaskFilter` 用正则替换 PII（手机/邮箱/信用卡 → 合成数据），LLM 全程看到的是脱敏数据；
- `hook_loader` 对第三方扩展做 Ed25519 签名验证（`hook_signing`），防止恶意 hook 窃取数据；
- `appsync_passive_transport` 实现云→地命令下发 + 结果回传，真实用户数据留在本地执行层。

---

### Q7：IPC 架构是怎么设计的？为什么需要白名单机制？

**核心答案：**

`IPCHandlerRegistry`（`gui/ipc/registry.py`）是统一的处理器注册和中间件系统。每个 IPC 调用经过三层中间件：

```
① 白名单检查 → ② JWT Token 校验 → ③ system-ready 检查 → ④ 业务处理器
```

**白名单的原因**：用户登录前或系统初始化阶段，前端需要调用 `login`、`get_system_status`、`get_initialization_progress` 等接口，此时 Token 不存在、MainWindow 未创建，必须跳过 ②③ 两层校验。白名单是静态集合（20+ 方法名）。

**性能优化**：system-ready 状态缓存动态 TTL——就绪时缓存 30s，非就绪时缓存 5s，避免每次调用都查询 `AppContext.get_main_window()`。

**桌面/Web 双态适配**：`ECAN_MODE == "web"` 时直接返回 ready，不依赖 Qt MainWindow，同一套处理器适配两种部署模式。

---

### Q8：`EC_Skill` 的 stable ID 生成策略是怎么设计的？为什么 code skill 用 UUID5？

**核心答案：**

`_generate_stable_id()` 区分两种来源：

| 来源 | ID 生成方式 | 原因 |
|---|---|---|
| `source == "code"` | UUID5（DNS 命名空间 + `"code:{name}"`） | 确定性生成，相同名字→相同 ID |
| `source == "ui"` | 随机 UUID4 | 每个用户创建的技能独一无二 |

**为什么 code skill 要确定性 ID**：代码内置技能（如 `ec_customer_support`）在不同机器、不同启动时刻多次实例化，必须拥有相同的 ID，这样数据库引用、技能编辑器关联、技能市场标识才不会错乱。

`@model_validator(mode='after')` 的 `_ensure_stable_id` 只在 ID 未设置时才生成，update 操作不改变 ID。`cloud_id` 字段专门存云端同步 UUID，与本地 ID 解耦，两者可以独立演化。

---

### Q9：描述事件驱动架构的统一信封格式和优先级实现。

**核心答案：**

所有消息经 `normalize_event()` 统一包装为信封：
```json
{
    "type": "human_chat | a2a | timer | browser_event | passive_command | webhook | websocket",
    "context": {
        "chatId", "sessionId", "task_id", "run_id", "timer_id", "senderId", ...
    },
    "data": { /* 类型特定载荷 */ }
}
```

**优先级队列** `_dequeue_with_priority()`，三级：
- **HIGH**：`human_chat / a2a`（直接客户交互，最高优先）
- **NORMAL**：其他事件
- **LOW**：`browser_event`（DOM 快照，可能大量涌入）

**防积压机制**：
- `browser_event` 同 label 去重合并（`_coalesce_queued_browser_events()`）——相同快照只保留最新一条；
- `_STALE_TTL_SEC` 内未处理的 chat/a2a 消息丢弃，防止消息堆积导致"过时消息延迟执行"。

---

### Q10：Skill Editor 的在线断点调试是怎么串联实现的？

**核心答案：**

四个机制协作：

1. **`BreakpointManager`**（`dev_defs.py`）：维护有断点的节点 ID 集合，前端编辑器点击节点注册/取消断点；

2. **`node_builder()` 检查**：每次节点执行前调用 `bp_manager.has_breakpoint(node_name)`，命中则调用 LangGraph 的 `interrupt()`，传入 `_safe_state_view()` 序列化的 state 快照（去循环引用、JSON 不安全类型）；

3. **`skip_bp_once` 机制**：前端点 Resume 时，TaskRunner 把当前节点 ID 加入 skip list，本次 resume 不再触发同一断点，避免死循环；

4. **`step_once` 标志**：前端点 Step Over 时设置此标志，`node_builder` 在当前节点执行完后，下一个节点进入时自动 `interrupt()`，实现 IDE 级别的单步执行。

整套机制使用户在浏览器里对 Agent 工作流做到真正的交互式调试。

---

## 4. 3 道系统设计题 + 架构方案

---

### 设计题一：多智能体电商客服系统

**题目**：为大型电商设计多智能体客服系统，支持 100 万+ SKU，日均 50 万对话，P99 响应 < 3s，支持退换货、物流查询、商品推荐等场景。

**架构图**：

```
用户消息（多渠道）
       │
       ▼
[API Gateway / WebSocket 长连接]
       │
       ▼
[Router Agent]  ←── 轻量 LLM，<100ms 意图识别
       │
       ├─── 退换货 ──► [RefundAgent]   ──► OMS MCP（订单查询/退款申请）
       ├─── 物流   ──► [LogisticsAgent] ──► WMS MCP（物流状态）
       └─── 推荐   ──► [RecommendAgent] ──► LightRAG（商品知识库，向量+图谱）
                                │
                         [QualityAgent]（实时抽样质检）
                                │
                         [EscalationAgent]（人工升级路由）
```

**关键设计决策**：

| 决策点 | 方案 | 理由 |
|---|---|---|
| 会话隔离 | `chatId` + `LIGHTRAG-WORKSPACE = user_id` | 用户数据不串 |
| MCP 并发 | 高频 OMS 查询走 ephemeral session | 避免序列化瓶颈 |
| 异步确认 | `pend_event_node` 等待用户确认退款 | Human-in-the-loop |
| P99 达标 | Router 小模型 + RAG 并发 + LangGraph parallel edges | 三路并行压时延 |
| 降级 | LLM 超时走规则引擎；专家 Agent 失败走人工+context 移交 | 保证服务可用性 |
| PII 保护 | 敏感字段（地址/支付）不进 LLM context，走工具调用获取 | 合规要求 |

---

### 设计题二：跨平台 RPA 任务调度系统

**题目**：设计支持 1000 台设备、跨 Windows/Mac/Linux 的 RPA 任务调度系统，任务需要浏览器自动化，支持失败重试、任务依赖、实时监控。

**架构图**：

```
[调度控制面]
CommanderAgent（全局调度，单实例）
       │
       │  unified_send_chat_message（自动 LAN/WAN 选路）
       ▼
[设备注册层]
  Zeroconf mDNS（局域网自动发现）
  AppSync WAN（跨网络设备注册）
  AgentEndpoint.is_lan_reachable（180s 心跳检测）
       │
       ▼
[每台设备]
  OperatorAgent（EC_Agent + browser-use）
  PassiveAgent（等待云端 AppSync 命令）
  PrivacyAgent（截图脱敏后上报）
  本地 MCP Server（截图/点击/输入/OCR 工具）
```

**关键设计决策**：

| 决策点 | 方案 |
|---|---|
| 任务依赖 | LangGraph DAG 条件边（天然支持有向无环图） |
| 失败重试 | `node_builder` 默认 3 次指数退避重试，可按任务覆盖 |
| 实时监控 | `_notify_node_status()` 推送 TaskProgressBus，前端 WebSocket 订阅 |
| 设备离线 | Commander 从 `AgentDirectory` 找备用设备重新路由 |
| 数据安全 | `PrivacyAgent` PII 脱敏，`hook_signing` 防恶意插件 |

---

### 设计题三：AI 采购 Agent（询价→比价→审批→下单）

**题目**：为 B2B 电商设计 AI 采购 Agent，整合多家供应商价格、库存，结合历史采购数据做智能推荐，并自动完成询价→比价→下单流程。

**LangGraph 工作流**：

```
[需求理解节点 - LLM]
  "我要买1000个螺丝" → 规格/数量/预算/时限
       │
       ▼
[供应商搜索节点 - MCP]
  search_suppliers（调 ERP/电商 API）
       │
       ▼
[并行询价 - Parallel LangGraph 分支]
  ├── 供应商 A 报价（MCP）
  ├── 供应商 B 报价（MCP）
  └── 供应商 C 报价（MCP）
       │
       ▼
[比价+推荐节点 - LLM + RAG]
  LightRAG 图谱：历史价格趋势（商品-供应商-价格-时间 关系）
  向量检索：供应商信用评级
  LLM 综合评分 → 推荐报告
       │
       ▼
[审批节点 - pend_event_node]
  等待采购经理确认（QA Form 表单）
  mapping_rules: event.data.qa_form → state.forms.approval
       │
       ▼
[下单节点 - MCP 或 browser-automation]
  调用供应商 API 或 RPA 完成下单
```

**技术亮点**：历史采购数据沉淀在 LightRAG 知识图谱，新询价时能做价格趋势分析；`TimerService` 触发价格波动时自动重新比价；`pend_event_node` 实现非阻塞的 Human-in-the-loop 审批。

---

## 5. 5 个高质量反问问题

根据面试进展选 2~3 个提问，展示深度思考。

---

**问题一（技术战略）**

> "贵司的 Agent 目前偏向 Orchestrator 模式（中央调度+分工执行）还是去中心化的 Swarm 模式？在电商高并发场景下，您们对两者的权衡是怎么看的？eCan.ai 我们选择了带监督层的层级模型，但 Swarm 在某些场景下弹性更好，很想了解贵司的实践。"

**问题二（可观测性）**

> "多智能体系统最难的工程问题之一是可观测性——当一条链路跨 5 个 Agent、10 个 LangGraph 节点时，怎么定位是哪一步出了问题？贵司现在有没有统一的 tracing 方案（比如 LangSmith 或自研），还是说这个问题还在解决中？"

**问题三（LLM 选型）**

> "国内电商 Agent 在 LLM 选型上面临延迟、合规、费用三重约束。贵司目前是单一 provider 还是有动态路由（简单意图用小模型，复杂推理用大模型）？这对理解工具调用层的设计很重要。"

**问题四（数据飞轮）**

> "Agent 产生的操作日志、用户反馈、成功/失败轨迹，在贵司有没有形成数据飞轮，用来改进提示词甚至做 SFT？我们在 eCan.ai 里通过 Skill Editor 的操作记录积累了素材，但还没有完整的闭环，很想了解贵司的探索。"

**问题五（团队协作）**

> "电商 AI Agent 项目通常需要 AI 工程师、后端、前端和业务方紧密配合。贵司的协作模式是 AI 团队提供平台能力+业务团队自助配置，还是 AI 工程师嵌入业务线？这对判断岗位重点职责很有帮助。"

---

## 6. 技术亮点差异化表达

用于回答"你和其他候选人有什么不同"或"项目最大的技术挑战是什么"。

| 竞争维度 | 普通候选人的水平 | 本项目的实际水平 |
|---|---|---|
| **技术深度** | "用 LangChain 搭了个 RAG，调了 OpenAI API" | 完整 Agent 执行引擎：Flowgram 编译器 + node_builder 运行时 + 可观测性 |
| **工程问题** | 能讲原理，无具体数据 | MCP 超时 60s→2.5s，有根因分析 + 数据 + 修复路径 |
| **系统思维** | 单节点/单技能实现 | A2A 死锁系统性解法；mapping_rules DSL 解耦数据绑定 |
| **安全意识** | "以后加安全层" | PrivacyAgent / hook_signing / 白名单鉴权从第一天就在架构里 |
| **业务理解** | 纯技术框架，不懂业务 | 9 种垂直场景（客服/采购/RPA）的实际业务逻辑都亲自参与 |
| **多模式部署** | 只会 API 调用 | 桌面（PyInstaller）/ Web（Docker+Uvicorn）/ 云工作节点（ECS Fargate+Xvfb）三种部署模式 |

**一句话总结**（可在自我介绍末尾或回答"优势"时使用）：

> "我的优势是端到端——不只是调 API 搭积木，而是真正参与了 Agent 执行引擎的设计和实现，碰过真实的生产性能问题，并且有完整的修复路径和数据。"

---

## 7. 潜在弱点及应对策略

提前准备，被问到时不慌乱。

---

### 弱点一：没有超大规模生产运营经验（10 万+ Agent 并发）

**应对话术**：

> "我们的系统目前覆盖中小型部署规模，这点我坦诚说。但 eCan.ai 的架构在设计时考虑了水平扩展路径：MCP Server 是无状态 HTTP 服务，可以水平扩；`AgentDirectory` 目前是进程内单例，大规模场景下可以替换为 Redis 或分布式服务发现（Consul/etcd）。我对微软 AutoGen 的分布式模式、LangGraph Platform 的多副本部署也有了解，但确实缺乏生产验证机会，这也是我希望在贵司获得的。"

---

### 弱点二：LLM 成本控制经验不足

**应对话术**：

> "我们在 eCan.ai 里通过 token_tracker 做了调用量统计，但没有做到精细的 prompt 缓存和成本优化。我理解 KV-cache、prefix caching、prompt compression 的原理，也研究过 Anthropic 和 OpenAI 的 batch API 如何降低成本。在贵司的规模下，我会把这个作为第一优先级补全的技能项。"

---

### 弱点三：被质疑技术"原创性"（是否只是套用开源框架）

**应对话术**：

> "`node_builder()` 的断点管理、state DSL、A2A 死锁解法这些在 LangChain 或 AutoGen 里找不到等价物。browser-use 是我们的浏览器操作基础库，但我们在它上面加了 `PrivacyAgent` 拦截层、`EXECUTION_TIER` 分级执行、`passive_agent` 云地协作架构——这些是我们从零设计的。如果您感兴趣，我可以现场 walk-through 其中任意一个的代码实现。"

---

### 弱点四：测试覆盖率可能被质疑

**应对话术**：

> "AI Agent 系统的测试挑战是 LLM 输出的不确定性，这是行业共性问题。我们的确定性模块（mapping_rules、节点状态机、路由逻辑）有单元测试覆盖；整体流程靠 Skill Editor 的实时调试和 test_product_listing_orchestrator_skill.py 这类 E2E 测试验证。我对如何设计 deterministic mocking、golden test set、LLM-as-judge 评测有思考，在有机会的情况下，愿意推动建立更完善的测试体系。"

---

## 8. 代码 Walk-through 准备

面试官要求"给我看看代码"时，主动引导到以下三段。**每段能在 3 分钟内讲清楚**。

---

### 代码段一：`node_builder()`——最值得讲，展示系统级思维

**位置**：`agent/ec_skill.py:430`

**讲解脚本（逐步引导）**：

```
"我给你展示我们项目里我最引以为傲的函数——node_builder()。"

[打开文件，定位到 430 行]

"它是一个高阶函数，接收原始节点函数，返回一个 LangGraph 兼容的 wrapper。
用户的业务节点函数完全不知道这些机制存在——横切关注点和业务逻辑完全解耦。"

[指向 step_once 相关代码]
"第一件事：检查 step_once 标志。如果处于单步调试模式，当前节点执行完后，
下个节点进入时自动 interrupt，这就是 IDE 里的 Step Over 语义。"

[指向断点检查代码]
"第二步：bp_manager.has_breakpoint(node_name)。命中则调用 LangGraph 的 interrupt()，
传入 _safe_state_view() 安全化的 state 快照给前端。skip_bp_once 机制防止 resume 
后重复命中同一断点。"

[指向重试逻辑]
"第三步：执行节点函数，重试是指数退避 base_delay * 2^attempts + jitter，
如果异常文本包含 [NON_RETRYABLE] 就立即停止——这是给业务代码的 escape hatch。"

[指向终止态检测]
"第四步：扫描 tool_result[node_name].status，识别四种终止态，
桌面走 IPC，云端走 cloud_logger，前端实时更新节点颜色。"
```

---

### 代码段二：`_NO_PERSISTENT_SESSION`——展示工程排查能力

**位置**：`agent/mcp/local_client.py:72`

**讲解脚本**：

```
"这一行代码背后有一段完整的排查过程。"

[指向第 72 行]
"_NO_PERSISTENT_SESSION = {'send_chat', 'rag_query'} ——这两个工具不用持久化 Session。"

"为什么？重现 bug：3 个并发 rag_query，2 个 60s 超时，然后在 2s 内完成。
这个 pattern 非常典型——序列化。"

"streamable-HTTP ClientSession 底层是 SSE 长连接，不是多路复用的，
多个并发 call_tool() 必须排队。第一个 rag_query 耗时 2s，
第二个等 60s 超时后自己用 2s 完成。"

"解决：热路径工具走 ephemeral session（建连→握手→调用→断开）。
低频工具继续用 persistent session 节省握手开销。"

"为了验证，我加了 8 阶段 PERF 打点——streams_open_ms / initialize_ms / 
call_tool_ms 等，下次性能问题可以精确定位在哪一阶段。"
```

---

### 代码段三：`a2a_send_chat_message_async()`——展示并发设计思维

**位置**：`agent/ec_agent.py:609`

**讲解脚本**：

```
"这段代码解决的是经典的 A2A 死锁问题。"

[画图：A ← → B 的双向通信]
"Agent A 发消息给 B，用同步等待——A 的线程阻塞。
B 处理完想回复 A——如果 B 也用同步，B 的线程阻塞等 A 确认，
但 A 的线程已经被自己占住了，无法处理 B 的消息。互等死锁。"

[指向 async 方法]
"解决方案是非对称设计：
- A→B 用 sync（A 需要确认 B 已收到任务，等一下合理）；
- B→A 用 async fire-and-forget，提交给专用 4 线程 _a2a_send_executor，
  立即返回，B 的主线程继续处理下一条消息。"

"这里我主动说一个已知 limitation：fire-and-forget 意味着我们不确认 A 
是否成功接收了 B 的回复。目前靠 AppSync 持久化兜底。
这是工程权衡——完美方案需要回复确认+重传机制，我们优先解决死锁，
可靠性靠基础设施兜底。"
```

---

## 9. 面试前自测清单

面试前一天，用以下清单自测：

- [ ] 3 分钟自我介绍能流畅说完，不看稿，控制在 3 分钟内
- [ ] 3 个 STAR 故事能各自在 3 分钟内讲完，S/T/A/R 四段清晰
- [ ] 能在白板/纸上徒手画出客服系统架构图（不超过 8 个组件）
- [ ] `node_builder()` 的 5 个核心机制能不看代码说出（重试/断点/单步/终止态/打点）
- [ ] MCP 超时问题的根因（Session 序列化）和解法（ephemeral session）能 1 分钟表述清楚
- [ ] A2A 死锁和非对称解法能画图解释清楚
- [ ] 5 个反问问题背熟，根据面试进展选 2~3 个

---

## 10. 关键代码路径速查

面试前务必熟读（按优先级排序）：

| 优先级 | 文件路径 | 核心内容 |
|---|---|---|
| ⭐⭐⭐ | `agent/ec_skill.py:430–1092` | `node_builder()` 完整实现（600+ 行） |
| ⭐⭐⭐ | `agent/mcp/local_client.py:50–171` | MCP Session 策略 + 8 阶段性能打点 |
| ⭐⭐⭐ | `agent/ec_agent.py:316–363, 609` | A2A 服务初始化 + 异步发送方法 |
| ⭐⭐ | `agent/a2a/discovery/directory.py` | `AgentEndpoint` 结构 + LAN/WAN 选路逻辑 |
| ⭐⭐ | `agent/ec_skills/flowgram2langgraph.py` | 编译器入口 + `function_registry` |
| ⭐⭐ | `gui/ipc/registry.py` | 中间件栈 + 白名单 + 双态适配 |
| ⭐⭐ | `agent/ec_skill.py:48–154` | `mapping_rules` DEFAULT_MAPPING_RULE 定义 |
| ⭐ | `docs/mapping-dsl.md`（前 100 行） | DSL 规范总览 |
| ⭐ | `agent/ec_skills/browser_use_extension/privacy_agent.py` | PII 脱敏实现 |
| ⭐ | `agent/ec_agents/`（浏览各文件名） | 9 种预置智能体模板，了解业务覆盖 |

---

*本文档基于 eCan.ai v0.7.0 代码库生成，结合电商 AI Agent 架构师岗位的面试特点定制。*
