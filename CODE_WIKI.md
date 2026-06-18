# eCan.ai Code Wiki

> **Version:** 0.7.0 (codebase) · **Latest Release:** v0.8.24  
> **License:** MIT  
> **Platform Support:** Windows · macOS · Linux · Web (Docker)

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Overall Architecture](#2-overall-architecture)
3. [Directory Structure](#3-directory-structure)
4. [Core Entry Points](#4-core-entry-points)
   - [main.py — Desktop GUI](#41-mainpy--desktop-gui)
   - [web_server.py — Headless Web Server](#42-web_serverpy--headless-web-server)
   - [ecan_cli.py — CLI Entry Point](#43-ecan_clipy--cli-entry-point)
5. [Module Reference](#5-module-reference)
   - [app_context.py — Global Application Context](#51-app_contextpy--global-application-context)
   - [agent/ — Core Agent Framework](#52-agent--core-agent-framework)
   - [auth/ — Authentication](#53-auth--authentication)
   - [cli/ — Command-Line Interface](#54-cli--command-line-interface)
   - [common/ — Shared Models & Services](#55-common--shared-models--services)
   - [config/ — Configuration & Settings](#56-config--configuration--settings)
   - [gui/ — Desktop Qt GUI](#57-gui--desktop-qt-gui)
   - [gui_v2/ — Web Frontend (Vue.js)](#58-gui_v2--web-frontend-vuejs)
   - [knowledge/ — RAG & LightRAG](#59-knowledge--rag--lightrag)
   - [rag_worker/ — Standalone RAG Worker](#510-rag_worker--standalone-rag-worker)
   - [ocr/ — Optical Character Recognition](#511-ocr--optical-character-recognition)
   - [e_commerce/ — E-Commerce Domain Logic](#512-e_commerce--e-commerce-domain-logic)
   - [telemetry/ — Observability](#513-telemetry--observability)
   - [utils/ — Shared Utilities](#514-utils--shared-utilities)
   - [ota/ — Over-The-Air Updates](#515-ota--over-the-air-updates)
   - [infrastructure/ — Cloud Infrastructure](#516-infrastructure--cloud-infrastructure)
   - [lambda_functions/ — AWS Lambda](#517-lambda_functions--aws-lambda)
   - [wabaileys-bridge/ — Node.js Bridge](#518-wabaileys-bridge--nodejs-bridge)
   - [build_system/ & build.py — Build Orchestration](#519-build_system--buildpy--build-orchestration)
   - [scripts/ — Deployment & Utility Scripts](#520-scripts--deployment--utility-scripts)
   - [settings/ & docs/](#521-settings--docs)
6. [Key Classes & Functions](#6-key-classes--functions)
7. [Agent & Skill System Deep Dive](#7-agent--skill-system-deep-dive)
8. [MCP Tool Integration](#8-mcp-tool-integration)
9. [IPC Architecture](#9-ipc-architecture)
10. [Dependency Map](#10-dependency-map)
11. [Configuration Reference](#11-configuration-reference)
12. [Running the Project](#12-running-the-project)
13. [Testing](#13-testing)

---

## 1. Project Overview

**eCan.ai** is an AI-native, privacy-first, cross-platform **e-commerce agent platform**. It lets users build, deploy, and orchestrate intelligent agents for automating multi-channel e-commerce workflows.

**Core value propositions:**

| Feature | Description |
|---|---|
| **Networked Agents (A2A)** | Agents communicate across hosts over LAN or WAN via the A2A protocol |
| **Multi-Agent / Multi-Task** | Parallel agent execution with scheduling and breakpoint management |
| **LangGraph Workflows** | State-machine based agent skills using the LangGraph framework |
| **Flowgram IDE** | Visual drag-and-drop workflow editor that compiles to LangGraph |
| **Browser Automation** | Playwright, Selenium, Crawl4ai, browser-use for RPA |
| **Computer Vision / OCR** | Screenshot understanding, OCR, RPA automation |
| **134 MCP Tools** | Ready-to-use tools via Model Context Protocol |
| **RAG** | Knowledge retrieval with LightRAG (vector + graph search) |
| **Privacy-First** | On-premise LLM support via Ollama; local data storage |
| **Multi-Channel Messaging** | WhatsApp, Telegram, Slack, Email, SMS, Discord, WeChat, DingTalk |
| **Multi-Deployment** | Desktop app, web server, Docker, AWS ECS |

---

## 2. Overall Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           eCan.ai Platform                              │
├────────────────────┬────────────────────────┬───────────────────────────┤
│   Desktop GUI      │     Web / Server Mode  │        CLI Mode           │
│   (main.py)        │   (web_server.py)      │   (ecan_cli.py / cli/)    │
│   PySide6 / Qt     │   FastAPI + Uvicorn    │   Click framework         │
└────────┬───────────┴──────────┬─────────────┴────────────┬──────────────┘
         │                      │                           │
         └──────────────────────▼───────────────────────────┘
                         AppContext (Singleton)
                         app_context.py
                         ┌─────────────────────────────────┐
                         │  config · logger · login        │
                         │  main_window · web_gui          │
                         │  thread_pool · main_loop        │
                         └────────────┬────────────────────┘
                                      │
         ┌────────────────────────────┼──────────────────────────┐
         │                            │                          │
         ▼                            ▼                          ▼
  ┌─────────────┐           ┌──────────────────┐       ┌────────────────┐
  │  gui/       │           │  agent/          │       │  knowledge/    │
  │  MainGUI    │◄──IPC────►│  EC_Agent        │◄─────►│  LightRAG      │
  │  WebGUI     │           │  EC_Skill        │       │  RAG client    │
  │  LocalServer│           │  LangGraph flows │       │  file extract  │
  └─────────────┘           └──────┬───────────┘       └────────────────┘
                                   │
              ┌────────────────────┼──────────────────────┐
              │                    │                       │
              ▼                    ▼                       ▼
       ┌────────────┐    ┌──────────────────┐    ┌──────────────────┐
       │  agent/mcp │    │  agent/a2a       │    │  agent/channels  │
       │  134 tools │    │  A2A protocol    │    │  WhatsApp/TG/    │
       │  MCP client│    │  discovery       │    │  Slack/Email...  │
       └────────────┘    └──────────────────┘    └──────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │  Supporting Layers                                               │
  │  auth/ (Cognito/OAuth) · config/ · utils/ · telemetry/          │
  │  ota/ (auto-updates) · e_commerce/ · ocr/ · common/             │
  └──────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │  Deployment                                                      │
  │  Desktop: PyInstaller bundle (Windows/macOS/Linux)               │
  │  Web: Dockerfile.web → uvicorn                                   │
  │  Worker: Dockerfile.worker → ECS Fargate + Xvfb                 │
  │  Cloud: lambda_functions/ + infrastructure/ (CloudFormation)     │
  └──────────────────────────────────────────────────────────────────┘
```

### Dual UI Strategy

The platform maintains two parallel UI implementations that share the same backend:

- **`gui/`** — PySide6/Qt native desktop GUI. Rich OS integration (system tray, file dialogs, native menus). Used for the packaged desktop app.
- **`gui_v2/`** — Vue.js + TypeScript web frontend. Served via Vite in dev or as static files in production. Embedded in the Qt `QWebEngineView` for the desktop app and served directly for the web mode.

The `gui/ipc/` layer is the bridge that connects both frontends to the Python backend via WebSocket-based message passing.

---

## 3. Directory Structure

```
eCan.ai/
├── main.py                    # Desktop app entry point (1,120 lines)
├── web_server.py              # Web/headless server entry point
├── ecan_cli.py                # CLI entry point wrapper
├── app_context.py             # Global application context singleton (380 lines)
│
├── agent/                     # Core agent framework (23 subdirectories)
│   ├── ec_agent.py            #   EC_Agent class
│   ├── ec_skill.py            #   EC_Skill (LangGraph wrapper)
│   ├── ec_org_ctrl.py         #   Organization controller
│   ├── ec_tasks/              #   Task management & persistence
│   ├── ec_agents/             #   Pre-built agent templates
│   ├── ec_skills/             #   Skill library (26 subdirs)
│   │   ├── build_agent_skills.py
│   │   ├── build_node.py
│   │   ├── flowgram2langgraph.py  # Visual workflow → LangGraph
│   │   ├── langgraph2flowgram.py  # LangGraph → visual workflow
│   │   ├── llm_hooks/
│   │   ├── browser_use_extension/
│   │   ├── browser_node/
│   │   ├── ec_customer_support/
│   │   ├── ecbot_rpa/
│   │   └── knowledge_builder/
│   ├── a2a/                   #   Agent-to-Agent protocol
│   │   ├── discovery/         #     LAN/cloud agent discovery
│   │   ├── specification/     #     A2A protocol specs
│   │   └── langgraph_agent/   #     LangGraph A2A adapter
│   ├── mcp/                   #   Model Context Protocol (134 tools)
│   │   ├── server/
│   │   ├── client/
│   │   └── local_client.py
│   ├── playwright/            #   Playwright browser setup
│   ├── cloud_worker/          #   AWS cloud worker
│   ├── chats/                 #   Unified messenger integration
│   ├── channels/              #   Multi-channel communication drivers
│   ├── db/                    #   Database models & services
│   ├── memory/                #   Agent memory management
│   ├── avatar/                #   Avatar/profile management
│   ├── business/              #   Business logic
│   ├── prompts/               #   Prompt templates & variables
│   └── human_chatter.py       #   Human-agent chat utilities
│
├── auth/                      # Authentication (Cognito, OAuth)
│   ├── auth_manager.py        #   Main auth controller (63 KB)
│   ├── auth_config.py
│   ├── auth_config.yml
│   ├── aws_credentials_provider.py
│   ├── cognito/
│   └── oauth/
│       └── local_oauth_server.py
│
├── cli/                       # Click-based CLI (15 subdirectories)
│   ├── main.py                #   Root command group
│   ├── base/                  #   Context, output, decorators
│   ├── auth/                  #   login, logout commands
│   ├── server/                #   start, stop, logs
│   ├── agents/                #   list, get, create, update
│   ├── skills/                #   skill operations
│   ├── tasks/                 #   task management
│   ├── vehicles/              #   host management
│   ├── tools/                 #   tool querying
│   ├── knowledge/             #   knowledge base ops
│   ├── prompts/               #   prompt management
│   ├── dev/                   #   development commands
│   ├── data/                  #   data import/export
│   └── config/                #   configuration commands
│
├── common/                    # Shared models & services
│   ├── db_init.py
│   ├── models/
│   └── services/
│
├── config/                    # Configuration & app settings
│   ├── app_info.py            #   AppInfo singleton (paths, version)
│   ├── app_settings.py        #   AppSettings (web mode, dirs)
│   ├── build_info.py          #   Build metadata & banner
│   ├── constants.py           #   APP_NAME, timeouts, folder names
│   └── envi.py                #   Environment variable helpers
│
├── gui/                       # Desktop Qt GUI (11 subdirectories)
│   ├── MainGUI.py             #   Main window (299 KB)
│   ├── WebGUI.py              #   Web-based GUI (88 KB)
│   ├── LoginoutGUI.py         #   Login dialog
│   ├── LocalServer.py         #   Local HTTP server (75 KB)
│   ├── async_preloader.py     #   Background module preloading
│   ├── log_viewer.py          #   Log display widget
│   ├── menu_manager.py        #   Menu bar management
│   ├── messages.py            #   UI message strings
│   ├── config/
│   ├── context/               #   Session & state management
│   ├── core/
│   ├── dialogs/
│   ├── ipc/                   #   Inter-process communication layer
│   │   ├── registry.py
│   │   ├── handlers.py
│   │   ├── context_handlers.py
│   │   ├── w2p_handlers.py    #     Web-to-Python bridges
│   │   └── callable/
│   ├── manager/
│   ├── tool/
│   └── webdriver/
│
├── gui_v2/                    # Modern web frontend (Vue.js + TypeScript)
│   ├── src/
│   │   ├── modules/           #   Feature modules
│   │   │   ├── skill-editor/  #     Flowgram visual workflow editor
│   │   │   ├── agent-management/
│   │   │   ├── task-management/
│   │   │   └── knowledge-base/
│   │   ├── stores/            #   Pinia state management
│   │   ├── router/            #   Vue Router
│   │   └── components/        #   Reusable components
│   ├── dist/                  #   Built production frontend
│   ├── vite.config.ts
│   ├── tsconfig.json
│   └── package.json
│
├── knowledge/                 # RAG with LightRAG (16 files, ~380 KB)
│   ├── lightrag_client.py     #   HTTP client for LightRAG API
│   ├── lightrag_server.py     #   Standalone LightRAG server
│   ├── lightrag_launcher.py   #   Process management & startup
│   ├── lightrag_health.py     #   Health checks & monitoring
│   ├── lightrag_direct.py     #   Direct (non-HTTP) API access
│   ├── advanced_chunker.py    #   Smart document chunking
│   ├── file_extractors.py     #   PDF/Word/text extraction
│   ├── lightrag_config_manager.py
│   ├── lightrag_confidence_scorer.py
│   ├── provider_limits_validator.py
│   ├── rerank_score_normalizer.py
│   └── table_preprocessor.py
│
├── rag_worker/                # Docker-based RAG worker
├── e_commerce/                # E-commerce domain logic
│   └── inventories.py
├── ocr/                       # OCR configuration
│   └── ocr_app_cfg.json
├── telemetry/                 # Telemetry service
│   ├── service.py
│   └── views.py
├── utils/                     # Shared utility modules (25+)
├── ota/                       # Over-The-Air update system (10 subdirs)
├── infrastructure/            # CloudFormation, ECS, IAM
├── lambda_functions/          # AWS Lambda handlers
├── lambda_layers/             # Lambda layer packages
├── wabaileys-bridge/          # Node.js IPC bridge
├── build_system/              # Unified build system
├── scripts/                   # Deployment & utility scripts (30+)
├── settings/                  # App settings templates
├── tests/                     # Test suite (11 subdirectories)
├── docs/                      # Documentation (55+ markdown files)
├── data/                      # Runtime data directory
├── resource/                  # Static resources (icons, fonts)
├── packages/                  # Python package distributions
│
├── Dockerfile.web             # Web server container
├── Dockerfile.worker          # Cloud/automation worker container
├── docker-compose.web.yml     # Multi-service web deployment
├── nginx.conf                 # Reverse proxy config
├── requirements-base.txt      # Core Python dependencies
├── requirements-web.txt       # Web-mode dependencies (no Qt)
├── requirements-worker.txt    # Worker container dependencies
├── requirements-linux.txt     # Linux extras
├── requirements-macos.txt     # macOS extras
├── requirements-windows.txt   # Windows extras
├── mcp_tools_schema.json      # 134 MCP tool definitions (202 KB)
├── schema_03_15.graphql       # AWS AppSync GraphQL schema
├── pytest.ini                 # Pytest configuration
├── VERSION                    # Current version string
├── CHANGELOG.md               # Release notes
└── README.md                  # User guide & quickstart
```

---

## 4. Core Entry Points

### 4.1 `main.py` — Desktop GUI

**Size:** ~1,120 lines · **Purpose:** Launches the full desktop application with Qt GUI.

**Startup sequence:**

```
main.py
 │
 ├── 1. Critical patches (lines 11–149)
 │      ├── LangGraph checkpoint serializer → ormsgpack recursion limit fix
 │      ├── browser-use UTF-8 encoding fix (Windows GBK issue)
 │      ├── Platform version detection patch (hide console windows)
 │      └── FFI/Pydantic deprecation warning suppression
 │
 ├── 2. Windows compatibility (lines 215–371)
 │      ├── ProcessPoolExecutor helper process detection
 │      ├── Multiprocessing bootstrap (spawn vs fork)
 │      ├── Diagnostic subprocess monitoring (EC_DIAG=1)
 │      └── Window enumeration logging for debugging
 │
 └── 3. Startup flow (lines 538–1073)
        ├── Platform detection (Linux display server check)
        ├── QApplication creation + single-instance guard
        ├── Minimal splash screen (instant visual feedback)
        ├── Full splash + async preload begins
        ├── Config, Logger, AppContext, AppSettings initialization
        ├── qasync event loop setup
        ├── Login system initialization
        ├── WebGUI instance creation
        ├── Local server startup
        ├── Proxy environment (background thread)
        ├── OTA updater background check
        ├── URL scheme handler registration
        └── Qt event loop execution
```

**Cleanup on exit:** stops OTA updater, releases OS sleep inhibitor, unregisters LAN/cloud services.

---

### 4.2 `web_server.py` — Headless Web Server

**Purpose:** Run eCan.ai as a multi-user WebSocket server without any Qt GUI dependency.

**Key functions:**

| Function | Description |
|---|---|
| `setup_logging()` | Configures the unified logger |
| `load_handlers()` | Registers IPC handlers from `gui.ipc` modules |
| `setup_session_manager()` | Initializes `SessionManager` singleton |

**Environment variables:**

| Variable | Default | Description |
|---|---|---|
| `ECAN_MODE` | `web` (auto-set) | Selects headless mode |
| `ECAN_WS_HOST` | `0.0.0.0` | WebSocket bind host |
| `ECAN_WS_PORT` | `8765` | WebSocket port |
| `ECAN_LOG_LEVEL` | `INFO` | Log verbosity |

---

### 4.3 `ecan_cli.py` — CLI Entry Point

A minimal 30-line wrapper that:
1. Sets `PYTHONUTF8=1` (enforce UTF-8 everywhere)
2. Sets `ECAN_MODE=web` (headless mode for CLI)
3. Delegates to `cli.main.main()`

---

## 5. Module Reference

### 5.1 `app_context.py` — Global Application Context

**Size:** 380 lines · **Pattern:** Singleton via `AppContextMeta` metaclass.

**Core attributes:**

```python
class AppContext:
    app: QApplication            # Qt application instance
    main_window: MainWindow      # Desktop GUI window
    web_gui: WebGUI              # Web-based GUI wrapper
    logger: Logger               # Unified logger
    config: AppSettings          # Configuration object
    thread_pool: QThreadPool     # Thread pool for async tasks
    app_info: AppInfo            # Application metadata (paths, version)
    main_loop: AbstractEventLoop # asyncio event loop
    login: Login                 # Login controller
    playwright_browsers_path: str
```

**RunContext lazy-load namespaces:**

| Namespace | Loaded objects |
|---|---|
| `core()` | `EC_Skill`, `WorkFlowContext`, `node_builder`, `BreakpointManager` |
| `graph()` | LangGraph `StateGraph`, `END`, `Runtime`, `GraphInterrupt` |
| `llm()` | LLM hooks, async utils, JSON parsing |
| `mcp()` | MCP tool-calling client |
| `log()` | Logger, traceback helpers |

**Thread-safe accessors:**

```python
AppContext.get_instance()       # Singleton accessor
AppContext.safe_get_login()     # Validates login availability
AppContext.safe_get_main_window()
AppContext.safe_get_web_gui()
```

---

### 5.2 `agent/` — Core Agent Framework

The heart of the platform. Contains all AI agent logic, skill definitions, communication, and automation.

#### `agent/ec_agent.py` — EC_Agent

Extends browser-use `Agent`. Manages tasks (`TaskRunner`, `ManagedTask`), integrates skill lists (`EC_Skill`), supports A2A protocol messaging, and handles cloud vs local deployment.

Key attributes: `skills: List[EC_Skill]`, `rank: str`, `cloud_id: str`

#### `agent/ec_skill.py` — EC_Skill

The primary skill abstraction. Wraps a LangGraph `StateGraph` with metadata and tool access.

```python
class EC_Skill(AgentSkill):
    id: str                           # Unique skill ID
    cloud_id: str                     # Cloud UUID
    work_flow: StateGraph             # LangGraph workflow definition
    diagram: dict                     # Flowgram visual representation
    runnable: CompiledStateGraph      # Compiled, executable workflow
    mcp_client: MultiServerMCPClient  # MCP tools access
    name: str
    description: str
    run_mode: str = "released"        # "development" | "released"
    source: str = "ui"               # "code" | "example" | "ui"
    mapping_rules: dict               # Resume/state mapping DSL
    version: str = "0.0.0"
    objectives: List[str]
    need_inputs: List[dict]
    tags, examples, inputModes, outputModes
```

**State Mapping DSL** (`mapping_rules`): declarative rules that map event data fields to skill state fields, supporting:
- Field path mappings: `event.data.X` → `state.attributes.Y`
- Conflict resolution: `merge_deep`, `overwrite`
- Transform functions: `to_string`
- Apply order: `top_down`

#### `agent/ec_skills/` — Skill Library

| File/Dir | Purpose |
|---|---|
| `build_agent_skills.py` | Skill builder factory — instantiates skills from definitions |
| `build_node.py` | Node construction utilities for LangGraph |
| `flowgram2langgraph.py` | **Compiles** Flowgram visual diagrams → LangGraph `StateGraph` |
| `langgraph2flowgram.py` | **Reverse-translates** LangGraph → Flowgram diagrams |
| `llm_hooks/` | LLM integration hooks (pre/post processing) |
| `browser_use_extension/` | Extensions to browser-use library |
| `browser_node/` | Browser action node implementations |
| `ec_customer_support/` | Customer service skill templates |
| `ecbot_rpa/` | RPA supervisor & operator skill implementations |
| `knowledge_builder/` | Skills for building knowledge graphs |

#### `agent/a2a/` — Agent-to-Agent Protocol

Implements the **A2A protocol** for inter-agent communication across hosts:

- `discovery/` — Discovers agents on LAN and via cloud registry
- `specification/` — A2A protocol message format definitions
- `langgraph_agent/` — Adapts LangGraph agents for A2A communication

#### `agent/mcp/` — Model Context Protocol

Provides access to 134 pre-built tools via MCP:

- `server/` — MCP tool server implementations
- `client/` — Multi-server MCP client wrapper
- `local_client.py` — Direct (in-process) MCP tool invocation

#### `agent/channels/` — Multi-Channel Messaging

Drivers for each communication channel:
WhatsApp · Telegram · Slack · Email · SMS · Discord · WeChat · DingTalk

#### `agent/db/` — Database Layer

SQLAlchemy models and service classes for agent-related data persistence.

#### `agent/memory/` — Agent Memory

Manages short-term and long-term memory for agents (via Langmem integration).

---

### 5.3 `auth/` — Authentication

**`AuthManager`** (`auth_manager.py`, 63 KB) is the central authentication controller:

```python
class AuthManager:
    cognito_service: CognitoService  # AWS Cognito provider
    tokens: Dict                      # JWT token storage
    current_user: str
    user_profile: Dict               # Name, picture, etc.
    signed_in: bool
    machine_role: str = "Platoon"    # Agent hierarchy role
    acct_file: str                   # uli.json (persisted tokens)
    _keychain_available: bool        # macOS/Windows keychain
```

**Features:** AWS Cognito JWT auth, OAuth/OIDC with local callback server, token persistence via `uli.json`, OS keychain integration, multi-machine role support (Platoon/Commander hierarchy).

---

### 5.4 `cli/` — Command-Line Interface

Built with **Click**. The root group is defined in `cli/main.py`.

**Global flags:** `--json` · `--quiet` · `--verbose` · `--no-color`

**Command categories:**

| Category | Commands |
|---|---|
| **QUERY** | `agents list`, `skills list`, `tasks list`, `tools list`, `knowledge search` |
| **CONTROL** | `server start/stop`, `auth login/logout`, `agents start/stop` |
| **OPERATION** | `agents create/update`, `skills add/remove`, `knowledge add` |

**`cli/base/`** provides shared infrastructure:
- `context.py` — CLI context management
- `output.py` — `OutputLevel`, formatted output helpers
- `decorators.py` — Reusable Click decorators
- `config.py` — Configuration loading for CLI

---

### 5.5 `common/` — Shared Models & Services

```
common/
├── db_init.py       # Database connection & schema initialization
├── models/          # Pydantic and SQLAlchemy model definitions
└── services/        # Business logic services shared across modules
```

---

### 5.6 `config/` — Configuration & Settings

#### `AppInfo` (`config/app_info.py`, 154 lines)

Singleton holding runtime path information:

```python
class AppInfo:
    app_home_path: str       # Root dir (dev cwd or PyInstaller _MEIPASS)
    app_resources_path: str  # resource/ directory
    appdata_path: str        # OS user data dir:
                             #   Windows: %LOCALAPPDATA%/eCan
                             #   macOS:   ~/Library/Application Support/eCan
                             #   Linux:   ~/.local/share/eCan
    appdata_temp_path: str   # Temporary directory
    version: str             # Read from VERSION file
```

#### `AppSettings` (`config/app_settings.py`, 173 lines)

Runtime configuration for GUI mode:

```python
class AppSettings:
    root_dir: Path           # Project root (auto-detected)
    gui_v2_dir: Path         # gui_v2/ directory
    dist_dir: Path           # gui_v2/dist (built frontend)
    web_mode: str            # "dev" | "prod"
    vite_dev_server: str     # "http://localhost:3000"

    @property
    def is_dev_mode(self) -> bool  # web_mode == "dev"
    @property
    def is_prod_mode(self) -> bool
    def get_web_url(self) -> str   # Vite dev server or file:// URL
```

#### `constants.py` — Key constants

| Constant | Value |
|---|---|
| `APP_NAME` | `"eCan"` |
| `DEFAULT_API_TIMEOUT` | `60.0` seconds |
| `EXTENDED_API_TIMEOUT` | `120.0` seconds |
| `FOLDER_DATA` | Data directory name |
| `FOLDER_RUNLOGS` | Log directory name |
| `FOLDER_SKILLS` | Skills directory name |

---

### 5.7 `gui/` — Desktop Qt GUI

Built with **PySide6** (Qt6). The largest module in the codebase.

| File | Size | Purpose |
|---|---|---|
| `MainGUI.py` | 299 KB | Main application window |
| `WebGUI.py` | 88 KB | `QWebEngineView` wrapper hosting `gui_v2` |
| `LocalServer.py` | 75 KB | Local HTTP/WS server for IPC |
| `menu_manager.py` | 75 KB | Menu bar & system tray management |
| `LoginoutGUI.py` | 31 KB | Login/logout dialog |
| `log_viewer.py` | 38 KB | Real-time log display widget |
| `async_preloader.py` | 17 KB | Background preloading of heavy modules |
| `lightrag_rerank_proxy.py` | 36 KB | RAG reranking proxy UI |

**`gui/ipc/`** — IPC layer connecting frontend to Python backend:

| File | Purpose |
|---|---|
| `registry.py` | Handler registration & dispatch table |
| `handlers.py` | Core message handlers |
| `context_handlers.py` | Context-aware handlers |
| `w2p_handlers.py` | Web-to-Python bridge handlers |
| `callable/` | Decorator utilities for callable handlers |

---

### 5.8 `gui_v2/` — Web Frontend (Vue.js)

Modern TypeScript/Vue.js SPA. In desktop mode it is embedded in `QWebEngineView`; in web mode it is served as static files.

**Tech stack:** Vue 3 · TypeScript · Pinia (state) · Vue Router · Vite

**Key feature modules (`src/modules/`):**

| Module | Description |
|---|---|
| `skill-editor/` | **Flowgram IDE** — visual drag-and-drop workflow builder |
| `agent-management/` | Agent list, configuration, status |
| `task-management/` | Task scheduling, monitoring, history |
| `knowledge-base/` | Document upload, RAG search, management |

**Build output:** `gui_v2/dist/` — served in production mode.

---

### 5.9 `knowledge/` — RAG & LightRAG

Graph-based Retrieval-Augmented Generation using **LightRAG**.

| File | Purpose |
|---|---|
| `lightrag_client.py` (59 KB) | HTTP client — calls a remote LightRAG server |
| `lightrag_server.py` (70 KB) | Standalone LightRAG server (FastAPI) |
| `lightrag_launcher.py` (21 KB) | Process manager — starts/stops the server subprocess |
| `lightrag_health.py` (23 KB) | Health checks, liveness monitoring |
| `lightrag_direct.py` (9 KB) | Direct in-process API (bypasses HTTP) |
| `advanced_chunker.py` (32 KB) | Smart document chunking strategies |
| `file_extractors.py` (14 KB) | Extracts text from PDF, Word, plain text |
| `lightrag_config_manager.py` (28 KB) | Configuration persistence |
| `lightrag_confidence_scorer.py` (28 KB) | Relevance confidence scoring |
| `provider_limits_validator.py` (10 KB) | LLM rate limit handling |
| `rerank_score_normalizer.py` (11 KB) | Score normalization for reranking |
| `table_preprocessor.py` (12 KB) | Structured table data extraction |
| `stop_controller.py` (5 KB) | Graceful shutdown coordination |

**RAG pipeline:**
```
Document → file_extractors → advanced_chunker → LightRAG ingest
Query    → lightrag_client → (vector + graph search) → confidence_scorer → result
```

---

### 5.10 `rag_worker/` — Standalone RAG Worker

A Docker-packaged worker for offloading RAG processing:

```
rag_worker/
├── main.py          # Worker entry point (20 KB)
├── Dockerfile       # Container definition
├── build.sh
└── requirements.txt
```

---

### 5.11 `ocr/` — Optical Character Recognition

```
ocr/
└── ocr_app_cfg.json  # OCR service configuration (engine selection, params)
```

Used by RPA nodes for screen text extraction via **RapidOCR** and **pytesseract**.

---

### 5.12 `e_commerce/` — E-Commerce Domain Logic

```
e_commerce/
└── inventories.py   # Inventory management utilities (2.4 KB)
```

---

### 5.13 `telemetry/` — Observability

```
telemetry/
├── service.py  # Telemetry event collection & reporting (2.8 KB)
└── views.py    # Telemetry data views (1.2 KB)
```

---

### 5.14 `utils/` — Shared Utilities

25+ utility modules (~376 KB total). Key modules:

| Module | Purpose |
|---|---|
| `logger_helper.py` (21 KB) | Unified logging: file rotation, levels, crash logging |
| `crash_boundary.py` (11 KB) | Crash recovery detection, heartbeat monitoring, checkpointing |
| `memory_monitor.py` (30 KB) | Real-time memory tracking, leak detection via tracemalloc |
| `app_setup_helper.py` (34 KB) | Application icon setup, desktop file config |
| `single_instance.py` (12 KB) | Prevents multiple app instances (platform-specific locks) |
| `sleep_inhibitor.py` (9 KB) | Prevents OS from sleeping during long agent tasks |
| `power_monitor.py` (7 KB) | Detects system sleep/wake events |
| `url_scheme_handler.py` (15 KB) | Custom URL scheme routing (e.g., `ecan://`) |
| `port_allocator.py` (8 KB) | Dynamic port allocation for local services |
| `subprocess_helper.py` (8 KB) | Safe subprocess management |
| `permission_helper.py` (8 KB) | OS permission management |
| `linux_permissions.py` (8 KB) | Linux dependency & capability checks |
| `lazy_import.py` (4 KB) | Deferred module importing to reduce startup time |
| `i18n_helper.py` (4 KB) | Internationalization support |
| `hot_reload.py` (3 KB) | Dev-only hot reload for GUI changes |
| `data_uri_sanitizer.py` (7 KB) | Sanitizes data URIs (security) |
| `gui_dispatch.py` (2 KB) | Thread-safe Qt GUI dispatcher |
| `win_subproc.py` | Windows subprocess patching |

**Crash boundary** monitors a heartbeat file; if the app crashes before writing a clean exit, the next launch detects the prior crash and can report diagnostics.

**Memory monitor** thresholds:
- Growth warning: > 30 MB/min
- Snapshot diff interval: 120 seconds

---

### 5.15 `ota/` — Over-The-Air Updates

```
ota/
├── core/           # Core update logic (download, verify, install)
├── config/         # Environment configs (dev/staging/prod)
├── gui/            # Update dialogs & progress UI
├── server/         # OTA server endpoints
├── scripts/        # Helper scripts
├── certificates/   # Ed25519 signing keys
└── docs/
```

**Features:**
- **Ed25519** digital signature verification on every update package
- Multi-environment support (dev, staging, production)
- S3 storage with accelerated transfer URLs
- **Sparkle**-compatible Appcast format (macOS standard)
- Background silent download; user-visible install dialog
- Graceful installation with rollback support

---

### 5.16 `infrastructure/` — Cloud Infrastructure

AWS infrastructure as code:

```
infrastructure/
├── cloudformation/  # CloudFormation stack templates
├── ecs/             # ECS task & service definitions
└── iam/             # IAM roles & policies
```

Supports ECS Fargate for scalable cloud worker deployment.

---

### 5.17 `lambda_functions/` — AWS Lambda

| Function | Purpose |
|---|---|
| `agentScheduler/` | Agent task scheduling |
| `chatter/` | Chat message routing & handling |
| `skill_editor_lambda/` | Skill editor API backend |
| `presigned_link_publisher/` | S3 presigned URL generation |
| `cloud_tester/` | Cloud integration testing |
| `myAPIKeygen.zip` | API key generation |

**AppSync integration:** resolvers documented in `lambda_functions/resolvers.md`; schema in `scripts/appsync_schema_latest.graphql`.

---

### 5.18 `wabaileys-bridge/` — Node.js Bridge

A Node.js component (~14 KB `index.js`) that acts as an IPC bridge. Used for WhatsApp integration and potentially other Node.js-native communication protocols.

---

### 5.19 `build_system/` & `build.py` — Build Orchestration

**`build.py`** is the user-facing build wrapper. Three modes:

| Command | Mode | Description | Est. Time |
|---|---|---|---|
| `python build.py fast` | Development | Incremental caching, skip optimization | 2–5 min |
| `python build.py dev` | Debug | Full debug build with symbols | 5–10 min |
| `python build.py prod` | Production | Fully optimized, LZMA compressed | 15–25 min |

**`build_system/`** contains:
- Parallel compilation across CPU cores
- PyInstaller spec management
- Data file collection (minimal for dev, full for prod)
- Platform-specific packaging (NSIS for Windows, DMG for macOS, AppImage for Linux)

---

### 5.20 `scripts/` — Deployment & Utility Scripts

30+ scripts for operations:

| Script | Purpose |
|---|---|
| `deploy-ubuntu.sh` | Full Ubuntu server deployment (9.8 KB) |
| `deploy-all.sh` | Deploy all services |
| `deploy-worker.sh` | Deploy cloud worker |
| `deploy-cloudformation.sh` | AWS CloudFormation deployment |
| `build-and-push-worker.sh` | Build & push Docker worker image |
| `ota_e2e_simulation.py` | End-to-end OTA update simulation (20 KB) |
| `ota_regression_test.py` | OTA regression test suite (24 KB) |
| `prompt_files2db.sh` | Initialize prompt database |
| `create_macos_icns.sh` | Create macOS app icon |
| `rds_ddl/` | RDS table DDL definitions |
| `appsync_vtl/` | AppSync VTL resolver templates |

---

### 5.21 `settings/` & `docs/`

**`settings/`** contains default configuration templates (e.g., `role.json` with default machine role).

**`docs/`** — 55+ markdown files (~660 KB), covering:
- Architecture deep-dives: `ipc-design.md`, `build-architecture.md`, `EVENT_DRIVEN_CHAT_ARCHITECTURE.md`
- Feature guides: `BROWSER_AUTOMATION_NODE_CONFIG.md`, `eCanMCP.md`, `AVATAR_GUIDE.md`
- Operations: `DEPLOYMENT_UBUNTU.md`, `WEB_DEPLOYMENT.md`, `OTA_PATH_STRUCTURE.md`, `RELEASE_GUIDE.md`
- Technical reference: `mapping-dsl.md`, `prompt-variable-resolution.md`, `MEMORY_MONITOR.md`
- Chinese localization: `docs/cn/`
- Full user manual: `docs/user_manual.html` (57 KB)

---

## 6. Key Classes & Functions

| Class / Function | Location | Purpose |
|---|---|---|
| `AppContext` | `app_context.py` | Global singleton holding all runtime references |
| `AppInfo` | `config/app_info.py` | OS paths, version info |
| `AppSettings` | `config/app_settings.py` | Runtime configuration (dev/prod mode, URLs) |
| `EC_Agent` | `agent/ec_agent.py` | Top-level AI agent; orchestrates skills & tasks |
| `EC_Skill` | `agent/ec_skill.py` | LangGraph workflow + metadata + MCP tools |
| `AuthManager` | `auth/auth_manager.py` | Cognito/OAuth authentication controller |
| `MainGUI` | `gui/MainGUI.py` | Primary Qt desktop window |
| `WebGUI` | `gui/WebGUI.py` | `QWebEngineView` wrapper for the web frontend |
| `LocalServer` | `gui/LocalServer.py` | Local HTTP/WS server for GUI↔backend IPC |
| `flowgram2langgraph()` | `agent/ec_skills/flowgram2langgraph.py` | Compiles visual diagrams → LangGraph |
| `langgraph2flowgram()` | `agent/ec_skills/langgraph2flowgram.py` | Decompiles LangGraph → visual diagram |
| `LightRAGClient` | `knowledge/lightrag_client.py` | HTTP client for the LightRAG RAG server |
| `AdvancedChunker` | `knowledge/advanced_chunker.py` | Smart chunking for document ingestion |
| `ConfidenceScorer` | `knowledge/lightrag_confidence_scorer.py` | Relevance scoring for RAG results |
| `CrashBoundary` | `utils/crash_boundary.py` | Detects prior crashes via heartbeat files |
| `MemoryMonitor` | `utils/memory_monitor.py` | Tracks memory usage; warns on leaks |
| `SingleInstance` | `utils/single_instance.py` | Prevents duplicate app processes |
| `PortAllocator` | `utils/port_allocator.py` | Dynamically allocates local ports |
| `OTAUpdater` | `ota/core/` | Downloads, verifies (Ed25519), installs updates |

---

## 7. Agent & Skill System Deep Dive

### Skill Lifecycle

```
User (Flowgram IDE)
      │
      ▼ Drag-and-drop nodes, connect edges
  gui_v2/modules/skill-editor/
      │
      ▼ Serialize to diagram JSON (Flowgram format)
  EC_Skill.diagram: dict
      │
      ▼ flowgram2langgraph.py
  EC_Skill.work_flow: StateGraph  (LangGraph)
      │
      ▼ StateGraph.compile()
  EC_Skill.runnable: CompiledStateGraph
      │
      ▼ EC_Agent.run_skill(skill, inputs)
  LangGraph execution (node-by-node state transitions)
      │
      ▼ MCP tools called via EC_Skill.mcp_client
  Results → state
      │
      ▼ mapping_rules applied (DSL)
  Final output returned
```

### Agent Hierarchy & A2A

```
Commander Agent (high-level planning)
      │  A2A protocol
      ▼
Platoon Agents (mid-level coordination)
      │  A2A protocol
      ▼
Operator Agents (task execution: browser, RPA, messaging)
```

Agents discover each other via:
1. **LAN discovery** — mDNS/broadcast within local network
2. **Cloud registry** — central service (AWS AppSync/Lambda)

### Run Modes

| Mode | Description |
|---|---|
| `released` | Production mode; strict state validation |
| `development` | Permissive mode; looser validation for testing |

---

## 8. MCP Tool Integration

The `mcp_tools_schema.json` file (202 KB) defines **134 tools** organized by category:

| Category | Examples |
|---|---|
| **RPA** | `rpa_supervisor_scheduling_work`, `rpa_operator_dispatch_works` |
| **Browser** | Navigate, click, extract, screenshot |
| **Knowledge** | Search, ingest, query knowledge base |
| **Data** | Parse, transform, validate structured data |
| **Communication** | Send messages across channels |
| **E-Commerce** | Product listing, inventory, order management |

Tools are served via the MCP protocol from `agent/mcp/server/` and accessed by skills via `EC_Skill.mcp_client` (a `MultiServerMCPClient`).

The `agent/mcp/local_client.py` provides direct in-process tool invocation for performance-critical paths.

---

## 9. IPC Architecture

Communication between the Vue.js frontend and Python backend follows an event-driven WebSocket pattern:

```
gui_v2 (Vue.js)
    │
    │  WebSocket messages (JSON)
    ▼
gui/LocalServer.py  (WebSocket server on localhost)
    │
    │  dispatch by message type
    ▼
gui/ipc/registry.py  (handler dispatch table)
    │
    ├── handlers.py          (core operations)
    ├── context_handlers.py  (context-aware)
    └── w2p_handlers.py      (web-to-python bridges)
         │
         ▼
    Python backend (agent/, knowledge/, auth/, etc.)
```

In web deployment mode, `web_server.py` replaces `LocalServer.py` as the WebSocket endpoint, but the same `gui/ipc/` handler registry is reused.

---

## 10. Dependency Map

### Python Core Dependencies

| Library | Version | Role |
|---|---|---|
| **LangGraph** | 1.0.5 | Agent workflow state machine |
| **LangChain** | 1.2.0 | LLM orchestration & chains |
| **Langmem** | 0.0.30 | Agent memory management |
| **A2A SDK** | 0.3.22 | Agent-to-Agent protocol |
| **PySide6** | 6.10.1 | Qt6 desktop GUI |
| **FastAPI** | 0.115.11 | Web API & WebSocket server |
| **Uvicorn** | 0.34.0 | ASGI server |
| **browser-use** | 0.12.5 | AI-native browser control |
| **Playwright** | 1.54.0 | Browser automation |
| **Selenium** | 4.32.0 | Browser automation (legacy) |
| **OpenCV** | 4.11.0 | Computer vision (headless) |
| **RapidOCR** | 1.3.25 | Lightweight OCR |
| **Pillow** | 12.1.1 | Image processing |
| **NumPy** | 2.4.0 | Numerical computing |
| **Pandas** | 2.3.3 | Data analysis |

### LLM Provider Support

| Provider | Package | Mode |
|---|---|---|
| OpenAI | `langchain-openai` | Cloud |
| Anthropic | `langchain-anthropic` | Cloud |
| Google | `langchain-google-*` | Cloud |
| DeepSeek | `langchain-deepseek` | Cloud |
| Ollama | `langchain-ollama` | **On-premise** (privacy mode) |
| VoyageAI | `voyageai` | Embeddings |

### Module Dependency Graph

```
main.py / web_server.py / ecan_cli.py
    └── app_context.py
            ├── config/app_settings.py
            ├── config/app_info.py
            ├── auth/auth_manager.py
            │       └── auth/cognito/
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
            ├── gui/MainGUI.py  (desktop only)
            │       ├── gui/WebGUI.py
            │       ├── gui/LocalServer.py
            │       └── gui/ipc/
            └── utils/ (logger, crash_boundary, memory_monitor, ...)
```

---

## 11. Configuration Reference

### Environment Variables

| Variable | Values | Description |
|---|---|---|
| `ECAN_MODE` | `web`, `worker`, `desktop` | Deployment mode |
| `ECAN_WS_HOST` | IP address | WebSocket bind host |
| `ECAN_WS_PORT` | Integer | WebSocket port (default 8765) |
| `ECAN_LOG_LEVEL` | `DEBUG`, `INFO`, `WARNING` | Log verbosity |
| `PYTHONUTF8` | `1` | Force UTF-8 encoding |
| `EC_DIAG` | `1` | Enable diagnostic subprocess logging |
| `AWS_DEFAULT_REGION` | Region string | AWS region for cloud services |
| `DISPLAY` | `:99` | X11 display for headless worker |

### Platform-Specific Data Paths

| OS | User Data Path |
|---|---|
| Windows | `%LOCALAPPDATA%\eCan` |
| macOS | `~/Library/Application Support/eCan` |
| Linux | `~/.local/share/eCan` |

### Folder Names (from `constants.py`)

| Constant | Purpose |
|---|---|
| `FOLDER_DATA` | Runtime data |
| `FOLDER_RUNLOGS` | Application logs |
| `FOLDER_SETTINGS` | User settings |
| `FOLDER_SKILLS` | Saved skill definitions |

---

## 12. Running the Project

### Desktop App (Development)

```bash
# Install dependencies
pip install -r requirements-base.txt
pip install -r requirements-macos.txt  # or requirements-linux.txt / requirements-windows.txt

# Install frontend dependencies
cd gui_v2 && pnpm install && pnpm build && cd ..

# Launch (dev mode with Vite hot-reload)
python main.py
```

### Web Server (Headless)

```bash
pip install -r requirements-web.txt
python web_server.py
# or
uvicorn web_server:app --host 0.0.0.0 --port 8765
```

### Docker (Web Mode)

```bash
# Single container
docker build -f Dockerfile.web -t ecan-web .
docker run -p 8765:8765 ecan-web

# Full stack with Nginx
docker-compose -f docker-compose.web.yml up
```

### Docker (Cloud Worker)

```bash
docker build -f Dockerfile.worker -t ecan-worker .
# Worker uses Xvfb for headless browser automation
# Requires AWS credentials for SQS/S3/ECS integration
```

### CLI

```bash
pip install -e .  # Install as package

ecan --help
ecan server start
ecan auth login
ecan agents list
ecan skills list
ecan tasks list
```

### Building Distributable Packages

```bash
python build.py fast   # Development (2–5 min, with caching)
python build.py dev    # Debug build (5–10 min)
python build.py prod   # Production (15–25 min, LZMA compressed)
```

---

## 13. Testing

### Test Structure

```
tests/
├── conftest.py               # Shared fixtures & pytest config
├── e2e/                      # End-to-end workflow tests
├── integration/              # Integration tests
├── unit/                     # Unit tests
├── framework/                # Framework component tests
├── smoke/                    # Quick smoke tests
├── browser_automation/       # Browser automation tests
├── mac_tests/                # macOS-specific tests
├── test_data/                # Test fixtures & data
└── test_results/             # Generated test reports
```

### Key Test Files

| File | Focus |
|---|---|
| `test_product_listing_orchestrator_skill.py` (92 KB) | Comprehensive orchestrator skill E2E |
| `test_skill_e2e_resume.py` | Skill resumption after interruption |
| `test_skill_node_flow.py` | Node-level flow execution |
| `test_node_builder_mapping.py` | State mapping DSL |
| `test_real_flow.py` | Real LLM flow integration |

### Running Tests

```bash
# All tests
pytest

# Specific category
pytest tests/unit/
pytest tests/e2e/
pytest tests/integration/

# With output
pytest -v --tb=short

# OTA regression suite
python scripts/ota_regression_test.py
```

### Pytest Configuration (`pytest.ini`)

Located at the project root. Configures test discovery, markers, and output format.

---

*Generated from codebase analysis of eCan.ai v0.7.0*
