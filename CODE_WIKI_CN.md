# eCan.ai 代码百科（Code Wiki）

> **版本：** 0.7.0（代码库）· **最新发布：** v0.8.24  
> **许可证：** MIT  
> **平台支持：** Windows · macOS · Linux · Web（Docker）

---

## 目录

1. [项目概述](#1-项目概述)
2. [整体架构](#2-整体架构)
3. [目录结构](#3-目录结构)
4. [核心入口文件](#4-核心入口文件)
   - [main.py — 桌面 GUI](#41-mainpy--桌面-gui)
   - [web_server.py — 无头 Web 服务器](#42-web_serverpy--无头-web-服务器)
   - [ecan_cli.py — CLI 入口](#43-ecan_clipy--cli-入口)
5. [模块详解](#5-模块详解)
   - [app_context.py — 全局应用上下文](#51-app_contextpy--全局应用上下文)
   - [agent/ — 核心智能体框架](#52-agent--核心智能体框架)
   - [auth/ — 身份认证](#53-auth--身份认证)
   - [cli/ — 命令行接口](#54-cli--命令行接口)
   - [common/ — 公共模型与服务](#55-common--公共模型与服务)
   - [config/ — 配置与设置](#56-config--配置与设置)
   - [gui/ — 桌面 Qt GUI](#57-gui--桌面-qt-gui)
   - [gui_v2/ — Web 前端（Vue.js）](#58-gui_v2--web-前端vuejs)
   - [knowledge/ — RAG 与 LightRAG](#59-knowledge--rag-与-lightrag)
   - [rag_worker/ — 独立 RAG 工作进程](#510-rag_worker--独立-rag-工作进程)
   - [ocr/ — 光学字符识别](#511-ocr--光学字符识别)
   - [e_commerce/ — 电商领域逻辑](#512-e_commerce--电商领域逻辑)
   - [telemetry/ — 可观测性](#513-telemetry--可观测性)
   - [utils/ — 公共工具集](#514-utils--公共工具集)
   - [ota/ — 空中升级（OTA）](#515-ota--空中升级ota)
   - [infrastructure/ — 云基础设施](#516-infrastructure--云基础设施)
   - [lambda_functions/ — AWS Lambda](#517-lambda_functions--aws-lambda)
   - [wabaileys-bridge/ — Node.js 桥接层](#518-wabaileys-bridge--nodejs-桥接层)
   - [build_system/ & build.py — 构建系统](#519-build_system--buildpy--构建系统)
   - [scripts/ — 部署与运维脚本](#520-scripts--部署与运维脚本)
   - [settings/ & docs/](#521-settings--docs)
6. [关键类与函数说明](#6-关键类与函数说明)
7. [智能体与技能系统深度解析](#7-智能体与技能系统深度解析)
8. [MCP 工具集成](#8-mcp-工具集成)
9. [IPC 通信架构](#9-ipc-通信架构)
10. [依赖关系图](#10-依赖关系图)
11. [配置参考](#11-配置参考)
12. [项目运行方式](#12-项目运行方式)
13. [测试](#13-测试)

---

## 1. 项目概述

**eCan.ai** 是一个 **AI 原生、隐私优先、跨平台的电商智能体平台**。它允许用户构建、部署并编排智能体，自动化执行多渠道电商工作流程。

**核心价值主张：**

| 功能 | 说明 |
|---|---|
| **智能体网络（A2A）** | 通过 A2A 协议跨主机（局域网/广域网）进行智能体通信 |
| **多智能体 / 多任务** | 并行执行多个智能体任务，支持调度与断点管理 |
| **LangGraph 工作流** | 基于状态机的智能体技能，采用 LangGraph 框架 |
| **Flowgram IDE** | 可视化拖拽式工作流编辑器，最终编译为 LangGraph |
| **浏览器自动化** | 集成 Playwright、Selenium、Crawl4ai、browser-use 实现 RPA |
| **计算机视觉 / OCR** | 截图理解、文字识别、RPA 自动化 |
| **134 个 MCP 工具** | 通过模型上下文协议（MCP）提供开箱即用的工具 |
| **RAG 知识库** | 基于 LightRAG 的向量 + 图谱混合检索 |
| **隐私优先** | 通过 Ollama 支持本地部署 LLM，数据不出私域 |
| **多渠道消息** | 支持 WhatsApp、Telegram、Slack、Email、SMS、Discord、微信、钉钉 |
| **多种部署方式** | 桌面应用、Web 服务、Docker、AWS ECS |

---

## 2. 整体架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           eCan.ai 平台                                  │
├────────────────────┬────────────────────────┬───────────────────────────┤
│   桌面 GUI         │     Web / 服务器模式    │        CLI 模式           │
│   (main.py)        │   (web_server.py)      │   (ecan_cli.py / cli/)    │
│   PySide6 / Qt     │   FastAPI + Uvicorn    │   Click 框架              │
└────────┬───────────┴──────────┬─────────────┴────────────┬──────────────┘
         │                      │                           │
         └──────────────────────▼───────────────────────────┘
                      AppContext（单例）
                         app_context.py
                  ┌─────────────────────────────────┐
                  │  配置 · 日志 · 登录              │
                  │  主窗口 · WebGUI                 │
                  │  线程池 · 事件循环               │
                  └────────────┬────────────────────┘
                               │
         ┌─────────────────────┼──────────────────────────┐
         │                     │                          │
         ▼                     ▼                          ▼
  ┌─────────────┐    ┌──────────────────┐       ┌────────────────┐
  │  gui/       │    │  agent/          │       │  knowledge/    │
  │  MainGUI    │◄─IPC─►EC_Agent       │◄─────►│  LightRAG      │
  │  WebGUI     │    │  EC_Skill        │       │  RAG 客户端    │
  │  LocalServer│    │  LangGraph 流程  │       │  文件提取      │
  └─────────────┘    └──────┬───────────┘       └────────────────┘
                             │
         ┌───────────────────┼──────────────────────┐
         │                   │                       │
         ▼                   ▼                       ▼
  ┌────────────┐   ┌──────────────────┐   ┌──────────────────┐
  │ agent/mcp  │   │  agent/a2a       │   │  agent/channels  │
  │ 134 个工具 │   │  A2A 协议        │   │  WhatsApp/TG/    │
  │ MCP 客户端 │   │  服务发现        │   │  Slack/Email...  │
  └────────────┘   └──────────────────┘   └──────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │  支撑层                                                          │
  │  auth/（Cognito/OAuth）· config/ · utils/ · telemetry/          │
  │  ota/（自动更新）· e_commerce/ · ocr/ · common/                 │
  └──────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │  部署层                                                          │
  │  桌面：PyInstaller 打包（Windows/macOS/Linux）                   │
  │  Web：Dockerfile.web → uvicorn                                   │
  │  工作节点：Dockerfile.worker → ECS Fargate + Xvfb               │
  │  云端：lambda_functions/ + infrastructure/（CloudFormation）     │
  └──────────────────────────────────────────────────────────────────┘
```

### 双 UI 策略

平台维护两套并行的 UI 实现，共享同一个 Python 后端：

- **`gui/`** — 基于 PySide6/Qt 的原生桌面 GUI，提供丰富的操作系统集成（系统托盘、文件对话框、原生菜单），用于打包后的桌面应用程序。
- **`gui_v2/`** — Vue.js + TypeScript 的 Web 前端 SPA。在开发模式下通过 Vite 热加载，生产模式下构建为静态文件。桌面应用中嵌入在 `QWebEngineView` 里，Web 模式下直接提供服务。

`gui/ipc/` 层是连接两套前端与 Python 后端的桥梁，通过 WebSocket 消息传递实现通信。

---

## 3. 目录结构

```
eCan.ai/
├── main.py                    # 桌面应用入口（1,120 行）
├── web_server.py              # Web/无头服务器入口
├── ecan_cli.py                # CLI 入口封装
├── app_context.py             # 全局应用上下文单例（380 行）
│
├── agent/                     # 核心智能体框架（23 个子目录）
│   ├── ec_agent.py            #   EC_Agent 类
│   ├── ec_skill.py            #   EC_Skill（LangGraph 封装）
│   ├── ec_org_ctrl.py         #   组织控制器
│   ├── ec_tasks/              #   任务管理与持久化
│   ├── ec_agents/             #   预构建智能体模板
│   ├── ec_skills/             #   技能库（26 个子目录）
│   │   ├── build_agent_skills.py  # 技能构建工厂
│   │   ├── build_node.py          # 节点构建工具
│   │   ├── flowgram2langgraph.py  # 可视化工作流 → LangGraph
│   │   ├── langgraph2flowgram.py  # LangGraph → 可视化工作流
│   │   ├── llm_hooks/             # LLM 集成钩子
│   │   ├── browser_use_extension/ # 浏览器自动化扩展
│   │   ├── browser_node/          # 浏览器动作节点实现
│   │   ├── ec_customer_support/   # 客服技能模板
│   │   ├── ecbot_rpa/             # RPA 监督员与操作员
│   │   └── knowledge_builder/     # 知识图谱构建技能
│   ├── a2a/                   #   智能体间通信协议（A2A）
│   │   ├── discovery/         #     局域网/云端服务发现
│   │   ├── specification/     #     A2A 协议规范
│   │   └── langgraph_agent/   #     LangGraph A2A 适配器
│   ├── mcp/                   #   模型上下文协议（134 个工具）
│   │   ├── server/
│   │   ├── client/
│   │   └── local_client.py    #     进程内直接调用
│   ├── playwright/            #   Playwright 浏览器配置
│   ├── cloud_worker/          #   AWS 云工作节点
│   ├── chats/                 #   统一消息集成
│   ├── channels/              #   多渠道通信驱动
│   ├── db/                    #   数据库模型与服务
│   ├── memory/                #   智能体记忆管理
│   ├── avatar/                #   头像/配置文件管理
│   ├── business/              #   业务逻辑
│   ├── prompts/               #   提示词模板与变量
│   └── human_chatter.py       #   人机对话工具
│
├── auth/                      # 身份认证（Cognito、OAuth）
│   ├── auth_manager.py        #   主认证控制器（63 KB）
│   ├── auth_config.py
│   ├── auth_config.yml
│   ├── aws_credentials_provider.py
│   ├── cognito/
│   └── oauth/
│       └── local_oauth_server.py
│
├── cli/                       # 基于 Click 的 CLI（15 个子目录）
│   ├── main.py                #   根命令组
│   ├── base/                  #   上下文、输出、装饰器
│   ├── auth/                  #   login、logout 命令
│   ├── server/                #   start、stop、logs 命令
│   ├── agents/                #   list、get、create、update
│   ├── skills/                #   技能操作
│   ├── tasks/                 #   任务管理
│   ├── vehicles/              #   主机管理
│   ├── tools/                 #   工具查询
│   ├── knowledge/             #   知识库操作
│   ├── prompts/               #   提示词管理
│   ├── dev/                   #   开发命令
│   ├── data/                  #   数据导入/导出
│   └── config/                #   配置命令
│
├── common/                    # 公共模型与服务
│   ├── db_init.py
│   ├── models/
│   └── services/
│
├── config/                    # 配置与应用设置
│   ├── app_info.py            #   AppInfo 单例（路径、版本）
│   ├── app_settings.py        #   AppSettings（Web 模式、目录）
│   ├── build_info.py          #   构建元数据与横幅
│   ├── constants.py           #   APP_NAME、超时时间、目录名
│   └── envi.py                #   环境变量辅助函数
│
├── gui/                       # 桌面 Qt GUI（11 个子目录）
│   ├── MainGUI.py             #   主窗口（299 KB）
│   ├── WebGUI.py              #   Web GUI 封装（88 KB）
│   ├── LoginoutGUI.py         #   登录对话框
│   ├── LocalServer.py         #   本地 HTTP 服务器（75 KB）
│   ├── async_preloader.py     #   后台模块预加载
│   ├── log_viewer.py          #   日志显示组件
│   ├── menu_manager.py        #   菜单栏管理
│   ├── messages.py            #   UI 消息字符串
│   ├── config/
│   ├── context/               #   会话与状态管理
│   ├── core/
│   ├── dialogs/
│   ├── ipc/                   #   进程间通信层
│   │   ├── registry.py        #     处理器注册与分发
│   │   ├── handlers.py        #     核心消息处理器
│   │   ├── context_handlers.py#     上下文感知处理器
│   │   ├── w2p_handlers.py    #     Web 到 Python 桥接器
│   │   └── callable/
│   ├── manager/
│   ├── tool/
│   └── webdriver/
│
├── gui_v2/                    # 现代 Web 前端（Vue.js + TypeScript）
│   ├── src/
│   │   ├── modules/           #   功能模块
│   │   │   ├── skill-editor/  #     Flowgram 可视化工作流编辑器
│   │   │   ├── agent-management/
│   │   │   ├── task-management/
│   │   │   └── knowledge-base/
│   │   ├── stores/            #   Pinia 状态管理
│   │   ├── router/            #   Vue Router
│   │   └── components/        #   可复用组件
│   ├── dist/                  #   生产环境构建产物
│   ├── vite.config.ts
│   ├── tsconfig.json
│   └── package.json
│
├── knowledge/                 # RAG 与 LightRAG（16 个文件，约 380 KB）
│   ├── lightrag_client.py     #   LightRAG API HTTP 客户端
│   ├── lightrag_server.py     #   独立 LightRAG 服务器
│   ├── lightrag_launcher.py   #   进程管理与启动
│   ├── lightrag_health.py     #   健康检查与监控
│   ├── lightrag_direct.py     #   直接（非 HTTP）API 访问
│   ├── advanced_chunker.py    #   智能文档分块
│   ├── file_extractors.py     #   PDF/Word/文本提取
│   ├── lightrag_config_manager.py
│   ├── lightrag_confidence_scorer.py
│   ├── provider_limits_validator.py
│   ├── rerank_score_normalizer.py
│   └── table_preprocessor.py
│
├── rag_worker/                # 基于 Docker 的 RAG 工作进程
├── e_commerce/                # 电商领域逻辑
│   └── inventories.py
├── ocr/                       # OCR 配置
│   └── ocr_app_cfg.json
├── telemetry/                 # 遥测服务
│   ├── service.py
│   └── views.py
├── utils/                     # 公共工具模块（25+）
├── ota/                       # 空中升级系统（10 个子目录）
├── infrastructure/            # CloudFormation、ECS、IAM
├── lambda_functions/          # AWS Lambda 处理器
├── lambda_layers/             # Lambda 层依赖包
├── wabaileys-bridge/          # Node.js IPC 桥接
├── build_system/              # 统一构建系统
├── scripts/                   # 部署与运维脚本（30+）
├── settings/                  # 应用配置模板
├── tests/                     # 测试套件（11 个子目录）
├── docs/                      # 文档（55+ 个 Markdown 文件）
├── data/                      # 运行时数据目录
├── resource/                  # 静态资源（图标、字体）
├── packages/                  # Python 包发行版
│
├── Dockerfile.web             # Web 服务器容器
├── Dockerfile.worker          # 云/自动化工作节点容器
├── docker-compose.web.yml     # 多服务 Web 部署
├── nginx.conf                 # 反向代理配置
├── requirements-base.txt      # 核心 Python 依赖
├── requirements-web.txt       # Web 模式依赖（无 Qt）
├── requirements-worker.txt    # 工作节点容器依赖
├── requirements-linux.txt     # Linux 附加依赖
├── requirements-macos.txt     # macOS 附加依赖
├── requirements-windows.txt   # Windows 附加依赖
├── mcp_tools_schema.json      # 134 个 MCP 工具定义（202 KB）
├── schema_03_15.graphql       # AWS AppSync GraphQL Schema
├── pytest.ini                 # Pytest 配置
├── VERSION                    # 当前版本号
├── CHANGELOG.md               # 更新日志
└── README.md                  # 用户指南与快速入门
```

---

## 4. 核心入口文件

### 4.1 `main.py` — 桌面 GUI

**大小：** 约 1,120 行 · **作用：** 启动完整的桌面应用程序（含 Qt GUI）。

**启动流程：**

```
main.py
 │
 ├── 1. 关键补丁（第 11–149 行）
 │      ├── LangGraph 检查点序列化器 → ormsgpack 递归深度限制修复
 │      ├── browser-use UTF-8 编码修复（Windows GBK 问题）
 │      ├── 平台版本检测补丁（隐藏控制台窗口）
 │      └── FFI/Pydantic 弃用警告抑制
 │
 ├── 2. Windows 兼容性处理（第 215–371 行）
 │      ├── ProcessPoolExecutor 辅助进程检测
 │      ├── 多进程引导配置（spawn vs fork）
 │      ├── 诊断子进程监控（EC_DIAG=1）
 │      └── 窗口枚举日志（调试用）
 │
 └── 3. 启动流程（第 538–1073 行）
        ├── 平台检测（Linux 显示服务器检查）
        ├── 创建 QApplication + 单实例守卫
        ├── 最小启动画面（立即视觉反馈）
        ├── 完整启动画面 + 异步预加载开始
        ├── 初始化 Config、Logger、AppContext、AppSettings
        ├── 配置 qasync 事件循环
        ├── 初始化登录系统
        ├── 创建 WebGUI 实例
        ├── 启动本地服务器
        ├── 代理环境配置（后台线程）
        ├── OTA 更新后台检查
        ├── 注册 URL Scheme 处理器
        └── 进入 Qt 事件循环
```

**退出时清理：** 停止 OTA 更新器、释放操作系统睡眠锁、注销局域网/云端服务。

---

### 4.2 `web_server.py` — 无头 Web 服务器

**作用：** 以多用户 WebSocket 服务器模式运行 eCan.ai，无需任何 Qt GUI 依赖。

**关键函数：**

| 函数 | 说明 |
|---|---|
| `setup_logging()` | 配置统一日志系统 |
| `load_handlers()` | 从 `gui.ipc` 模块注册 IPC 处理器 |
| `setup_session_manager()` | 初始化 `SessionManager` 单例 |

**环境变量：**

| 变量 | 默认值 | 说明 |
|---|---|---|
| `ECAN_MODE` | `web`（自动设置） | 选择无头模式 |
| `ECAN_WS_HOST` | `0.0.0.0` | WebSocket 绑定地址 |
| `ECAN_WS_PORT` | `8765` | WebSocket 端口 |
| `ECAN_LOG_LEVEL` | `INFO` | 日志详细程度 |

---

### 4.3 `ecan_cli.py` — CLI 入口

仅 30 行的轻量封装：
1. 设置 `PYTHONUTF8=1`（全局强制 UTF-8）
2. 设置 `ECAN_MODE=web`（CLI 使用无头模式）
3. 委托给 `cli.main.main()` 执行

---

## 5. 模块详解

### 5.1 `app_context.py` — 全局应用上下文

**大小：** 380 行 · **模式：** 通过 `AppContextMeta` 元类实现单例。

**核心属性：**

```python
class AppContext:
    app: QApplication            # Qt 应用实例
    main_window: MainWindow      # 桌面 GUI 主窗口
    web_gui: WebGUI              # Web GUI 封装
    logger: Logger               # 统一日志器
    config: AppSettings          # 配置对象
    thread_pool: QThreadPool     # 异步任务线程池
    app_info: AppInfo            # 应用元数据（路径、版本）
    main_loop: AbstractEventLoop # asyncio 事件循环
    login: Login                 # 登录控制器
    playwright_browsers_path: str
```

**RunContext 惰性加载命名空间：**

| 命名空间 | 加载的对象 |
|---|---|
| `core()` | `EC_Skill`、`WorkFlowContext`、`node_builder`、`BreakpointManager` |
| `graph()` | LangGraph 的 `StateGraph`、`END`、`Runtime`、`GraphInterrupt` |
| `llm()` | LLM 钩子、异步工具、JSON 解析 |
| `mcp()` | MCP 工具调用客户端 |
| `log()` | 日志器、traceback 辅助函数 |

**线程安全访问器：**

```python
AppContext.get_instance()          # 单例访问器
AppContext.safe_get_login()        # 验证 login 对象可用性
AppContext.safe_get_main_window()
AppContext.safe_get_web_gui()
```

---

### 5.2 `agent/` — 核心智能体框架

平台的核心，包含所有 AI 智能体逻辑、技能定义、通信和自动化功能。

#### `agent/ec_agent.py` — EC_Agent

继承自 browser-use 的 `Agent`。管理任务（`TaskRunner`、`ManagedTask`），集成技能列表（`EC_Skill`），支持 A2A 协议消息传递，处理云端与本地部署。

主要属性：`skills: List[EC_Skill]`、`rank: str`（层级角色）、`cloud_id: str`

#### `agent/ec_skill.py` — EC_Skill

核心技能抽象，将 LangGraph `StateGraph` 与元数据和工具访问封装在一起：

```python
class EC_Skill(AgentSkill):
    id: str                           # 唯一技能 ID
    cloud_id: str                     # 云端 UUID
    work_flow: StateGraph             # LangGraph 工作流定义
    diagram: dict                     # Flowgram 可视化表示
    runnable: CompiledStateGraph      # 编译后的可执行工作流
    mcp_client: MultiServerMCPClient  # MCP 工具访问
    name: str
    description: str
    run_mode: str = "released"        # "development" | "released"
    source: str = "ui"               # "code" | "example" | "ui"
    mapping_rules: dict               # 状态恢复/映射 DSL
    version: str = "0.0.0"
    objectives: List[str]
    need_inputs: List[dict]
    tags, examples, inputModes, outputModes
```

**状态映射 DSL**（`mapping_rules`）：声明式规则，将事件数据字段映射到技能状态字段，支持：
- 字段路径映射：`event.data.X` → `state.attributes.Y`
- 冲突解决策略：`merge_deep`（深度合并）、`overwrite`（覆盖）
- 转换函数：`to_string`
- 应用顺序：`top_down`（自顶向下）

#### `agent/ec_skills/` — 技能库

| 文件/目录 | 作用 |
|---|---|
| `build_agent_skills.py` | 技能构建工厂，从定义实例化技能 |
| `build_node.py` | LangGraph 节点构建工具 |
| `flowgram2langgraph.py` | **编译** Flowgram 可视化图 → LangGraph `StateGraph` |
| `langgraph2flowgram.py` | **反编译** LangGraph → Flowgram 可视化图 |
| `llm_hooks/` | LLM 集成钩子（前/后处理） |
| `browser_use_extension/` | browser-use 库扩展 |
| `browser_node/` | 浏览器动作节点实现 |
| `ec_customer_support/` | 客服技能模板 |
| `ecbot_rpa/` | RPA 监督员与操作员技能实现 |
| `knowledge_builder/` | 构建知识图谱的技能 |

#### `agent/a2a/` — 智能体间通信协议

实现跨主机 A2A 通信协议：

- `discovery/` — 通过局域网广播和云端注册表发现智能体
- `specification/` — A2A 协议消息格式定义
- `langgraph_agent/` — 将 LangGraph 智能体适配为 A2A 通信

#### `agent/mcp/` — 模型上下文协议

通过 MCP 协议提供 134 个工具：

- `server/` — MCP 工具服务端实现
- `client/` — 多服务器 MCP 客户端封装
- `local_client.py` — 进程内直接工具调用（高性能路径）

#### `agent/channels/` — 多渠道消息

每个通信渠道的驱动程序：
WhatsApp · Telegram · Slack · Email · SMS · Discord · 微信 · 钉钉

#### `agent/db/` — 数据库层

智能体相关数据持久化的 SQLAlchemy 模型和服务类。

#### `agent/memory/` — 智能体记忆

通过 Langmem 集成管理智能体的短期和长期记忆。

---

### 5.3 `auth/` — 身份认证

**`AuthManager`**（`auth_manager.py`，63 KB）是中央认证控制器：

```python
class AuthManager:
    cognito_service: CognitoService  # AWS Cognito 提供商
    tokens: Dict                      # JWT 令牌存储
    current_user: str
    user_profile: Dict               # 姓名、头像等信息
    signed_in: bool
    machine_role: str = "Platoon"    # 智能体层级角色
    acct_file: str                   # uli.json（令牌持久化）
    _keychain_available: bool        # macOS/Windows 钥匙串
```

**功能：** AWS Cognito JWT 认证、本地回调服务器的 OAuth/OIDC 流程、通过 `uli.json` 持久化令牌、操作系统钥匙串集成、多机器角色支持（排（Platoon）/指挥官（Commander）层级）。

---

### 5.4 `cli/` — 命令行接口

基于 **Click** 构建，根命令组定义在 `cli/main.py`。

**全局选项：** `--json` · `--quiet` · `--verbose` · `--no-color`

**命令分类：**

| 分类 | 命令示例 |
|---|---|
| **查询** | `agents list`、`skills list`、`tasks list`、`tools list`、`knowledge search` |
| **控制** | `server start/stop`、`auth login/logout`、`agents start/stop` |
| **操作** | `agents create/update`、`skills add/remove`、`knowledge add` |

**`cli/base/`** 提供共享基础设施：
- `context.py` — CLI 上下文管理
- `output.py` — `OutputLevel`、格式化输出辅助
- `decorators.py` — 可复用 Click 装饰器
- `config.py` — CLI 配置加载

---

### 5.5 `common/` — 公共模型与服务

```
common/
├── db_init.py       # 数据库连接与 Schema 初始化
├── models/          # Pydantic 和 SQLAlchemy 模型定义
└── services/        # 跨模块共享的业务逻辑服务
```

---

### 5.6 `config/` — 配置与设置

#### `AppInfo`（`config/app_info.py`，154 行）

保存运行时路径信息的单例：

```python
class AppInfo:
    app_home_path: str       # 根目录（开发时为 cwd，打包时为 PyInstaller _MEIPASS）
    app_resources_path: str  # resource/ 目录
    appdata_path: str        # 操作系统用户数据目录：
                             #   Windows: %LOCALAPPDATA%/eCan
                             #   macOS:   ~/Library/Application Support/eCan
                             #   Linux:   ~/.local/share/eCan
    appdata_temp_path: str   # 临时目录
    version: str             # 从 VERSION 文件读取
```

#### `AppSettings`（`config/app_settings.py`，173 行）

GUI 模式的运行时配置：

```python
class AppSettings:
    root_dir: Path           # 项目根目录（自动检测）
    gui_v2_dir: Path         # gui_v2/ 目录
    dist_dir: Path           # gui_v2/dist（已构建前端）
    web_mode: str            # "dev" | "prod"
    vite_dev_server: str     # "http://localhost:3000"

    @property
    def is_dev_mode(self) -> bool   # web_mode == "dev"
    @property
    def is_prod_mode(self) -> bool
    def get_web_url(self) -> str    # 返回 Vite 开发服务器或 file:// URL
```

#### `constants.py` — 关键常量

| 常量 | 值 |
|---|---|
| `APP_NAME` | `"eCan"` |
| `DEFAULT_API_TIMEOUT` | `60.0` 秒 |
| `EXTENDED_API_TIMEOUT` | `120.0` 秒（慢速 API） |
| `FOLDER_DATA` | 数据目录名 |
| `FOLDER_RUNLOGS` | 日志目录名 |
| `FOLDER_SKILLS` | 技能目录名 |

---

### 5.7 `gui/` — 桌面 Qt GUI

基于 **PySide6**（Qt6）构建，是代码库中最大的模块。

| 文件 | 大小 | 作用 |
|---|---|---|
| `MainGUI.py` | 299 KB | 应用主窗口 |
| `WebGUI.py` | 88 KB | 托管 `gui_v2` 的 `QWebEngineView` 封装 |
| `LocalServer.py` | 75 KB | 用于 IPC 的本地 HTTP/WebSocket 服务器 |
| `menu_manager.py` | 75 KB | 菜单栏与系统托盘管理 |
| `LoginoutGUI.py` | 31 KB | 登录/注销对话框 |
| `log_viewer.py` | 38 KB | 实时日志显示组件 |
| `async_preloader.py` | 17 KB | 重量级模块后台预加载 |
| `lightrag_rerank_proxy.py` | 36 KB | RAG 重排序代理 UI |

**`gui/ipc/`** — 连接前端与 Python 后端的 IPC 层：

| 文件 | 作用 |
|---|---|
| `registry.py` | 处理器注册与消息分发表 |
| `handlers.py` | 核心消息处理器 |
| `context_handlers.py` | 上下文感知处理器 |
| `w2p_handlers.py` | Web 到 Python 桥接处理器 |
| `callable/` | 可调用处理器的装饰器工具 |

---

### 5.8 `gui_v2/` — Web 前端（Vue.js）

现代 TypeScript/Vue.js 单页应用（SPA）。桌面模式下嵌入 `QWebEngineView`；Web 模式下作为静态文件提供服务。

**技术栈：** Vue 3 · TypeScript · Pinia（状态管理）· Vue Router · Vite

**主要功能模块（`src/modules/`）：**

| 模块 | 说明 |
|---|---|
| `skill-editor/` | **Flowgram IDE** — 可视化拖拽工作流构建器 |
| `agent-management/` | 智能体列表、配置、状态监控 |
| `task-management/` | 任务调度、监控、历史记录 |
| `knowledge-base/` | 文档上传、RAG 搜索、知识管理 |

**构建产物：** `gui_v2/dist/` — 生产模式下提供服务。

---

### 5.9 `knowledge/` — RAG 与 LightRAG

基于 **LightRAG** 的图谱增强检索，融合向量搜索与知识图谱。

| 文件 | 作用 |
|---|---|
| `lightrag_client.py`（59 KB） | HTTP 客户端，调用远程 LightRAG 服务器 |
| `lightrag_server.py`（70 KB） | 独立 LightRAG 服务器（FastAPI） |
| `lightrag_launcher.py`（21 KB） | 进程管理器，启动/停止服务器子进程 |
| `lightrag_health.py`（23 KB） | 健康检查、存活监控 |
| `lightrag_direct.py`（9 KB） | 进程内直接 API（绕过 HTTP） |
| `advanced_chunker.py`（32 KB） | 智能文档分块策略 |
| `file_extractors.py`（14 KB） | 提取 PDF、Word、纯文本内容 |
| `lightrag_config_manager.py`（28 KB） | 配置持久化 |
| `lightrag_confidence_scorer.py`（28 KB） | RAG 结果置信度评分 |
| `provider_limits_validator.py`（10 KB） | LLM 速率限制处理 |
| `rerank_score_normalizer.py`（11 KB） | 重排序分数归一化 |
| `table_preprocessor.py`（12 KB） | 结构化表格数据提取 |
| `stop_controller.py`（5 KB） | 优雅关闭协调 |

**RAG 数据管道：**
```
文档 → file_extractors → advanced_chunker → LightRAG 摄取
查询 → lightrag_client → （向量 + 图谱搜索）→ confidence_scorer → 结果
```

---

### 5.10 `rag_worker/` — 独立 RAG 工作进程

用于卸载 RAG 处理任务的 Docker 化工作进程：

```
rag_worker/
├── main.py          # 工作进程入口（20 KB）
├── Dockerfile       # 容器定义
├── build.sh
└── requirements.txt
```

---

### 5.11 `ocr/` — 光学字符识别

```
ocr/
└── ocr_app_cfg.json  # OCR 服务配置（引擎选择、参数）
```

RPA 节点通过 **RapidOCR** 和 **pytesseract** 进行屏幕文字提取。

---

### 5.12 `e_commerce/` — 电商领域逻辑

```
e_commerce/
└── inventories.py   # 库存管理工具（2.4 KB）
```

---

### 5.13 `telemetry/` — 可观测性

```
telemetry/
├── service.py  # 遥测事件收集与上报（2.8 KB）
└── views.py    # 遥测数据视图（1.2 KB）
```

---

### 5.14 `utils/` — 公共工具集

25+ 个工具模块（约 376 KB）。核心模块：

| 模块 | 作用 |
|---|---|
| `logger_helper.py`（21 KB） | 统一日志：文件轮转、日志级别、崩溃日志 |
| `crash_boundary.py`（11 KB） | 崩溃恢复检测，通过心跳文件监控，支持检查点 |
| `memory_monitor.py`（30 KB） | 实时内存跟踪，通过 tracemalloc 检测泄漏 |
| `app_setup_helper.py`（34 KB） | 应用图标设置、桌面文件配置 |
| `single_instance.py`（12 KB） | 防止应用多实例运行（平台特定锁） |
| `sleep_inhibitor.py`（9 KB） | 长时间智能体任务期间阻止操作系统休眠 |
| `power_monitor.py`（7 KB） | 检测系统睡眠/唤醒事件 |
| `url_scheme_handler.py`（15 KB） | 自定义 URL Scheme 路由（如 `ecan://`） |
| `port_allocator.py`（8 KB） | 为本地服务动态分配端口 |
| `subprocess_helper.py`（8 KB） | 安全的子进程管理 |
| `permission_helper.py`（8 KB） | 操作系统权限管理 |
| `linux_permissions.py`（8 KB） | Linux 依赖与能力检查 |
| `lazy_import.py`（4 KB） | 延迟模块导入，减少启动时间 |
| `i18n_helper.py`（4 KB） | 国际化支持 |
| `hot_reload.py`（3 KB） | 仅开发模式使用的 GUI 热重载 |
| `data_uri_sanitizer.py`（7 KB） | Data URI 安全过滤 |
| `gui_dispatch.py`（2 KB） | 线程安全的 Qt GUI 调度器 |
| `win_subproc.py` | Windows 子进程补丁 |

**崩溃边界** 通过监控心跳文件运作：若应用在写入干净退出标志前崩溃，下次启动时会检测到并上报诊断信息。

**内存监控** 阈值：
- 增长告警：> 30 MB/分钟
- 快照差异间隔：120 秒

---

### 5.15 `ota/` — 空中升级（OTA）

```
ota/
├── core/           # 核心更新逻辑（下载、验证、安装）
├── config/         # 环境配置（dev/staging/prod）
├── gui/            # 更新对话框与进度 UI
├── server/         # OTA 服务端接口
├── scripts/        # 辅助脚本
├── certificates/   # Ed25519 签名密钥
└── docs/
```

**功能：**
- 对每个更新包进行 **Ed25519** 数字签名验证
- 支持多环境（开发、预发、生产）
- S3 存储 + 加速传输 URL
- 兼容 **Sparkle** Appcast 格式（macOS 标准）
- 后台静默下载，用户可见安装对话框
- 优雅安装，支持回滚

---

### 5.16 `infrastructure/` — 云基础设施

AWS 基础设施即代码：

```
infrastructure/
├── cloudformation/  # CloudFormation 堆栈模板
├── ecs/             # ECS 任务与服务定义
└── iam/             # IAM 角色与策略
```

支持 ECS Fargate 进行可扩展的云工作节点部署。

---

### 5.17 `lambda_functions/` — AWS Lambda

| 函数 | 作用 |
|---|---|
| `agentScheduler/` | 智能体任务调度 |
| `chatter/` | 聊天消息路由与处理 |
| `skill_editor_lambda/` | 技能编辑器 API 后端 |
| `presigned_link_publisher/` | S3 预签名 URL 生成 |
| `cloud_tester/` | 云集成测试 |
| `myAPIKeygen.zip` | API 密钥生成 |

**AppSync 集成：** 解析器文档见 `lambda_functions/resolvers.md`；Schema 见 `scripts/appsync_schema_latest.graphql`。

---

### 5.18 `wabaileys-bridge/` — Node.js 桥接层

约 14 KB 的 `index.js` Node.js 组件，作为 IPC 桥接层使用，主要用于 WhatsApp 集成和其他 Node.js 原生通信协议。

---

### 5.19 `build_system/` & `build.py` — 构建系统

**`build.py`** 是面向用户的构建封装脚本，支持三种模式：

| 命令 | 模式 | 说明 | 预计时间 |
|---|---|---|---|
| `python build.py fast` | 开发 | 增量缓存，跳过优化 | 2–5 分钟 |
| `python build.py dev` | 调试 | 含调试符号的完整调试构建 | 5–10 分钟 |
| `python build.py prod` | 生产 | 完全优化，LZMA 压缩 | 15–25 分钟 |

**`build_system/`** 包含：
- 跨 CPU 核心并行编译
- PyInstaller spec 文件管理
- 数据文件收集（开发最小化，生产完整）
- 平台特定打包（Windows NSIS、macOS DMG、Linux AppImage）

---

### 5.20 `scripts/` — 部署与运维脚本

30+ 个运维脚本：

| 脚本 | 作用 |
|---|---|
| `deploy-ubuntu.sh` | Ubuntu 服务器完整部署（9.8 KB） |
| `deploy-all.sh` | 部署所有服务 |
| `deploy-worker.sh` | 部署云工作节点 |
| `deploy-cloudformation.sh` | AWS CloudFormation 部署 |
| `build-and-push-worker.sh` | 构建并推送 Docker 工作节点镜像 |
| `ota_e2e_simulation.py` | OTA 更新端到端模拟（20 KB） |
| `ota_regression_test.py` | OTA 回归测试套件（24 KB） |
| `prompt_files2db.sh` | 初始化提示词数据库 |
| `create_macos_icns.sh` | 创建 macOS 应用图标 |
| `rds_ddl/` | RDS 数据表 DDL 定义 |
| `appsync_vtl/` | AppSync VTL 解析器模板 |

---

### 5.21 `settings/` & `docs/`

**`settings/`** 包含默认配置模板（如 `role.json`，定义默认机器角色）。

**`docs/`** — 55+ 个 Markdown 文件（约 660 KB），涵盖：
- 架构深度解析：`ipc-design.md`、`build-architecture.md`、`EVENT_DRIVEN_CHAT_ARCHITECTURE.md`
- 功能指南：`BROWSER_AUTOMATION_NODE_CONFIG.md`、`eCanMCP.md`、`AVATAR_GUIDE.md`
- 运维手册：`DEPLOYMENT_UBUNTU.md`、`WEB_DEPLOYMENT.md`、`OTA_PATH_STRUCTURE.md`、`RELEASE_GUIDE.md`
- 技术参考：`mapping-dsl.md`、`prompt-variable-resolution.md`、`MEMORY_MONITOR.md`
- 中文文档：`docs/cn/`
- 完整用户手册：`docs/user_manual.html`（57 KB）

---

## 6. 关键类与函数说明

| 类 / 函数 | 所在位置 | 作用 |
|---|---|---|
| `AppContext` | `app_context.py` | 保存所有运行时引用的全局单例 |
| `AppInfo` | `config/app_info.py` | 操作系统路径、版本信息 |
| `AppSettings` | `config/app_settings.py` | 运行时配置（开发/生产模式、URL） |
| `EC_Agent` | `agent/ec_agent.py` | 顶层 AI 智能体，编排技能与任务 |
| `EC_Skill` | `agent/ec_skill.py` | LangGraph 工作流 + 元数据 + MCP 工具 |
| `AuthManager` | `auth/auth_manager.py` | Cognito/OAuth 认证控制器 |
| `MainGUI` | `gui/MainGUI.py` | Qt 桌面主窗口 |
| `WebGUI` | `gui/WebGUI.py` | 托管 Web 前端的 `QWebEngineView` 封装 |
| `LocalServer` | `gui/LocalServer.py` | GUI 与后端 IPC 的本地 HTTP/WebSocket 服务器 |
| `flowgram2langgraph()` | `agent/ec_skills/flowgram2langgraph.py` | 将可视化图编译为 LangGraph |
| `langgraph2flowgram()` | `agent/ec_skills/langgraph2flowgram.py` | 将 LangGraph 反编译为可视化图 |
| `LightRAGClient` | `knowledge/lightrag_client.py` | LightRAG 服务器 HTTP 客户端 |
| `AdvancedChunker` | `knowledge/advanced_chunker.py` | 文档智能分块 |
| `ConfidenceScorer` | `knowledge/lightrag_confidence_scorer.py` | RAG 结果置信度评分 |
| `CrashBoundary` | `utils/crash_boundary.py` | 通过心跳文件检测上次崩溃 |
| `MemoryMonitor` | `utils/memory_monitor.py` | 内存使用跟踪，泄漏告警 |
| `SingleInstance` | `utils/single_instance.py` | 阻止应用多实例运行 |
| `PortAllocator` | `utils/port_allocator.py` | 动态分配本地端口 |
| `OTAUpdater` | `ota/core/` | 下载、Ed25519 验证并安装更新 |

---

## 7. 智能体与技能系统深度解析

### 技能生命周期

```
用户（Flowgram IDE）
      │
      ▼ 拖拽节点，连接边
  gui_v2/modules/skill-editor/
      │
      ▼ 序列化为 diagram JSON（Flowgram 格式）
  EC_Skill.diagram: dict
      │
      ▼ flowgram2langgraph.py
  EC_Skill.work_flow: StateGraph（LangGraph）
      │
      ▼ StateGraph.compile()
  EC_Skill.runnable: CompiledStateGraph
      │
      ▼ EC_Agent.run_skill(skill, inputs)
  LangGraph 执行（逐节点状态转换）
      │
      ▼ 通过 EC_Skill.mcp_client 调用 MCP 工具
  结果 → 状态
      │
      ▼ 应用 mapping_rules（DSL）
  最终输出返回
```

### 智能体层级与 A2A 通信

```
指挥官智能体（高层规划）
      │  A2A 协议
      ▼
排级智能体（中层协调）
      │  A2A 协议
      ▼
操作员智能体（任务执行：浏览器、RPA、消息）
```

智能体通过以下方式发现彼此：
1. **局域网发现** — mDNS/广播（本地网络内）
2. **云端注册表** — 中央服务（AWS AppSync/Lambda）

### 运行模式

| 模式 | 说明 |
|---|---|
| `released`（发布） | 生产模式，严格状态校验 |
| `development`（开发） | 宽松模式，校验较少，便于调试 |

---

## 8. MCP 工具集成

`mcp_tools_schema.json`（202 KB）定义了 **134 个工具**，按类别划分：

| 类别 | 示例 |
|---|---|
| **RPA** | `rpa_supervisor_scheduling_work`、`rpa_operator_dispatch_works` |
| **浏览器** | 导航、点击、内容提取、截图 |
| **知识库** | 搜索、摄取、查询知识库 |
| **数据处理** | 解析、转换、校验结构化数据 |
| **通信** | 跨渠道消息发送 |
| **电商** | 商品上架、库存管理、订单处理 |

工具通过 MCP 协议从 `agent/mcp/server/` 提供服务，技能通过 `EC_Skill.mcp_client`（`MultiServerMCPClient`）访问工具。

`agent/mcp/local_client.py` 提供进程内直接工具调用，用于性能敏感路径。

---

## 9. IPC 通信架构

Vue.js 前端与 Python 后端之间的通信采用事件驱动的 WebSocket 模式：

```
gui_v2（Vue.js）
    │
    │  WebSocket 消息（JSON）
    ▼
gui/LocalServer.py（localhost 上的 WebSocket 服务器）
    │
    │  按消息类型分发
    ▼
gui/ipc/registry.py（处理器分发表）
    │
    ├── handlers.py          （核心操作）
    ├── context_handlers.py  （上下文感知）
    └── w2p_handlers.py      （Web 到 Python 桥接）
         │
         ▼
    Python 后端（agent/、knowledge/、auth/ 等）
```

在 Web 部署模式下，`web_server.py` 替换 `LocalServer.py` 作为 WebSocket 端点，但复用相同的 `gui/ipc/` 处理器注册表。

---

## 10. 依赖关系图

### Python 核心依赖

| 库 | 版本 | 作用 |
|---|---|---|
| **LangGraph** | 1.0.5 | 智能体工作流状态机 |
| **LangChain** | 1.2.0 | LLM 编排与链式调用 |
| **Langmem** | 0.0.30 | 智能体记忆管理 |
| **A2A SDK** | 0.3.22 | 智能体间通信协议 |
| **PySide6** | 6.10.1 | Qt6 桌面 GUI |
| **FastAPI** | 0.115.11 | Web API 与 WebSocket 服务器 |
| **Uvicorn** | 0.34.0 | ASGI 服务器 |
| **browser-use** | 0.12.5 | AI 原生浏览器控制 |
| **Playwright** | 1.54.0 | 浏览器自动化 |
| **Selenium** | 4.32.0 | 浏览器自动化（传统方式） |
| **OpenCV** | 4.11.0 | 计算机视觉（无头模式） |
| **RapidOCR** | 1.3.25 | 轻量级 OCR |
| **Pillow** | 12.1.1 | 图像处理 |
| **NumPy** | 2.4.0 | 数值计算 |
| **Pandas** | 2.3.3 | 数据分析 |

### LLM 提供商支持

| 提供商 | 包名 | 模式 |
|---|---|---|
| OpenAI | `langchain-openai` | 云端 |
| Anthropic（Claude） | `langchain-anthropic` | 云端 |
| Google | `langchain-google-*` | 云端 |
| DeepSeek | `langchain-deepseek` | 云端 |
| Ollama | `langchain-ollama` | **本地部署**（隐私模式） |
| VoyageAI | `voyageai` | Embedding 向量化 |

### 模块依赖图

```
main.py / web_server.py / ecan_cli.py
    └── app_context.py
            ├── config/app_settings.py
            ├── config/app_info.py
            ├── auth/auth_manager.py
            │       ├── auth/cognito/
            │       └── auth/oauth/
            ├── agent/ec_agent.py
            │       ├── agent/ec_skill.py
            │       │       ├── agent/ec_skills/
            │       │       └── agent/mcp/
            │       ├── agent/a2a/
            │       ├── agent/channels/
            │       ├── agent/db/
            │       └── agent/memory/
            ├── knowledge/lightrag_client.py
            │       └── knowledge/lightrag_server.py
            ├── gui/MainGUI.py（仅桌面模式）
            │       ├── gui/WebGUI.py
            │       ├── gui/LocalServer.py
            │       └── gui/ipc/
            └── utils/（logger、crash_boundary、memory_monitor 等）
```

---

## 11. 配置参考

### 环境变量

| 变量 | 取值 | 说明 |
|---|---|---|
| `ECAN_MODE` | `web`、`worker`、`desktop` | 部署模式 |
| `ECAN_WS_HOST` | IP 地址 | WebSocket 绑定地址 |
| `ECAN_WS_PORT` | 整数 | WebSocket 端口（默认 8765） |
| `ECAN_LOG_LEVEL` | `DEBUG`、`INFO`、`WARNING` | 日志详细程度 |
| `PYTHONUTF8` | `1` | 强制 UTF-8 编码 |
| `EC_DIAG` | `1` | 启用子进程诊断日志 |
| `AWS_DEFAULT_REGION` | 区域字符串 | 云服务 AWS 区域 |
| `DISPLAY` | `:99` | 无头工作节点的 X11 显示 |

### 各平台用户数据路径

| 操作系统 | 用户数据路径 |
|---|---|
| Windows | `%LOCALAPPDATA%\eCan` |
| macOS | `~/Library/Application Support/eCan` |
| Linux | `~/.local/share/eCan` |

### 目录名常量（来自 `constants.py`）

| 常量 | 作用 |
|---|---|
| `FOLDER_DATA` | 运行时数据 |
| `FOLDER_RUNLOGS` | 应用日志 |
| `FOLDER_SETTINGS` | 用户设置 |
| `FOLDER_SKILLS` | 已保存的技能定义 |

---

## 12. 项目运行方式

### 桌面应用（开发模式）

```bash
# 安装依赖
pip install -r requirements-base.txt
pip install -r requirements-macos.txt  # 或 requirements-linux.txt / requirements-windows.txt

# 安装前端依赖并构建
cd gui_v2 && pnpm install && pnpm build && cd ..

# 启动（开发模式，支持 Vite 热重载）
python main.py
```

### Web 服务器（无头模式）

```bash
pip install -r requirements-web.txt
python web_server.py
# 或者
uvicorn web_server:app --host 0.0.0.0 --port 8765
```

### Docker（Web 模式）

```bash
# 单容器
docker build -f Dockerfile.web -t ecan-web .
docker run -p 8765:8765 ecan-web

# 含 Nginx 的完整部署
docker-compose -f docker-compose.web.yml up
```

### Docker（云工作节点）

```bash
docker build -f Dockerfile.worker -t ecan-worker .
# 工作节点使用 Xvfb 实现无头浏览器自动化
# 需要 AWS 凭证用于 SQS/S3/ECS 集成
```

### CLI 命令

```bash
pip install -e .  # 以包形式安装

ecan --help
ecan server start
ecan auth login
ecan agents list
ecan skills list
ecan tasks list
```

### 构建发行版

```bash
python build.py fast   # 开发构建（2–5 分钟，含缓存）
python build.py dev    # 调试构建（5–10 分钟）
python build.py prod   # 生产构建（15–25 分钟，LZMA 压缩）
```

---

## 13. 测试

### 测试结构

```
tests/
├── conftest.py               # 公共 Fixture 与 Pytest 配置
├── e2e/                      # 端到端工作流测试
├── integration/              # 集成测试
├── unit/                     # 单元测试
├── framework/                # 框架组件测试
├── smoke/                    # 快速冒烟测试
├── browser_automation/       # 浏览器自动化测试
├── mac_tests/                # macOS 专项测试
├── test_data/                # 测试 Fixture 与数据
└── test_results/             # 生成的测试报告
```

### 重要测试文件

| 文件 | 测试重点 |
|---|---|
| `test_product_listing_orchestrator_skill.py`（92 KB） | 编排器技能完整端到端测试 |
| `test_skill_e2e_resume.py` | 技能中断后的恢复机制 |
| `test_skill_node_flow.py` | 节点级流程执行 |
| `test_node_builder_mapping.py` | 状态映射 DSL |
| `test_real_flow.py` | 真实 LLM 流程集成测试 |

### 运行测试

```bash
# 运行全部测试
pytest

# 按类别运行
pytest tests/unit/
pytest tests/e2e/
pytest tests/integration/

# 详细输出
pytest -v --tb=short

# OTA 回归测试套件
python scripts/ota_regression_test.py
```

### Pytest 配置

`pytest.ini` 位于项目根目录，配置了测试发现规则、标记（markers）和输出格式。

---

*本文档基于 eCan.ai v0.7.0 代码库分析生成*
