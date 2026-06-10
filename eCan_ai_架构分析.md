# eCan.ai 深度架构分析报告

> 生成日期：2026-06-10  
> 仓库：https://github.com/scszcoder/eCan.ai  
> 版本：0.8.24

---

## 目录

1. [项目概述](#1-项目概述)
2. [整体架构](#2-整体架构)
3. [技术栈](#3-技术栈)
4. [模块划分](#4-模块划分)
5. [数据库设计](#5-数据库设计)
6. [核心业务流程](#6-核心业务流程)
7. [关键类与函数关系](#7-关键类与函数关系)
8. [API 与通信层](#8-api-与通信层)
9. [前端架构](#9-前端架构)
10. [设计模式](#10-设计模式)
11. [关键技术决策](#11-关键技术决策)
12. [系统边界与扩展点](#12-系统边界与扩展点)

---

## 1. 项目概述

eCan.ai 是一个 **AI Agent 驱动的自动化平台**，核心定位是面向电商/企业的智能体工作流系统。

### 核心能力

| 能力 | 实现方式 |
|------|----------|
| 多 LLM 支持 | OpenAI、Anthropic Claude、Ollama、Google GenAI、Deepseek、Qwen |
| 可视化工作流编排 | FlowGram.ai + LangGraph 状态机 |
| 多渠道消息接入 | Telegram、Slack、Discord、WhatsApp、微信、钉钉、Email |
| 浏览器自动化 | Selenium + Playwright + browser-use 三层 |
| OCR/视觉能力 | OpenCV + RapidOCR + Tesseract |
| RAG 知识库 | LightRAG / 标准 RAG |
| Agent 间通信 | A2A SDK 0.3.22 |
| 双端部署 | 桌面端（Qt）+ Web 端（FastAPI + React） |

---

## 2. 整体架构

### 2.1 顶层架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        eCan.ai Platform                         │
│                                                                 │
│  ┌──────────────────┐          ┌──────────────────────────────┐ │
│  │   Frontend       │          │      Backend (Python)        │ │
│  │   gui_v2/        │          │                              │ │
│  │  React + Vite    │◄────────►│  ┌──────────────────────┐   │ │
│  │  TypeScript      │  IPC /   │  │   FastAPI / Flask    │   │ │
│  │  Ant Design      │  HTTP /  │  │   web_server.py      │   │ │
│  │  Zustand         │  WS      │  └──────────┬───────────┘   │ │
│  └──────────────────┘          │             │               │ │
│                                │  ┌──────────▼───────────┐   │ │
│  ┌──────────────────┐          │  │   Agent Core         │   │ │
│  │   Legacy GUI     │          │  │   EC_Agent           │   │ │
│  │   gui/ (Qt)      │          │  │   EC_Skill           │   │ │
│  │  PySide6         │◄────────►│  │   TaskRunner         │   │ │
│  │  WebChannel      │          │  │   LangGraph          │   │ │
│  └──────────────────┘          │  └──────────┬───────────┘   │ │
│                                │             │               │ │
│  ┌──────────────────┐          │  ┌──────────▼───────────┐   │ │
│  │   CLI            │          │  │   Service Layer      │   │ │
│  │   cli/           │◄────────►│  │   SQLAlchemy ORM     │   │ │
│  └──────────────────┘          │  │   SQLite (WAL)       │   │ │
│                                │  └──────────────────────┘   │ │
└─────────────────────────────────────────────────────────────────┘
           │                                   │
           ▼                                   ▼
┌────────────────────┐              ┌─────────────────────┐
│  AWS AppSync       │              │  External Services  │
│  GraphQL (Cloud)   │              │  LLM APIs           │
│  Cognito Auth      │              │  MCP Servers        │
└────────────────────┘              │  A2A Protocol       │
                                    └─────────────────────┘
```

### 2.2 双端部署模式

```
桌面端模式 (Desktop)
━━━━━━━━━━━━━━━━━━━
React Frontend
     │
     │ Qt WebChannel (file:// protocol)
     │
Python Backend (PySide6 QWebEngineView)
     │
SQLite (本地)


Web 端模式 (Web / Cloud)
━━━━━━━━━━━━━━━━━━━━━━━
React Frontend (BrowserRouter)
     │
     ├──── HTTP/GraphQL ──► FastAPI / AWS AppSync
     │
     └──── WebSocket ──────► uvicorn
                              │
                         SQLite / RDS
```

`gui_v2/src/config/platform.ts` 通过检测是否为 `localhost`/私有 IP 或云端域名来动态切换通信策略。

---

## 3. 技术栈

### 3.1 后端

```
Python 3.10+
├── Web 框架
│   ├── FastAPI + uvicorn (HTTP API + WebSocket)
│   ├── Flask (部分接口)
│   └── Starlette (中间件、SSE)
│
├── AI/ML 核心
│   ├── LangGraph 1.0.5        ← 工作流状态机
│   ├── LangChain 生态          ← 工具链
│   │   ├── langchain-anthropic
│   │   ├── langchain-openai
│   │   └── langchain-core
│   ├── LangSmith              ← 可观测性
│   └── LangMem                ← 记忆管理
│
├── 浏览器自动化
│   ├── Playwright 1.54.0
│   ├── Selenium 4.32.0
│   └── browser-use 0.12.5
│
├── 视觉/OCR
│   ├── OpenCV (headless)
│   ├── RapidOCR (ONNX)
│   └── pytesseract
│
├── 数据库
│   ├── SQLAlchemy ORM
│   └── SQLite (WAL 模式)
│
├── 通信
│   ├── A2A SDK 0.3.22         ← Agent 间协议
│   ├── Socket.io
│   └── sse-starlette
│
└── 桌面 GUI
    └── PySide6 6.10.1 (Qt)
```

### 3.2 前端

```
Node 18+ / React 18.3.1 + TypeScript
├── 构建工具:  Vite 6.3.5
├── UI 组件:   Ant Design 5.26.7 + Semi Design 2.92.2
├── 状态管理:  Zustand 5.0.8 (+ persist)
├── 工作流编辑: FlowGram.ai 1.0.3
├── 代码编辑:  Monaco Editor 0.52.2
├── 图可视化:  React Sigma 5.0.4
├── 数据图表:  Ant Design Plots 2.4.0
├── 路由:      React Router 7.6.1
├── HTTP:      Axios 1.6.7
├── 认证:      Google OAuth + AWS Cognito
├── 国际化:    i18next (中文/英文)
└── 样式:      Emotion + Ant Design 主题
```

---

## 4. 模块划分

### 4.1 顶层目录结构

```
eCan.ai/
├── agent/              # 核心 Agent 系统 ★★★
│   ├── ec_agent.py     # EC_Agent 主类
│   ├── ec_skill.py     # EC_Skill（LangGraph 工作流）
│   ├── ec_tasks/       # 任务执行子系统
│   ├── db/             # 数据库层
│   ├── mcp/            # MCP 服务器/客户端
│   ├── channels/       # 多渠道消息适配器
│   └── ec_skills/      # 内置技能节点
│
├── gui_v2/             # 现代 React 前端 ★★★
│   └── src/
│       ├── pages/      # 页面组件
│       ├── stores/     # Zustand 状态
│       ├── services/   # API/通信服务
│       ├── components/ # 通用组件
│       └── config/     # 平台配置
│
├── gui/                # 遗留 Qt 前端（逐步废弃）
│   ├── ipc/            # IPC 通信层
│   └── windows/        # Qt 窗口
│
├── config/             # 应用配置
├── utils/              # 通用工具函数
├── common/             # 共享代码
├── auth/               # 认证模块
├── cli/                # 命令行接口
├── ota/                # OTA 更新系统
├── rag_worker/         # RAG 工作进程
├── wabaileys-bridge/   # WhatsApp 集成桥
├── build_system/       # 构建/部署工具
├── tests/              # 测试套件
├── knowledge/          # 知识库文档
├── scripts/            # 工具脚本
│
├── main.py             # 桌面端入口
├── web_server.py       # Web 端入口
├── app_context.py      # 全局应用上下文
└── ecan_cli.py         # CLI 入口
```

### 4.2 Agent 核心模块详解

```
agent/
├── ec_agent.py              # EC_Agent：浏览器 Agent 包装 + A2A + 技能管理
├── ec_skill.py              # EC_Skill：LangGraph StateGraph 包装 + MCP 集成
│
├── ec_tasks/                # 任务执行子系统
│   ├── executor.py          # TaskExecutor：流式执行、状态、中断
│   ├── runner.py            # TaskRunner：任务生命周期、调度、事件路由
│   ├── scheduler.py         # Scheduler：任务就绪检测
│   ├── models.py            # ManagedTask Pydantic 模型
│   └── message_sender.py    # ChatMessageSender：IPC/WebSocket 消息发送
│
├── db/                      # 数据库层
│   ├── models/              # SQLAlchemy 模型
│   │   ├── base_model.py    # BaseModel + TimestampMixin + CRUD
│   │   ├── agent_model.py   # DBAgent（层级、能力、A2A 字段）
│   │   ├── skill_model.py   # DBAgentSkill（工作流、标签）
│   │   ├── chat_model.py    # Chat/Member/Message/Notification
│   │   ├── task_model.py    # DBAgentTask
│   │   ├── tool_model.py    # DBAgentTool
│   │   ├── knowledge_model.py
│   │   └── avatar_model.py  # DBAvatarResource
│   └── services/            # 数据访问层
│       ├── base_service.py  # BaseService + Session 管理
│       ├── db_agent_service.py
│       ├── db_skill_service.py
│       ├── db_chat_service.py
│       └── db_task_service.py
│
├── mcp/                     # Model Context Protocol
│   ├── server/
│   │   ├── server.py        # MCP Server 实现
│   │   └── skill_editor_tools.py  # 技能编辑器专用工具
│   └── client/              # MCP 客户端适配器
│
├── channels/                # 多渠道消息系统
│   ├── base.py              # ChannelPlugin 基类 + 枚举
│   ├── channel_manager.py   # 生命周期管理 + 重启策略
│   ├── bridge.py            # 消息路由
│   └── adapters/            # 各平台适配器
│       ├── whatsapp/        # WhatsApp (Baileys + 传统)
│       ├── telegram/
│       ├── slack/
│       ├── discord/
│       ├── email/
│       ├── wechat/
│       └── dingtalk/
│
└── ec_skills/               # 内置技能节点库
    ├── browser_node/        # Playwright 浏览器节点
    ├── selenium_node/       # Selenium 浏览器节点
    ├── browser_use_extension/  # browser-use 扩展
    ├── ocr/                 # OCR/视觉节点
    ├── rag/                 # RAG 检索节点
    ├── llm_utils/           # LLM 工具函数
    ├── llm_hooks/           # LLM 生命周期钩子
    ├── knowledge_builder/   # 知识库构建
    ├── file_utils/          # 文件操作节点
    ├── sys_utils/           # 系统工具节点
    ├── git_utils/           # Git 操作节点
    └── node_runtime/        # 自定义节点运行时
```

---

## 5. 数据库设计

### 5.1 实体关系图

```
┌──────────────────────────────────────────────────────────────────┐
│                         数据库实体关系                            │
└──────────────────────────────────────────────────────────────────┘

DBAgentOrg (组织树)
    id, name, description, parent_id, org_type, level
    │ 1
    │ N
DBAgentOrgRel (M:M 关联)
    agent_id ──────────────────┐
    org_id   ──────────────────┤
                               │
DBAgent (智能体)               │
    id (UUID PK)               │◄──────────────────
    name, title, rank          │
    owner, gender              │
    supervisor_id (FK self)    │  ← 上级 Agent
    personalities (JSON)       │
    capabilities (JSON)        │
    a2a_* fields               │
    avatar_resource_id ────────┼──► DBAvatarResource
                               │        id, name, resource_type
                               │        image_url, video_path
    skills (M:M) ─────────────┼──► DBAgentSkill
    tasks  (1:N) ─────────────┼──► DBAgentTask
    tools  (M:M) ─────────────┼──► DBAgentTool
    knowledge (M:M) ──────────┤──► DBAgentKnowledge

DBAgentSkill (技能/工作流)
    id, name, owner
    path (文件路径)
    work_flow (JSON - LangGraph 序列化)
    diagram (JSON - FlowGram 图形数据)
    tags (JSON)

DBAgentTask (任务)
    id, name, status, priority
    owner, task_type, trigger_type
    schedule (cron/interval)
    last_run_at, next_run_at

Chat 系统
    Chat (id, type, name, agent_id)
     │ 1:N
    Member (chat_id, user_id, role)
     │ 1:N
    Message (chat_id, sender_id, content, timestamp)
     │ 1:N
    ChatNotification (chat_id, user_id, unread_count)

User 系统
    User (id, username, email, hashed_password)
     │ 1:N
    UserSession (user_id, session_id, token, expires_at)
```

### 5.2 关键设计特性

| 特性 | 实现 |
|------|------|
| 主键 | UUID 自动生成 |
| 时间戳 | TimestampMixin（timezone-aware created_at/updated_at） |
| 灵活字段 | JSON 列（personalities、capabilities、work_flow） |
| 并发安全 | SQLite WAL 模式 |
| 级联删除 | 关联表配置 cascade |
| 层级结构 | Agent.supervisor_id 自引用 FK + Org parent_id 树 |
| 扩展性 | ExtensibleMixin 支持自定义字段 |

---

## 6. 核心业务流程

### 6.1 Agent 任务执行流程

```
用户触发任务
     │
     ▼
Scheduler.detect_ready_tasks()
     │ 检查 status=pending + schedule
     ▼
TaskRunner.enqueue(task)
     │ 创建 ManagedTask
     ▼
TaskRunner.run_loop()
     │
     ├──► TaskExecutor.execute_stream()
     │         │
     │         ▼
     │    EC_Skill.invoke()
     │         │
     │         ▼
     │    LangGraph StateGraph.stream()
     │         │
     │    ┌────┴────────────────────┐
     │    │ 节点类型判断            │
     │    │                         │
     │    ├── LLM 节点             │
     │    │   └── 调用 LLM API     │
     │    ├── 浏览器节点           │
     │    │   └── Playwright/       │
     │    │       Selenium/         │
     │    │       browser-use       │
     │    ├── OCR 节点             │
     │    │   └── RapidOCR/        │
     │    │       Tesseract         │
     │    ├── RAG 节点             │
     │    │   └── 检索知识库       │
     │    ├── MCP 节点             │
     │    │   └── 调用 MCP 工具    │
     │    └── 代码节点             │
     │        └── 执行 Python/JS   │
     │                             │
     │    产生事件流 ◄─────────────┘
     │         │
     ▼         ▼
TaskRunner.event_queue
     │
     ├── 更新 task.status (DB)
     ├── 发送实时更新 (WebSocket/IPC)
     └── 处理中断请求 (pause/resume/cancel)
```

### 6.2 多渠道消息接收流程

```
外部消息到达（如 WhatsApp）
     │
     ▼
ChannelPlugin.on_message()  ← 每渠道独立线程
     │
     ▼
ChannelManager.route_message()
     │
     ▼
bridge.py 消息路由
     │ 匹配 agent 和 skill 规则
     ▼
EC_Agent.receive_message()
     │
     ▼
EC_Skill 触发对应工作流
     │
     ▼
TaskRunner 创建并执行新任务
     │
     ▼
ChatMessageSender 发送回复
     │
     ├── IPC (桌面端)
     └── WebSocket (Web 端)
```

### 6.3 Skill（工作流）编辑与保存流程

```
用户在 FlowGram.ai 编辑器中操作
     │
     ▼
FlowGram 节点/边操作
     │ React 状态更新
     ▼
SkillEditor Page 序列化
     │ 将图形数据转换为 LangGraph JSON
     ▼
IPC/HTTP 调用 save_skill
     │
     ▼
DBSkillService.save()
     │ 保存 work_flow (JSON) + diagram (JSON)
     ▼
SQLite 持久化

运行时：
DBSkillService.load() → EC_Skill.from_db() → LangGraph StateGraph 重建
```

### 6.4 A2A (Agent-to-Agent) 通信流程

```
Agent A 发起调用
     │
     ▼
EC_Agent.send_a2a_message(target_agent)
     │
     ▼
A2A SDK Client
     │ HTTP 调用目标 Agent 的 A2A 端点
     ▼
Agent B 的 A2A Server 接收
     │
     ▼
EC_Agent B 处理消息
     │ 可触发子任务
     ▼
响应返回 Agent A
```

### 6.5 前端 API 路由决策流程

```
前端发起 API 调用
     │
     ▼
api-router.ts 判断平台
     │
     ├── isDesktop() == true
     │   └──► IPC 调用
     │         │ Qt WebChannel → Python IPC Handler
     │         └── IPCHandlerRegistry.dispatch()
     │
     └── isDesktop() == false (Web)
         └──► HTTP/GraphQL 调用
               │
               ├── 本地开发: FastAPI REST
               └── 云端: AWS AppSync GraphQL
```

---

## 7. 关键类与函数关系

### 7.1 后端核心类层次

```
object
└── EC_Agent (agent/ec_agent.py)
    ├── 继承: browser_use.Agent
    ├── 属性: tasks[], skills[], supervisor, avatars[]
    ├── ec_skill: EC_Skill
    └── 方法:
        ├── run_task(task_name)
        ├── send_a2a_message(target, msg)
        ├── receive_message(channel_msg)
        └── get_skill(skill_name)

EC_Skill (agent/ec_skill.py)
    ├── 基类: Pydantic BaseModel
    ├── 属性: graph (StateGraph), state_schema, mcp_clients[]
    ├── default_mapping_rules: EventRoutingRules
    └── 方法:
        ├── invoke(input_state) → AsyncGenerator
        ├── from_db(db_skill) → EC_Skill
        └── to_langgraph() → StateGraph

TaskRunner (agent/ec_tasks/runner.py)
    ├── 属性: task_queue, event_queue, managed_tasks{}
    └── 方法:
        ├── enqueue(task: ManagedTask)
        ├── run_loop() [async]
        ├── pause_task(task_id)
        ├── resume_task(task_id)
        └── route_event(event)

TaskExecutor (agent/ec_tasks/executor.py)
    └── 方法:
        ├── execute_stream(skill, input) → AsyncGenerator
        ├── handle_interrupt(checkpoint)
        └── serialize_state(state)

ChannelManager (agent/channels/channel_manager.py)
    ├── 属性: plugins{}, status{}, threads{}
    └── 方法:
        ├── start_channel(name, config)
        ├── stop_channel(name)
        ├── restart_with_backoff(name)
        └── route_message(msg: ChannelMessage)

ChannelPlugin (agent/channels/base.py)  [Abstract]
    ├── 枚举: ChannelStatus (idle/starting/running/stopping/stopped/error)
    ├── ChannelMessage dataclass
    └── 方法 (override):
        ├── start()
        ├── stop()
        └── send_message(msg)

BaseModel (agent/db/models/base_model.py)
    ├── 混入: TimestampMixin, ExtensibleMixin
    └── 方法 (class):
        ├── create(**kwargs)
        ├── get(id)
        ├── update(id, **kwargs)
        └── delete(id)

BaseService (agent/db/services/base_service.py)
    ├── session: SQLAlchemy Session
    └── 方法:
        ├── get_session() → contextmanager
        └── 子类实现各领域 CRUD

AppContext (app_context.py)  [Singleton]
    ├── app: QApplication
    ├── main_window: MainWindow
    ├── config: AppSettings
    ├── logger: Logger
    └── 元类: 支持直接属性访问
```

### 7.2 前端核心服务关系

```
App.tsx (根组件)
├── 初始化 IPC Client (桌面)
├── 初始化 AWS AppSync (Web)
├── 启动 TokenRefreshService
├── 初始化 StoreSync
└── 渲染 ThemeProvider > RouterProvider

api-router.ts (APIRouter 单例)
├── isDesktop() → 选择 IPC 路径
├── isWeb()     → 选择 HTTP/GraphQL 路径
└── 方法:
    ├── query(gql, vars) → Promise
    ├── mutate(gql, vars) → Promise
    └── subscribe(gql, vars) → Observable

UnifiedChatService (services/chat/unifiedChatService.ts)
├── 桌面: 调用 IPC chat.* 方法
├── Web:  调用 AppSync chatApi
└── 方法:
    ├── sendMessage(chatId, content)
    ├── getMessages(chatId, cursor)
    └── subscribeToChat(chatId, callback)

Zustand Stores (stores/)
├── agentStore    → agents[], fetchAgents(), updateAgent()
├── userStore     → username, userId
├── appStore      → systemInfo, theme, language
├── contextStore  → activeContext, metadata
├── ragStore      → ragDocs[], ragConfig
└── toolStore     → tools[], toolConfig
```

### 7.3 关键函数调用链

```
任务触发:
Scheduler.check_ready()
  → TaskRunner.enqueue(ManagedTask)
    → TaskExecutor.execute_stream(EC_Skill, input)
      → EC_Skill.invoke(state)
        → LangGraph.StateGraph.astream(state)
          → [节点函数] → yield 事件
            → TaskRunner.event_queue.put(event)
              → ChatMessageSender.send(event)
                → IPC/WebSocket 推送前端

技能保存:
Frontend.SkillEditor.onSave()
  → serialize(flowgramGraph) → LangGraphJSON
    → apiRouter.mutate(SAVE_SKILL, {skill})
      → IPC/HTTP → Python
        → DBSkillService.save(skill_data)
          → SQLite INSERT/UPDATE
```

---

## 8. API 与通信层

### 8.1 IPC 通信（桌面端）

```python
# IPCHandlerRegistry (gui/ipc/registry.py)
# 中间件链：Token 验证 → 权限检查 → 处理器分发

class IPCHandlerRegistry:
    middleware: List[Middleware]
    whitelist: Set[str]          # 无需 Token 的方法
    handlers: Dict[str, Handler]

# 常用 IPC 方法
"login" / "signup" / "refresh_token"
"get_agents" / "save_agent" / "delete_agent"
"get_skills" / "save_skill" / "run_skill"
"chat.send_message" / "chat.get_messages"
"task.run" / "task.pause" / "task.resume"
"file.open_dialog" / "file.read" / "file.write"
```

### 8.2 GraphQL Schema（云端）

```graphql
# 主查询
GetAllMine(owner: String!, userId: String!)
  → { agents, tasks, skills, tools, knowledge, settings }

GetOrgAgentTree(rootId: String!, username: String!)
  → { org { id, name, level, children, agents } }

GetAgentTasks()
  → { tasks[] }

# 聊天
SendChatMessage(chatId, content, attachments)
GetChatMessages(chatId, cursor, limit)

# 订阅（实时）
OnTaskStatusChanged(taskId)
OnNewChatMessage(chatId)
OnAgentStatusChanged(agentId)
```

### 8.3 WebSocket 事件类型

| 事件 | 方向 | 描述 |
|------|------|------|
| `task.status` | Server→Client | 任务状态变更 |
| `task.log` | Server→Client | 任务执行日志流 |
| `chat.message` | Server→Client | 新聊天消息 |
| `agent.status` | Server→Client | Agent 在线状态 |
| `task.interrupt` | Client→Server | 暂停/恢复请求 |

---

## 9. 前端架构

### 9.1 页面结构

```
src/pages/
├── Agents/          # Agent 管理 + 组织导航
│   ├── AgentList    # Agent 列表
│   ├── AgentDetail  # Agent 详情/编辑
│   └── OrgTree      # 组织树形结构
├── Chat/            # 聊天界面（统一服务）
├── SkillEditor/     # 可视化工作流编辑器
│   ├── FlowGram 编辑器嵌入
│   ├── 自定义节点类型
│   └── Monaco 代码编辑
├── Tasks/           # 任务监控与管理
├── Skills/          # 技能库浏览
├── Dashboard/       # 概览与分析
├── Settings/        # 系统配置
├── Tools/           # 工具管理
├── RAG/             # RAG 文档管理
├── Knowledge/       # 知识库（LightRAG）
└── Account/         # 账户与计费
```

### 9.2 路由配置

```typescript
// routes/index.tsx - 懒加载 + 错误边界
const routes = [
  { path: "/", element: <Layout/>, children: [
    { path: "agents",      lazy: () => import("./Agents") },
    { path: "agents/:id",  lazy: () => import("./AgentDetail") },
    { path: "chat",        lazy: () => import("./Chat") },
    { path: "skill-editor/:skillId", lazy: () => import("./SkillEditor") },
    { path: "tasks",       lazy: () => import("./Tasks") },
    // ...
  ]},
  { path: "/login",  element: <Login/> },
  { path: "/signup", element: <Signup/> },
]

// 桌面端: HashRouter (兼容 file:// 协议)
// Web 端:  BrowserRouter
```

### 9.3 平台检测逻辑

```typescript
// config/platform.ts
export const isDesktop = () =>
  window.location.hostname === "localhost" ||
  isPrivateIP(window.location.hostname);

export const isWeb = () =>
  CLOUD_DOMAINS.some(d => window.location.hostname.includes(d));

// 特性标志基于平台
export const features = {
  ipc:         isDesktop(),
  cognito:     isWeb(),
  appSync:     isWeb(),
  localSQLite: isDesktop(),
};
```

---

## 10. 设计模式

### 10.1 单例模式 (Singleton)

| 类 | 位置 | 用途 |
|----|------|------|
| `AppContext` | `app_context.py` | 全局应用状态 |
| `IPCHandlerRegistry` | `gui/ipc/registry.py` | IPC 处理器注册 |
| `IPCAPI` | `gui/ipc/api.py` | 桌面通信 |
| `APIRouter` | `gui_v2/services/api/api-router.ts` | 前端路由决策 |
| Zustand Stores | `gui_v2/stores/*.ts` | 前端状态容器 |

### 10.2 策略模式 (Strategy)

```python
# LLM 提供者策略 - 运行时可切换
LLMStrategy:
├── OpenAIProvider    → ChatOpenAI
├── AnthropicProvider → ChatAnthropic
├── OllamaProvider    → ChatOllama
├── GoogleProvider    → ChatGoogleGenAI
├── DeepseekProvider  → ChatDeepseek
└── QwenProvider      → ChatQwen

# 浏览器自动化策略
BrowserStrategy:
├── PlaywrightBrowser
├── SeleniumBrowser
└── BrowserUseAgent

# 渠道通信策略
ChannelStrategy (ChannelPlugin 子类):
├── WhatsAppPlugin
├── TelegramPlugin
├── SlackPlugin
├── DiscordPlugin
└── EmailPlugin
```

### 10.3 观察者模式 (Observer)

```
事件系统:
TaskRunner.event_queue → 订阅者:
  ├── UI 更新推送 (WebSocket/IPC)
  ├── 数据库状态同步
  └── 下游 Agent 触发

Zustand 订阅:
store.subscribe(selector, callback) → 组件响应式更新
```

### 10.4 适配器模式 (Adapter)

```
browser-use Agent
    ▲ 适配
EC_Agent (ec_agent.py)
    └── 统一接口: run_task(), receive_message(), send_a2a_message()

A2A SDK
    ▲ 适配
EC_Agent A2A 方法
    └── 统一接口: send_to_agent(), handle_a2a_request()
```

### 10.5 仓储模式 (Repository)

```python
# 层次清晰的数据访问
BaseModel          # SQLAlchemy 映射 + 基础 CRUD
    └── DBAgent, DBAgentSkill, DBAgentTask...

BaseService        # 业务逻辑 + Session 管理
    └── DBAgentService.get_hierarchy_tree()
    └── DBSkillService.find_by_tags()
    └── DBChatService.get_recent_messages()
```

### 10.6 状态机模式 (State Machine)

```python
# 任务状态机
TaskStatus: pending → running → paused → running → completed
                                                 └── failed

# 渠道状态机
ChannelStatus: idle → starting → running → stopping → stopped
                                         └── error → (retry with backoff)

# LangGraph - 核心工作流状态机
StateGraph:
  nodes: {node_id: node_fn}
  edges: {from: to}
  conditional_edges: {from: routing_fn}
```

### 10.7 中间件模式 (Middleware)

```python
# IPC 中间件链
request
  → TokenValidationMiddleware
  → WhitelistCheckMiddleware
  → RateLimitMiddleware
  → HandlerDispatch
  → response
```

### 10.8 依赖注入 (Dependency Injection)

```
FlowGram.ai 插件系统:
editor.use(RoutingPlugin)
editor.use(CustomNodesPlugin)
editor.use(MonacoEditorPlugin)

A2A SDK 插件:
a2a_client.register_plugin(AuthPlugin)
a2a_client.register_plugin(LoggingPlugin)
```

---

## 11. 关键技术决策

| 决策 | 选择 | 理由 |
|------|------|------|
| 工作流引擎 | LangGraph（非 CrewAI/AutoGen） | 精细的状态控制、中断/恢复、检查点 |
| 数据库 | SQLite + WAL 模式 | 本地部署无需外部 DB，WAL 保证并发 |
| 前端状态 | Zustand（非 Redux） | 更轻量、hooks 友好、无样板代码 |
| IPC 方案 | Qt WebChannel + JSON | 统一桌面/Web 接口，前端无感知 |
| 浏览器自动化 | 三层方案（Selenium/Playwright/browser-use） | 兼容不同场景和复杂度 |
| 任务调度 | 自研 Scheduler + TaskRunner | 支持 Agent 特有的中断/事件路由 |
| 检查点序列化 | ormsgpack + Pickle 兜底 | 处理 LangGraph 深度递归限制 |
| 路由策略 | HashRouter(桌面)/BrowserRouter(Web) | file:// 协议兼容性 |
| 会话管理 | 登录后宽限期 | 处理后端初始化期间的 INVALID_TOKEN |
| OTA 更新 | 独立 ota/ 模块 | 支持桌面端热更新 |

---

## 12. 系统边界与扩展点

### 12.1 可扩展的接入点

```
1. LLM 提供者扩展
   agent/ec_skills/llm_utils/providers/ 新增 Provider 类

2. 渠道适配器扩展
   agent/channels/adapters/ 新增 ChannelPlugin 子类

3. 技能节点扩展
   agent/ec_skills/ 新增节点类型
   gui_v2 SkillEditor 注册对应的 FlowGram 节点

4. MCP 工具扩展
   agent/mcp/server/skill_editor_tools.py 注册新工具

5. IPC 处理器扩展
   gui/ipc/handlers.py 注册新处理器

6. API 端点扩展
   web_server.py 添加新路由
```

### 12.2 系统外部依赖

```
必须:
├── Python 3.10+
├── Node.js 18+
└── SQLite 3.35+

可选（按功能）:
├── GPU/ONNX Runtime (RapidOCR 加速)
├── Tesseract OCR (OCR 节点)
├── Chrome/Chromium (浏览器自动化)
├── AWS 账户 (Cognito + AppSync，Web 部署)
└── 各平台 Bot Token (Telegram/Slack/Discord 等)
```

---

## 附录：关键文件速查表

| 文件 | 职责 |
|------|------|
| `agent/ec_agent.py` | EC_Agent 主类，核心智能体 |
| `agent/ec_skill.py` | EC_Skill，LangGraph 工作流封装 |
| `agent/ec_tasks/executor.py` | 流式任务执行器 |
| `agent/ec_tasks/runner.py` | 任务生命周期管理 |
| `agent/channels/channel_manager.py` | 多渠道生命周期管理 |
| `agent/db/models/agent_model.py` | DBAgent 数据模型 |
| `agent/db/services/db_agent_service.py` | Agent CRUD 服务 |
| `app_context.py` | 全局单例上下文 |
| `web_server.py` | FastAPI 入口 |
| `gui/ipc/registry.py` | IPC 处理器注册与中间件 |
| `gui_v2/src/App.tsx` | React 应用根组件 |
| `gui_v2/src/config/platform.ts` | 平台检测与特性标志 |
| `gui_v2/src/services/api/api-router.ts` | API 路由决策 |
| `gui_v2/src/stores/agentStore.ts` | Agent 全局状态 |
| `gui_v2/src/services/chat/unifiedChatService.ts` | 统一聊天服务 |

---

*本文档由 eCan.ai 仓库自动深度分析生成，覆盖 40+ 核心文件。*
