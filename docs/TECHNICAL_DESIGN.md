# Hermes-Agent 技术设计文档

> 全面的系统架构、本体设计、时序分析与算法设计文档

---

## 目录

1. [系统概览](#1-系统概览)
2. [本体设计（Ontology）](#2-本体设计ontology)
   - 2.1 [核心实体与类层次](#21-核心实体与类层次)
   - 2.2 [实体关系图](#22-实体关系图)
   - 2.3 [数据模型](#23-数据模型)
3. [时序设计（Sequence）](#3-时序设计sequence)
   - 3.1 [CLI 会话主流程](#31-cli-会话主流程)
   - 3.2 [Gateway 消息处理流程](#32-gateway-消息处理流程)
   - 3.3 [Agent 对话循环（核心循环）](#33-agent-对话循环核心循环)
   - 3.4 [工具调用执行流程](#34-工具调用执行流程)
   - 3.5 [上下文压缩流程](#35-上下文压缩流程)
   - 3.6 [子代理委托流程](#36-子代理委托流程)
   - 3.7 [MCP 工具发现流程](#37-mcp-工具发现流程)
4. [算法设计](#4-算法设计)
   - 4.1 [上下文窗口压缩算法](#41-上下文窗口压缩算法)
   - 4.2 [工具发现与解析算法](#42-工具发现与解析算法)
   - 4.3 [工具集递归解析算法](#43-工具集递归解析算法)
   - 4.4 [Prompt 缓存策略](#44-prompt-缓存策略)
   - 4.5 [模型上下文长度解析算法](#45-模型上下文长度解析算法)
   - 4.6 [并行工具执行调度算法](#46-并行工具执行调度算法)
   - 4.7 [危险命令检测算法](#47-危险命令检测算法)
   - 4.8 [迭代预算管理算法](#48-迭代预算管理算法)
   - 4.9 [会话全文搜索算法（FTS5）](#49-会话全文搜索算法fts5)
   - 4.10 [代码执行沙箱机制](#410-代码执行沙箱机制)
5. [模块依赖链](#5-模块依赖链)
6. [扩展性设计](#6-扩展性设计)
7. [安全设计](#7-安全设计)

---

## 1. 系统概览

Hermes-Agent 是一个多平台 AI 代理框架，支持通过交互式 CLI、Telegram、Discord、Slack、WhatsApp、Signal、Email、SMS、Home Assistant 等多种渠道与大型语言模型（LLM）交互。系统提供丰富的工具能力（终端执行、文件操作、Web 搜索、浏览器自动化、代码执行等），并具备会话持久化、上下文压缩、提示词缓存、子代理委托等高级特性。

### 架构分层

```
┌─────────────────────────────────────────────────────────────────┐
│                      接入层 (Presentation)                       │
│  ┌──────────┐  ┌──────────────────────────────────────────────┐ │
│  │  CLI     │  │           Gateway (消息网关)                   │ │
│  │ (cli.py) │  │  ┌────────┬────────┬────────┬─────────────┐ │ │
│  │          │  │  │Telegram│Discord │ Slack  │WhatsApp/... │ │ │
│  └────┬─────┘  │  └───┬────┴───┬────┴───┬────┴──────┬──────┘ │ │
│       │        │      └────────┴────────┴───────────┘        │ │
│       │        └───────────────────┬──────────────────────────┘ │
│       │                            │                             │
├───────┼────────────────────────────┼─────────────────────────────┤
│       │      代理核心层 (Agent Core)│                             │
│       ▼                            ▼                             │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                   AIAgent (run_agent.py)                     │ │
│  │  ┌──────────────┬─────────────┬──────────────────────────┐ │ │
│  │  │PromptBuilder │ContextComp. │   PromptCaching          │ │ │
│  │  │              │             │                          │ │ │
│  │  │  系统提示组装  │ 上下文压缩   │   Anthropic缓存          │ │ │
│  │  └──────────────┴─────────────┴──────────────────────────┘ │ │
│  │  ┌──────────────┬─────────────┬──────────────────────────┐ │ │
│  │  │ModelMetadata  │AuxClient   │ UsagePricing             │ │ │
│  │  │  模型元数据    │ 辅助LLM客户端│   计费计价                │ │ │
│  │  └──────────────┴─────────────┴──────────────────────────┘ │ │
│  └────────────────────────┬────────────────────────────────────┘ │
│                            │                                     │
├────────────────────────────┼─────────────────────────────────────┤
│       工具编排层 (Tool Orchestration)                              │
│                            ▼                                     │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │               model_tools.py (工具编排器)                     │ │
│  │  ┌─────────────────────────┬────────────────────────────┐  │ │
│  │  │ _discover_tools()       │ handle_function_call()     │  │ │
│  │  │ get_tool_definitions()  │ _run_async()               │  │ │
│  │  └─────────────────────────┴────────────────────────────┘  │ │
│  └────────────────────────┬────────────────────────────────────┘ │
│                            │                                     │
│  ┌─────────────────────────┼───────────────────────────────────┐ │
│  │      toolsets.py        │         tools/registry.py         │ │
│  │   (工具集定义与解析)     │        (注册表单例)                │ │
│  └─────────────────────────┴───────────────────────────────────┘ │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│       工具实现层 (Tool Implementations)                            │
│  ┌─────────┬──────────┬──────────┬────────────┬───────────────┐ │
│  │terminal │file_tools│web_tools │browser_tool│delegate_tool  │ │
│  │         │          │          │            │               │ │
│  ├─────────┼──────────┼──────────┼────────────┼───────────────┤ │
│  │code_exec│mcp_tool  │process   │approval    │skills_tool    │ │
│  │         │          │_registry │            │               │ │
│  └─────────┴──────────┴──────────┴────────────┴───────────────┘ │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│       持久化层 (Persistence)                                      │
│  ┌──────────────────────┬──────────────────────────────────────┐ │
│  │   hermes_state.py    │     gateway/session.py               │ │
│  │   SessionDB (SQLite) │     SessionStore (JSON+SQLite)       │ │
│  │   FTS5 全文搜索       │     会话键管理 + 转录持久化            │ │
│  └──────────────────────┴──────────────────────────────────────┘ │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│       配置层 (Configuration)                                      │
│  ┌──────────────┬─────────────┬──────────────┬────────────────┐ │
│  │config.py     │auth.py      │skin_engine.py│commands.py     │ │
│  │(YAML配置)    │(认证存储)    │(主题引擎)     │(命令注册表)     │ │
│  └──────────────┴─────────────┴──────────────┴────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. 本体设计（Ontology）

### 2.1 核心实体与类层次

#### 2.1.1 AIAgent（代理核心）

```
AIAgent
├── 构造参数
│   ├── model: str                  # LLM 模型标识
│   ├── max_iterations: int         # 最大循环迭代次数
│   ├── enabled_toolsets: list      # 启用的工具集
│   ├── disabled_toolsets: list     # 禁用的工具集
│   ├── platform: str              # 平台标识 ("cli", "telegram", ...)
│   ├── session_id: str            # 会话 ID
│   ├── provider: str              # LLM 提供商
│   ├── api_key: str               # API 密钥
│   ├── base_url: str              # API 基础 URL
│   ├── api_mode: str              # API 模式 (chat_completions/anthropic_messages/codex_responses)
│   ├── fallback_model: str        # 备用模型
│   ├── save_trajectories: bool    # 保存轨迹
│   ├── quiet_mode: bool           # 静默模式
│   ├── skip_memory: bool          # 跳过记忆加载
│   ├── skip_context_files: bool   # 跳过上下文文件
│   └── callbacks                  # 各类回调函数集合
│       ├── tool_progress_callback
│       ├── thinking_callback
│       ├── reasoning_callback
│       ├── clarify_callback
│       ├── step_callback
│       ├── stream_delta_callback
│       └── status_callback
│
├── 核心组件
│   ├── client: OpenAI              # LLM API 客户端
│   ├── iteration_budget: IterationBudget  # 迭代预算管理器
│   ├── context_compressor: ContextCompressor  # 上下文压缩器
│   ├── _session_db: SessionDB      # 会话数据库
│   ├── _todo_store: TodoStore      # 任务管理存储
│   ├── _memory_store: MemoryStore  # 记忆存储
│   ├── _checkpoint_mgr             # 检查点管理器
│   └── _honcho                     # Honcho 记忆管理器
│
├── 核心方法
│   ├── chat(message) → str                    # 简单接口
│   ├── run_conversation(user_message, ...) → dict  # 完整接口
│   ├── _build_system_prompt() → str           # 构建系统提示
│   ├── _execute_tool_calls()                   # 执行工具调用
│   ├── _compress_context()                     # 触发上下文压缩
│   ├── _persist_session()                      # 持久化会话
│   └── interrupt() / clear_interrupt()         # 中断控制
│
└── 状态
    ├── tools: list                 # 工具定义列表
    ├── valid_tool_names: set       # 有效工具名集合
    ├── _cached_system_prompt: str  # 缓存的系统提示
    ├── _user_turn_count: int       # 用户轮次计数
    └── session counters            # Token / 成本计数器
```

#### 2.1.2 ToolRegistry（工具注册表）

```
ToolRegistry (单例)
├── _tools: Dict[str, ToolEntry]          # 工具名 → 工具条目映射
├── _toolset_checks: Dict[str, Callable]  # 工具集 → 可用性检查函数
│
├── register(name, toolset, schema, handler, check_fn, ...)
├── get_definitions(tool_names) → List[dict]
├── dispatch(name, args, **kwargs) → str (JSON)
├── is_toolset_available(toolset) → bool
└── get_tool_to_toolset_map() → Dict[str, str]

ToolEntry (值对象)
├── name: str           # 工具名
├── toolset: str        # 所属工具集
├── schema: dict        # OpenAI 函数调用 schema
├── handler: Callable   # 执行处理器
├── check_fn: Callable  # 可用性检查函数
├── requires_env: list  # 需要的环境变量
├── is_async: bool      # 是否异步
├── description: str    # 描述
└── emoji: str          # 显示 emoji
```

#### 2.1.3 ContextCompressor（上下文压缩器）

```
ContextCompressor
├── 配置
│   ├── model: str                      # 主模型名
│   ├── threshold_percent: float        # 触发阈值 (默认 0.50)
│   ├── protect_first_n: int            # 保护头部消息数 (默认 3)
│   ├── protect_last_n: int             # 保护尾部最小消息数 (默认 20)
│   ├── tail_token_budget: int          # 尾部 token 预算
│   ├── max_summary_tokens: int         # 摘要最大 token
│   └── summary_model: str             # 摘要模型覆盖
│
├── 状态
│   ├── compression_count: int          # 压缩次数
│   ├── _previous_summary: str          # 上次压缩摘要
│   ├── last_prompt_tokens: int         # 最近 prompt token
│   └── context_length: int             # 模型上下文窗口大小
│
├── 核心方法
│   ├── should_compress(prompt_tokens) → bool
│   ├── should_compress_preflight(messages) → bool
│   ├── compress(messages) → List[dict]
│   ├── _prune_old_tool_results(messages, protect_tail_count)
│   ├── _generate_summary(turns) → str
│   ├── _serialize_for_summary(turns) → str
│   └── _sanitize_tool_pairs(messages) → List[dict]
```

#### 2.1.4 SessionDB（会话数据库）

```
SessionDB
├── 存储: SQLite (WAL 模式)
│   ├── sessions 表      # 会话元数据
│   ├── messages 表      # 消息历史
│   └── messages_fts 表  # FTS5 全文搜索虚拟表
│
├── 会话生命周期
│   ├── create_session(session_id, source, model, ...)
│   ├── end_session(session_id, reason)
│   ├── update_system_prompt(session_id, prompt)
│   ├── update_token_counts(session_id, tokens)
│   └── set_session_title(session_id, title)
│
├── 消息操作
│   ├── append_message(session_id, role, content, ...)
│   ├── get_messages(session_id) → List
│   └── get_messages_as_conversation(session_id) → List[dict]
│
└── 搜索
    ├── search_messages(query, session_id, source, role, limit)
    └── search_sessions(query, limit)
```

#### 2.1.5 HermesCLI（CLI 控制器）

```
HermesCLI
├── 配置
│   ├── CLI_CONFIG: dict            # 合并后的配置
│   ├── model / provider / api_key  # LLM 相关
│   ├── max_turns: int              # 最大轮次
│   └── enabled_toolsets: list      # 启用的工具集
│
├── 组件
│   ├── agent: AIAgent              # 代理实例 (懒初始化)
│   ├── session_db: SessionDB       # 会话数据库
│   ├── console: ChatConsole        # Rich 控制台
│   └── conversation_history: list  # 会话历史
│
├── 核心方法
│   ├── run()                        # 启动交互循环
│   ├── chat(message, images)        # 发送消息
│   ├── _init_agent()                # 初始化/重置代理
│   ├── process_command(command)      # 处理斜杠命令
│   └── _ensure_runtime_credentials() # 确保凭证
```

#### 2.1.6 GatewayRunner（消息网关）

```
GatewayRunner
├── 配置
│   ├── config: dict                 # Gateway 配置
│   ├── adapters: Dict[Platform, BasePlatformAdapter]
│   └── session_store: SessionStore
│
├── 状态
│   ├── _running_agents: Dict[str, AIAgent]  # 活跃代理
│   ├── _pending_messages: Dict[str, Queue]  # 待处理消息队列
│   └── delivery_router                      # 消息路由器
│
├── 生命周期
│   ├── start()        # 启动所有适配器
│   ├── stop()         # 停止
│   └── wait_for_shutdown()
│
├── 消息处理
│   ├── _handle_message(event: MessageEvent)
│   ├── _handle_message_with_agent(event, session_key)
│   ├── _run_agent(event, session_key, agent, messages)
│   └── 斜杠命令处理器 (handle_*_command)
│
└── 后台服务
    ├── _session_expiry_watcher()      # 会话过期监控
    ├── _platform_reconnect_watcher()  # 平台重连监控
    └── _run_process_watcher()         # 进程监控
```

#### 2.1.7 平台适配器层次

```
BasePlatformAdapter (ABC)
├── connect() → bool
├── disconnect()
├── send_message(chat_id, text, ...) → str
├── send_typing(chat_id)
├── set_message_handler(handler)
├── format_for_platform(text) → str
│
├── TelegramAdapter
├── DiscordAdapter
├── SlackAdapter
├── WhatsAppAdapter
├── SignalAdapter
├── MatrixAdapter
├── MattermostAdapter
├── HomeAssistantAdapter
├── DingtalkAdapter
├── EmailAdapter
├── SMSAdapter
├── WebhookAdapter
└── APIServerAdapter
```

#### 2.1.8 斜杠命令系统

```
CommandDef (frozen dataclass)
├── name: str            # 规范名 (如 "model")
├── description: str     # 描述
├── category: str        # 分类 ("Session", "Configuration", ...)
├── aliases: tuple       # 别名元组
├── args_hint: str       # 参数提示
├── cli_only: bool       # 仅 CLI 可用
└── gateway_only: bool   # 仅 Gateway 可用

COMMAND_REGISTRY: List[CommandDef]
    │
    ├── → COMMANDS: dict               # 名称+别名 → 规范名
    ├── → COMMANDS_BY_CATEGORY: dict   # 分类 → 命令列表 (CLI help)
    ├── → GATEWAY_KNOWN_COMMANDS: set  # Gateway 已知命令集
    ├── → telegram_bot_commands()      # Telegram BotCommand 列表
    └── → slack_subcommand_map()       # Slack 子命令映射
```

### 2.2 实体关系图

```
                            ┌─────────────┐
                            │ CommandDef  │◄──── COMMAND_REGISTRY
                            │(斜杠命令定义) │      (中央命令注册表)
                            └──────┬──────┘
                                   │ dispatches to
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
            ┌────────────┐ ┌──────────────┐ ┌──────────────┐
            │ HermesCLI  │ │GatewayRunner │ │ BatchRunner  │
            │ (交互式CLI) │ │ (消息网关)    │ │ (批处理器)    │
            └─────┬──────┘ └──────┬───────┘ └──────┬───────┘
                  │               │                │
                  │    creates    │     creates     │  creates
                  ▼               ▼                ▼
            ┌─────────────────────────────────────────┐
            │              AIAgent                     │
            │         (AI 代理核心实体)                  │
            └──┬──────┬──────┬──────┬──────┬──────────┘
               │      │      │      │      │
    ┌──────────┘   ┌──┘   ┌──┘   ┌──┘      └──────────────┐
    ▼              ▼      ▼      ▼                         ▼
┌────────┐  ┌─────────┐┌─────────┐┌─────────┐      ┌────────────┐
│Iteration│  │Context  ││Session  ││Memory   │      │ Tool       │
│Budget   │  │Compress.││DB      ││Store    │      │ Definitions│
│(迭代预算)│  │(压缩器) ││(会话DB) ││(记忆存储)│      │ (工具定义)  │
└────────┘  └─────────┘└─────────┘└─────────┘      └──────┬─────┘
                                                          │
                                                          │ queries
                                                          ▼
                                                   ┌────────────┐
                                ┌──────────────────│model_tools │
                                │                  │ (工具编排器) │
                                │                  └──────┬─────┘
                                │                         │
                                ▼                         ▼
                         ┌────────────┐           ┌────────────┐
                         │ toolsets   │           │ToolRegistry│
                         │(工具集定义) │           │ (注册表)    │
                         └────────────┘           └──────┬─────┘
                                                         │
                              ┌───────┬───────┬──────────┼───────┬──────────┐
                              ▼       ▼       ▼          ▼       ▼          ▼
                         ┌────────┐┌───────┐┌───────┐┌───────┐┌───────┐┌───────┐
                         │terminal││file   ││web    ││browser││code   ││mcp    │
                         │_tool   ││_tools ││_tools ││_tool  ││_exec  ││_tool  │
                         └────────┘└───────┘└───────┘└───────┘└───────┘└───────┘

                    ┌──────────────┐
                    │  SkinConfig  │◄──── _BUILTIN_SKINS (内置主题)
                    │ (主题配置)    │◄──── ~/.hermes/skins/*.yaml (用户主题)
                    └──────────────┘

                    ┌──────────────┐
                    │ProviderConfig│◄──── PROVIDER_REGISTRY (提供商注册表)
                    │(提供商配置)   │
                    └──────────────┘
```

### 2.3 数据模型

#### 2.3.1 消息格式（OpenAI 兼容）

```python
# 用户消息
{"role": "user", "content": "请帮我搜索..."}

# 助手消息（含工具调用）
{
    "role": "assistant",
    "content": "",
    "reasoning": "...",          # 推理过程（可选）
    "tool_calls": [
        {
            "id": "call_abc123",
            "type": "function",
            "function": {
                "name": "web_search",
                "arguments": "{\"query\": \"...\"}"
            }
        }
    ]
}

# 工具结果消息
{
    "role": "tool",
    "tool_call_id": "call_abc123",
    "content": "{\"results\": [...]}"     # 始终为 JSON 字符串
}

# 系统消息
{"role": "system", "content": "You are Hermes..."}
```

#### 2.3.2 SQLite 数据库 Schema

```sql
-- 会话表
sessions (
    id TEXT PRIMARY KEY,
    source TEXT NOT NULL,           -- 'cli', 'telegram', 'discord', ...
    user_id TEXT,
    model TEXT,
    model_config TEXT,              -- JSON
    system_prompt TEXT,
    parent_session_id TEXT,         -- 压缩分裂时的父会话链
    started_at REAL,
    ended_at REAL,
    end_reason TEXT,
    message_count INTEGER,
    tool_call_count INTEGER,
    input_tokens / output_tokens / cache_* / reasoning_tokens INTEGER,
    estimated_cost_usd / actual_cost_usd REAL,
    title TEXT
)

-- 消息表
messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT REFERENCES sessions(id),
    role TEXT NOT NULL,
    content TEXT,
    tool_call_id TEXT,
    tool_calls TEXT,                -- JSON
    tool_name TEXT,
    timestamp REAL,
    token_count INTEGER,
    finish_reason TEXT,
    reasoning TEXT,
    reasoning_details TEXT,         -- JSON
    codex_reasoning_items TEXT      -- JSON
)

-- FTS5 全文搜索虚拟表
messages_fts (content, content=messages, content_rowid=id)
-- 配合 INSERT/UPDATE/DELETE 触发器保持同步
```

#### 2.3.3 配置数据模型（`~/.hermes/config.yaml`）

```yaml
model: "anthropic/claude-opus-4.6"
toolsets:
  enabled: []
  disabled: []
agent:
  max_turns: 90
terminal:
  env: "local"               # local | docker | ssh | modal | daytona | singularity
  timeout: 120
browser:
  provider: "local"          # local | browserbase
compression:
  enabled: true
  threshold: 0.50
  summary_model: ""
smart_model_routing:
  enabled: false
auxiliary:
  vision: { model: "", provider: "" }
  compression: { model: "", provider: "" }
display:
  skin: "default"
  compact_mode: false
  streaming: true
  tool_progress: true
  background_process_notifications: "all"
memory:
  enabled: true
  auto_memory: true
delegation:
  enabled: true
  model: ""
honcho:
  enabled: false
approvals:
  mode: "manual"            # manual | smart | off
security:
  tirith: { enabled: false }
```

---

## 3. 时序设计（Sequence）

### 3.1 CLI 会话主流程

```
用户                    HermesCLI              AIAgent              model_tools           LLM API
 │                         │                      │                     │                    │
 │  输入消息/命令           │                      │                     │                    │
 │─────────────────────►  │                      │                     │                    │
 │                         │                      │                     │                    │
 │  [如果是斜杠命令]        │                      │                     │                    │
 │                         │ process_command()    │                     │                    │
 │                         │──────────┐           │                     │                    │
 │                         │          │           │                     │                    │
 │                         │◄─────────┘           │                     │                    │
 │  显示命令结果            │                      │                     │                    │
 │◄─────────────────────  │                      │                     │                    │
 │                         │                      │                     │                    │
 │  [如果是聊天消息]        │                      │                     │                    │
 │                         │ _init_agent()        │                     │                    │
 │                         │─────────────────────►│                     │                    │
 │                         │                      │ get_tool_definitions│                    │
 │                         │                      │────────────────────►│                    │
 │                         │                      │ ◄───tools schemas───│                    │
 │                         │ ◄────agent ready──── │                     │                    │
 │                         │                      │                     │                    │
 │                         │ chat(message)        │                     │                    │
 │                         │ [后台线程]            │                     │                    │
 │                         │─────────────────────►│ run_conversation()  │                    │
 │                         │                      │                     │                    │
 │  [显示旋转动画]          │                      │────API call────────────────────────────►│
 │  ╭( ◕ω◕ )╮ thinking... │                      │                     │                    │
 │                         │                      │◄───response─────────────────────────────│
 │                         │                      │                     │                    │
 │                         │                      │ [如果有工具调用]     │                    │
 │  ┊ 🔍 web_search(...)   │◄─tool_progress_cb── │                     │                    │
 │                         │                      │─handle_function_call│                    │
 │                         │                      │────────────────────►│                    │
 │                         │                      │                     │ registry.dispatch()│
 │                         │                      │ ◄───result─────────│                    │
 │  ┊ ✓ web_search done    │◄─tool_progress_cb── │                     │                    │
 │                         │                      │                     │                    │
 │                         │                      │ [循环直到无工具调用]  │                    │
 │                         │                      │────API call────────────────────────────►│
 │                         │                      │◄───final response──────────────────────│
 │                         │ ◄──结果返回──────────│                     │                    │
 │                         │                      │                     │                    │
 │  ┌─── Response ────┐   │                      │                     │                    │
 │  │ 回答内容...       │   │                      │                     │                    │
 │  └─────────────────┘   │                      │                     │                    │
 │◄─────────────────────  │                      │                     │                    │
```

### 3.2 Gateway 消息处理流程

```
平台消息          PlatformAdapter       GatewayRunner        SessionStore       AIAgent
    │                   │                    │                    │               │
    │  收到消息           │                    │                    │               │
    │──────────────────►│                    │                    │               │
    │                   │ _handle_message()   │                    │               │
    │                   │───────────────────►│                    │               │
    │                   │                    │                    │               │
    │                   │                    │ 鉴权检查            │               │
    │                   │                    │ _is_user_authorized │               │
    │                   │                    │                    │               │
    │                   │                    │ 检查是否有活跃代理   │               │
    │                   │                    │ _running_agents    │               │
    │                   │                    │                    │               │
    │                   │                    │ [代理已在运行]      │               │
    │                   │                    │  → 中断/排队/特殊命令│               │
    │                   │                    │                    │               │
    │                   │                    │ [代理空闲]          │               │
    │                   │                    │ 解析斜杠命令         │               │
    │                   │                    │   或                │               │
    │                   │                    │ _handle_message_with_agent()        │
    │                   │                    │                    │               │
    │                   │                    │ get_or_create_session│              │
    │                   │                    │───────────────────►│               │
    │                   │                    │◄──session_entry────│               │
    │                   │                    │                    │               │
    │                   │                    │ load_transcript()  │               │
    │                   │                    │───────────────────►│               │
    │                   │                    │◄──history──────────│               │
    │                   │                    │                    │               │
    │                   │                    │ _run_agent()       │               │
    │                   │                    │──────────────────────────────────►│
    │                   │                    │                    │               │
    │                   │                    │                    │  run_conversation()
    │                   │                    │                    │  [多轮工具调用循环]
    │                   │                    │                    │               │
    │  发送打字状态       │◄─send_typing───── │                    │               │
    │◄─────────────────│                    │                    │               │
    │                   │                    │                    │               │
    │                   │                    │◄─────────final_response───────────│
    │                   │                    │                    │               │
    │                   │                    │ append_to_transcript│              │
    │                   │                    │───────────────────►│               │
    │                   │                    │                    │               │
    │  发送回复           │◄─send_message──── │                    │               │
    │◄─────────────────│                    │                    │               │
```

### 3.3 Agent 对话循环（核心循环）

```
                     run_conversation() 入口
                            │
                            ▼
                   ┌────────────────┐
                   │ 初始化阶段       │
                   │ • 创建 task_id   │
                   │ • 重置预算       │
                   │ • 复制历史       │
                   │ • Honcho 预取   │
                   └────────┬───────┘
                            │
                            ▼
                   ┌────────────────┐
                   │ 构建系统提示     │
                   │ (首次或从DB恢复) │
                   │ • 身份 + 平台提示│
                   │ • 技能索引       │
                   │ • 项目上下文文件  │
                   │ • 记忆          │
                   │ • SOUL.md      │
                   └────────┬───────┘
                            │
                            ▼
                   ┌────────────────┐
                   │ 预飞行压缩检查   │
                   │ (历史过长?)     │
                   │ ← 最多3轮压缩   │
                   └────────┬───────┘
                            │
              ┌─────────────┼──────────────────┐
              │             ▼                  │
              │    ┌────────────────┐          │
              │    │ 主循环入口       │          │
              │    │ while count <  │          │
              │    │ max_iterations  │          │
              │    │ && budget > 0  │          │
              │    └────────┬───────┘          │
              │             │                  │
              │             ▼                  │
              │    ┌────────────────┐          │
              │    │ 中断检查        │          │
              │    │ 消耗迭代预算     │          │
              │    └────────┬───────┘          │
              │             │                  │
              │             ▼                  │
              │    ┌────────────────┐          │
              │    │ 构建 API 参数   │          │
              │    │ • 消息序列化     │          │
              │    │ • Honcho 注入   │          │
              │    │ • 缓存控制       │          │
              │    │ • 消息清洗       │          │
              │    └────────┬───────┘          │
              │             │                  │
              │             ▼                  │
              │    ┌────────────────┐          │
              │    │ LLM API 调用   │          │
              │    │ (带重试、降级)   │          │
              │    └────────┬───────┘          │
              │             │                  │
              │        ┌────┴────┐             │
              │        │         │             │
              │   有工具调用   纯文本回复        │
              │        │         │             │
              │        ▼         ▼             │
              │  ┌──────────┐ ┌──────────┐    │
              │  │ 工具执行   │ │ 返回结果  │    │
              │  │ (并行/串行)│ │ break    │    │
              │  └─────┬────┘ └──────────┘    │
              │        │                       │
              │        ▼                       │
              │  ┌──────────┐                  │
              │  │ 追加结果   │                  │
              │  │ 压缩检查   │                  │
              │  │ continue  │                  │
              │  └──────────┘                  │
              │        │                       │
              └────────┘  (循环)                │
                                               │
                            ┌──────────────────┘
                            ▼
                   ┌────────────────┐
                   │ 收尾阶段        │
                   │ • 保存轨迹      │
                   │ • 持久化会话    │
                   │ • Honcho 同步  │
                   │ • 后台记忆刷新  │
                   └────────────────┘
```

### 3.4 工具调用执行流程

```
AIAgent._execute_tool_calls()
        │
        ▼
┌───────────────────────┐
│ _should_parallelize?  │──── 判断是否可并行
└───────┬───────────────┘
        │
   ┌────┴────┐
   │         │
 可并行    须串行
   │         │
   ▼         ▼
┌────────┐  ┌───────────────────────┐
│并行执行 │  │ 串行执行               │
│        │  │                       │
│ Thread │  │  for tc in tool_calls:│
│ Pool   │  │    _invoke_tool(tc)   │
│ Exec   │  │                       │
│(max 8) │  │ 特殊处理:              │
│        │  │  • todo → TodoStore   │
│        │  │  • memory → MemoryStore│
│        │  │  • clarify → callback │
│        │  │  • delegate → 子代理   │
│        │  │  • session_search → DB│
└───┬────┘  └───────────┬───────────┘
    │                   │
    └─────────┬─────────┘
              │
              ▼
    ┌─────────────────┐
    │ handle_function  │
    │ _call()          │
    │                  │
    │ registry.dispatch│
    │   ↓              │
    │ handler(args)    │
    │   ↓              │
    │ JSON 字符串结果   │
    └─────────────────┘

并行安全判断 (_should_parallelize_tool_batch):
  ├─ clarify → 永不并行
  ├─ _NEVER_PARALLEL_TOOLS → 串行
  ├─ _PARALLEL_SAFE_TOOLS → 可并行
  ├─ 路径作用域工具 → 路径不重叠时可并行
  └─ 默认 → 串行
```

### 3.5 上下文压缩流程

```
触发条件: should_compress(prompt_tokens) == True
         或 should_compress_preflight(messages) == True
              │
              ▼
     ┌────────────────────────┐
     │ compress(messages)      │
     └────────┬───────────────┘
              │
              ▼
     ┌────────────────────────┐
     │ 1. 修剪旧工具结果       │
     │    _prune_old_tool_     │
     │    results()            │
     │    (>200字符→占位符)     │
     └────────┬───────────────┘
              │
              ▼
     ┌────────────────────────┐
     │ 2. 分割消息区域         │
     │                        │
     │  ┌──────┐ ┌──────┐ ┌──────┐
     │  │ Head │ │Middle│ │ Tail │
     │  │(前3条)│ │(摘要)│ │(尾部)│
     │  └──────┘ └──────┘ └──────┘
     │                        │
     │  Head: protect_first_n │
     │  Tail: token_budget    │
     │  Middle: 待压缩区域     │
     └────────┬───────────────┘
              │
              ▼
     ┌────────────────────────┐
     │ 3. 对齐边界             │
     │    确保工具调用/结果配对  │
     │    不被分割             │
     │    _align_boundary_*    │
     └────────┬───────────────┘
              │
              ▼
     ┌────────────────────────┐
     │ 4. 生成摘要             │
     │    _generate_summary()  │
     │                        │
     │    首次: 从零生成结构化   │
     │      摘要模板:          │
     │      • Goal            │
     │      • Constraints     │
     │      • Progress        │
     │      • Key Decisions   │
     │      • Relevant Files  │
     │      • Next Steps      │
     │      • Critical Context│
     │                        │
     │    后续: 迭代更新已有摘要│
     │      保留旧信息         │
     │      添加新进展         │
     └────────┬───────────────┘
              │
              ▼
     ┌────────────────────────┐
     │ 5. 组装压缩后消息       │
     │    Head + Summary + Tail│
     │                        │
     │ 6. 清理孤立工具对       │
     │    _sanitize_tool_pairs│
     └────────────────────────┘
```

### 3.6 子代理委托流程

```
父代理 AIAgent          delegate_tool          子代理 AIAgent
    │                       │                      │
    │ delegate_task(        │                      │
    │   goal, context,     │                      │
    │   toolsets, tasks)   │                      │
    │─────────────────────►│                      │
    │                       │                      │
    │                       │ 检查深度 (MAX_DEPTH=2)│
    │                       │                      │
    │                       │ 过滤阻止工具:         │
    │                       │  - delegate_task     │
    │                       │  - clarify           │
    │                       │  - memory            │
    │                       │  - send_message      │
    │                       │  - execute_code      │
    │                       │                      │
    │                       │ 保存 _last_resolved_  │
    │                       │ tool_names (全局状态) │
    │                       │                      │
    │                       │ 创建子 AIAgent        │
    │                       │─────────────────────►│
    │                       │                      │
    │                       │                      │ run_conversation()
    │                       │                      │ [独立对话循环]
    │                       │                      │
    │                       │                      │ 完成
    │                       │◄─────────────────────│
    │                       │                      │
    │                       │ 恢复全局状态          │
    │                       │                      │
    │                       │ 聚合结果              │
    │◄──JSON结果────────────│                      │
    │                       │                      │

批量任务时:
  tasks: [task1, task2, task3]  (最多3个)
     │
     ▼
  ThreadPoolExecutor (MAX_CONCURRENT_CHILDREN=3)
     ├─ 子代理1 → 结果1
     ├─ 子代理2 → 结果2
     └─ 子代理3 → 结果3
     │
     ▼
  聚合为 JSON 数组返回父代理
```

### 3.7 MCP 工具发现流程

```
model_tools._discover_tools()
        │
        ▼
  导入所有 tools/*.py 模块
  (各模块在导入时调用 registry.register())
        │
        ▼
  discover_mcp_tools()
        │
        ▼
┌───────────────────────────────┐
│ 读取 config.yaml mcp_servers │
│ 或 mcp.json / mcp_*.json    │
└──────────┬────────────────────┘
           │
           ▼ (对每个 MCP 服务器)
┌───────────────────────────────┐
│ 创建 asyncio 事件循环(守护线程) │
│ _ensure_mcp_loop()            │
└──────────┬────────────────────┘
           │
           ▼
┌───────────────────────────────┐
│ 连接 MCP 服务器               │
│ • stdio: 启动子进程           │
│ • sse: HTTP SSE 连接          │
│ • streamable-http: WebSocket  │
└──────────┬────────────────────┘
           │
           ▼
┌───────────────────────────────┐
│ session.list_tools()          │
│ 获取服务器暴露的工具列表       │
└──────────┬────────────────────┘
           │
           ▼ (对每个 MCP 工具)
┌───────────────────────────────┐
│ 生成唯一名称:                  │
│ mcp_{server}_{tool}           │
│                               │
│ 转换 Schema:                  │
│ _convert_mcp_schema()         │
│                               │
│ 创建处理器:                    │
│ _make_tool_handler() →        │
│   session.call_tool(name,args)│
│                               │
│ registry.register(            │
│   name, toolset, schema,      │
│   handler, check_fn)          │
└──────────┬────────────────────┘
           │
           ▼
┌───────────────────────────────┐
│ 可选: 注册资源/提示工具        │
│ list_resources, read_resource │
│ list_prompts, get_prompt      │
│                               │
│ _sync_mcp_toolsets():         │
│ 将 MCP 工具名加入 TOOLSETS    │
│ 的 hermes-cli 等平台工具集    │
└───────────────────────────────┘
```

---

## 4. 算法设计

### 4.1 上下文窗口压缩算法

上下文压缩是 Hermes-Agent 最核心的算法之一，确保在长对话中不超出 LLM 上下文窗口限制。

#### 算法概述

```
输入: messages[] (OpenAI 格式消息列表)
输出: compressed_messages[] (压缩后的消息列表)

阶段 1: 工具输出修剪 (O(n), 无 LLM 调用)
  对保护尾部之外的所有 tool 消息:
    if len(content) > 200:
      content ← "[Old tool output cleared to save context space]"

阶段 2: 区域划分
  head ← messages[0 : protect_first_n]
  对齐 head 边界: 向前跳过孤立工具结果

  tail_cut ← 从末尾向前累积 token 直到 tail_token_budget
  tail_cut ← max(tail_cut, len(messages) - protect_last_n)
  对齐 tail 边界: 确保工具调用组不被分割

  middle ← messages[head_end : tail_cut]

阶段 3: LLM 摘要生成
  summary_budget ← min(content_tokens * 0.20, context_length * 0.05, 12000)
  summary_budget ← max(summary_budget, 2000)

  if _previous_summary exists:
    prompt ← 迭代更新提示 (保留旧信息 + 整合新进展)
  else:
    prompt ← 初始摘要提示 (结构化模板)

  summary ← call_llm(prompt, max_tokens=summary_budget)

阶段 4: 组装
  result ← head + [summary_message] + tail
  result ← _sanitize_tool_pairs(result)
    - 移除无对应调用的工具结果
    - 为缺失结果的工具调用补充存根
```

#### 关键参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `threshold_percent` | 0.50 | 上下文使用率触发阈值 |
| `protect_first_n` | 3 | 始终保留的头部消息数 |
| `protect_last_n` | 20 | 尾部最小保护消息数 |
| `_SUMMARY_RATIO` | 0.20 | 压缩内容的摘要占比 |
| `_SUMMARY_TOKENS_CEILING` | 12,000 | 摘要 token 绝对上限 |
| `_MIN_SUMMARY_TOKENS` | 2,000 | 摘要 token 最小值 |
| `_CHARS_PER_TOKEN` | 4 | 粗略 token 估算系数 |

#### 工具对完整性保障

```python
def _sanitize_tool_pairs(messages):
    """确保每个 tool_call 有对应结果, 每个 tool result 有对应调用"""

    # 收集所有 tool_call_id
    call_ids = {tc.id for msg in messages for tc in msg.tool_calls if msg.role == "assistant"}
    result_ids = {msg.tool_call_id for msg in messages if msg.role == "tool"}

    # 孤立结果: 删除
    messages = [m for m in messages if m.role != "tool" or m.tool_call_id in call_ids]

    # 缺失结果: 补充存根
    for call_id in (call_ids - result_ids):
        messages.append({"role": "tool", "tool_call_id": call_id,
                         "content": "[Result cleared during compression]"})
```

### 4.2 工具发现与解析算法

#### 工具发现 (`_discover_tools()`)

```python
def _discover_tools():
    """启动时导入所有工具模块, 触发 registry.register() 调用"""

    tool_modules = [
        "tools.terminal_tool",      # terminal, process
        "tools.file_tools",         # read_file, write_file, patch, search_files
        "tools.web_tools",          # web_search, web_extract
        "tools.browser_tool",       # browser_* (11个工具)
        "tools.code_execution_tool", # execute_code
        "tools.delegate_tool",      # delegate_task
        "tools.vision_tool",        # vision_analyze
        "tools.image_gen_tool",     # image_generate
        "tools.moa_tool",           # mixture_of_agents
        "tools.skills_tool",        # skills_list, skill_view, skill_manage
        "tools.tts_tool",           # text_to_speech
        "tools.todo_tool",          # todo
        "tools.memory_tool",        # memory
        "tools.session_search_tool", # session_search
        "tools.clarify_tool",       # clarify
        "tools.cronjob_tool",       # cronjob
        "tools.send_message_tool",  # send_message
        "tools.honcho_tool",        # honcho_*
        "tools.homeassistant_tool", # ha_*
        "tools.rl_tools",           # rl_*
    ]

    for module_name in tool_modules:
        try:
            importlib.import_module(module_name)
        except Exception as e:
            logger.warning("Failed to import %s: %s", module_name, e)

    # MCP 动态发现
    try:
        from tools.mcp_tool import discover_mcp_tools
        discover_mcp_tools()
    except Exception:
        pass

    # 插件发现
    try:
        from hermes_cli.plugins import discover_plugins
        discover_plugins()
    except Exception:
        pass
```

#### 工具定义过滤 (`get_tool_definitions()`)

```
输入: enabled_toolsets, disabled_toolsets, quiet_mode
输出: List[dict] (OpenAI function-calling schemas)

算法:
  1. 确定要包含的工具名集合:
     if enabled_toolsets:
       tools_to_include ← union(resolve_toolset(ts) for ts in enabled_toolsets)
     elif disabled_toolsets:
       all_tools ← union(all toolset tools)
       disabled ← union(resolve_toolset(ts) for ts in disabled_toolsets)
       tools_to_include ← all_tools - disabled
     else:
       tools_to_include ← all registered tools

  2. 可用性过滤:
     for tool in tools_to_include:
       entry = registry.get(tool)
       if entry.check_fn and not entry.check_fn():
         skip  # API 密钥缺失等

  3. 后处理:
     - 动态构建 execute_code schema (描述中列出可用工具)
     - 条件移除 browser_navigate 的 web_search 提示
     - 更新全局 _last_resolved_tool_names

  返回: [{"type": "function", "function": schema}, ...]
```

### 4.3 工具集递归解析算法

```python
def resolve_toolset(name: str, visited: Set[str] = None) -> List[str]:
    """递归解析工具集, 支持组合和钻石依赖"""

    if visited is None:
        visited = set()

    # 特殊别名: "all" / "*" → 所有工具
    if name in {"all", "*"}:
        return union(resolve_toolset(ts, visited.copy()) for ts in get_toolset_names())

    # 循环/钻石检测: 已访问则返回空 (工具已通过其他路径收集)
    if name in visited:
        return []

    visited.add(name)

    toolset = TOOLSETS.get(name)
    if not toolset:
        # 回退: 从插件注册表查找
        return plugin_tools_for_name(name) or []

    # 直接工具
    tools = set(toolset["tools"])

    # 递归展开包含的工具集 (共享 visited 防止重复)
    for included in toolset.get("includes", []):
        tools.update(resolve_toolset(included, visited))

    return list(tools)
```

**复杂度:** O(V + E), 其中 V 是工具集数量, E 是包含关系数。

**钻石依赖处理:** `visited` 集合在兄弟 `includes` 间共享, 确保每个工具集最多解析一次。

### 4.4 Prompt 缓存策略

Hermes-Agent 使用 Anthropic 的 prompt caching（`system_and_3` 策略）来减少重复输入成本。

```
策略: system_and_3
最大断点数: 4 (Anthropic 限制)

断点分配:
  1. 系统提示 (如果 messages[0].role == "system")
  2-4. 最后 3 条非系统消息 (滚动窗口)

缓存标记注入:
  对每个断点消息:
    if content is str:
      content → [{"type": "text", "text": content, "cache_control": {"type": "ephemeral"}}]
    if content is list:
      content[-1]["cache_control"] = {"type": "ephemeral"}
    if role == "tool" and native_anthropic:
      msg["cache_control"] = {"type": "ephemeral"}

效果:
  • 系统提示在整个会话中缓存 (不变)
  • 近期消息在连续轮次中缓存
  • 输入 token 成本降低约 75%
```

**缓存保护策略：**

Hermes-Agent 在整个会话中保持缓存有效，做法包括：
1. 系统提示在首轮构建后缓存 (`_cached_system_prompt`)，会话中不再修改
2. 恢复会话时从数据库读取原始系统提示，而非重新构建
3. 禁止会话中途更换工具集、重新加载记忆等可能改变前缀的操作
4. 唯一允许修改上下文的操作是压缩（此时重建缓存断点）

### 4.5 模型上下文长度解析算法

```
输入: model (模型标识), base_url, api_key, provider, config_context_length
输出: context_length (int, 上下文窗口 token 数)

解析优先级链 (从高到低):
  1. config_context_length (用户配置覆盖)
         ↓ 未命中
  2. 磁盘缓存 (~/.hermes/context_lengths.yaml)
         ↓ 未命中
  3. 自定义端点 /models API 查询
     (仅未知主机)
         ↓ 未命中
  4. 本地服务器探测
     (localhost/私有 IP → Ollama/vLLM/LM Studio)
         ↓ 未命中
  5. Anthropic /v1/models API
     (provider == "anthropic")
         ↓ 未命中
  6. 推断 provider + models.dev 查询
     (api.json 远程注册表)
         ↓ 未命中
  7. OpenRouter 模型缓存
     (OPENROUTER_MODELS_URL)
         ↓ 未命中
  8. 内置 DEFAULT_CONTEXT_LENGTHS 模糊匹配
     (已知模型家族静态映射)
         ↓ 未命中
  9. 本地服务器再次尝试
         ↓ 未命中
  10. 默认值 128,000 tokens
```

**错误恢复的上下文探测：**

```python
CONTEXT_PROBE_TIERS = [128_000, 64_000, 32_000, 16_000, 8_000, 4_096]

def get_next_probe_tier(current: int) -> int:
    """413 错误后降级到下一个较小的上下文层级"""
    for tier in CONTEXT_PROBE_TIERS:
        if tier < current:
            return tier
    return 4_096  # 最终兜底
```

### 4.6 并行工具执行调度算法

```python
def _should_parallelize_tool_batch(tool_calls) -> bool:
    """决定一批工具调用是否可以安全并行执行"""

    # 规则 1: clarify 工具永不并行 (需要用户交互)
    if any(tc.name == "clarify" for tc in tool_calls):
        return False

    # 规则 2: 检查每个工具的并行安全性
    for tc in tool_calls:
        name = tc.function.name

        # 永不并行的工具 (如 terminal 有副作用)
        if name in _NEVER_PARALLEL_TOOLS:
            return False

        # 已知并行安全的工具
        if name in _PARALLEL_SAFE_TOOLS:
            continue

        # 路径作用域工具 (如 read_file, write_file)
        if name in _PATH_SCOPED_TOOLS:
            path = _extract_parallel_scope_path(tc)
            if path:
                # 检查路径是否与其他工具重叠
                other_paths = [_extract_parallel_scope_path(o)
                               for o in tool_calls if o != tc]
                if any(_paths_overlap(path, op) for op in other_paths if op):
                    return False
            continue

        # 未知工具 → 保守串行
        return False

    return True

# 并行执行器
MAX_TOOL_WORKERS = 8
with ThreadPoolExecutor(max_workers=MAX_TOOL_WORKERS) as pool:
    futures = {pool.submit(_invoke_tool, tc): tc for tc in tool_calls}
    for future in as_completed(futures):
        result = future.result()
```

**分类表:**

| 类别 | 工具示例 | 策略 |
|------|---------|------|
| `_NEVER_PARALLEL` | `terminal` | 永远串行 |
| `_PARALLEL_SAFE` | `web_search`, `web_extract` | 始终可并行 |
| `_PATH_SCOPED` | `read_file`, `write_file`, `patch`, `search_files` | 路径不重叠时可并行 |
| 代理级工具 | `todo`, `memory`, `clarify`, `delegate_task` | 串行 |

### 4.7 危险命令检测算法

```python
DANGEROUS_PATTERNS = [
    (r"rm\s+(-[a-zA-Z]*)*\s*(/|~|\$HOME)", "Recursive/root deletion"),
    (r"mkfs\.", "Filesystem formatting"),
    (r"dd\s+", "Raw disk write"),
    (r"chmod\s+(-[a-zA-Z]*\s+)*0?777", "World-writable permissions"),
    (r">\s*/dev/sd[a-z]", "Direct device write"),
    (r"curl\s+.*\|\s*(ba)?sh", "Pipe to shell"),
    (r"wget\s+.*\|\s*(ba)?sh", "Pipe to shell"),
    (r":(){ :\|:& };:", "Fork bomb"),
    # ... 更多模式
]

def detect_dangerous_command(command: str) -> Tuple[bool, str, str]:
    """检测命令是否匹配危险模式"""
    command_lower = command.lower()
    for pattern, description in DANGEROUS_PATTERNS:
        if re.search(pattern, command_lower, re.IGNORECASE | re.DOTALL):
            return (True, description, description)
    return (False, None, None)

def check_all_command_guards(command, task_id, ...) -> Tuple[bool, str]:
    """组合所有安全检查"""

    warnings = []

    # 1. Tirith 安全策略检查 (可选)
    if tirith_enabled:
        result = check_command_security(command)
        if result.action == "block":
            return (False, result.message)  # 不可批准的硬阻断
        if result.warnings:
            warnings.extend(result.warnings)

    # 2. 危险模式检测
    is_dangerous, pattern_key, desc = detect_dangerous_command(command)
    if is_dangerous:
        warnings.append((pattern_key, desc))

    # 3. 根据批准模式处理
    if not warnings:
        return (True, "")

    mode = config.get("approvals.mode", "manual")

    if mode == "off":
        return (True, "")

    if mode == "smart":
        # LLM 辅助判断
        verdict = _smart_approve(command, warnings)
        if verdict == "APPROVE":
            return (True, "")
        if verdict == "DENY":
            return (False, "Smart approval denied")
        # ESCALATE → 继续到人工审批

    # manual 模式或 smart ESCALATE
    return prompt_user_for_approval(command, warnings)
```

### 4.8 迭代预算管理算法

```python
class IterationBudget:
    """线程安全的迭代预算管理器"""

    def __init__(self, max_total: int):
        self.max_total = max_total
        self._used = 0
        self._lock = threading.Lock()

    def consume(self) -> bool:
        """消耗一次迭代。返回 False 表示预算耗尽。"""
        with self._lock:
            if self._used >= self.max_total:
                return False
            self._used += 1
            return True

    def refund(self):
        """退还一次迭代 (用于 execute_code 等轻量工具)"""
        with self._lock:
            if self._used > 0:
                self._used -= 1

    @property
    def remaining(self) -> int:
        with self._lock:
            return max(0, self.max_total - self._used)

# 预算警告注入
def _get_budget_warning(api_call_count: int, max_iterations: int) -> str:
    """在工具结果中注入预算警告"""
    usage = api_call_count / max_iterations

    if usage >= 0.90:
        return ("⚠️ CRITICAL: Only {remaining} iterations remaining. "
                "Wrap up immediately.")
    elif usage >= 0.70:
        return ("⚠️ Budget alert: {used}/{max} iterations used. "
                "Consider wrapping up soon.")
    return ""

# 退还规则:
# - 如果一个工具批次中所有工具都是 execute_code → 退还一次
# - 压缩重启时退还一次
```

### 4.9 会话全文搜索算法（FTS5）

```sql
-- FTS5 虚拟表 (外部内容表模式)
CREATE VIRTUAL TABLE messages_fts USING fts5(
    content,
    content=messages,
    content_rowid=id
);

-- 同步触发器
CREATE TRIGGER messages_ai AFTER INSERT ON messages BEGIN
    INSERT INTO messages_fts(rowid, content) VALUES (new.id, new.content);
END;

CREATE TRIGGER messages_au AFTER UPDATE ON messages BEGIN
    INSERT INTO messages_fts(messages_fts, rowid, content)
        VALUES('delete', old.id, old.content);
    INSERT INTO messages_fts(rowid, content) VALUES (new.id, new.content);
END;

CREATE TRIGGER messages_ad AFTER DELETE ON messages BEGIN
    INSERT INTO messages_fts(messages_fts, rowid, content)
        VALUES('delete', old.id, old.content);
END;
```

```python
def search_messages(self, query, session_id=None, source=None, role=None, limit=20):
    """使用 FTS5 全文搜索消息"""

    # 查询清洗
    safe_query = self._sanitize_fts5_query(query)

    sql = """
        SELECT m.*, s.source, s.model,
               snippet(messages_fts, 0, '>>>', '<<<', '...', 64) as snippet,
               rank
        FROM messages_fts
        JOIN messages m ON m.id = messages_fts.rowid
        JOIN sessions s ON s.id = m.session_id
        WHERE messages_fts MATCH ?
    """
    params = [safe_query]

    if session_id:
        sql += " AND m.session_id = ?"
        params.append(session_id)
    if source:
        sql += " AND s.source = ?"
        params.append(source)
    if role:
        sql += " AND m.role = ?"
        params.append(role)

    sql += " ORDER BY rank LIMIT ?"
    params.append(limit)

    return self.conn.execute(sql, params).fetchall()

def _sanitize_fts5_query(self, query: str) -> str:
    """清洗 FTS5 查询, 防止注入和语法错误"""
    # 引用短语
    # 移除危险的 FTS 操作符 (NEAR, NOT 等)
    # 连字符术语加引号
    # 确保合法的 FTS5 语法
    ...
```

### 4.10 代码执行沙箱机制

```
execute_code 工具的沙箱架构:

┌──────────────────────────────────────────────────┐
│                   父进程 (Agent)                  │
│                                                   │
│  1. 生成 hermes_tools.py (RPC 存根)              │
│     - web_search() → Unix socket → 父进程        │
│     - read_file() → Unix socket → 父进程         │
│     - terminal() → Unix socket → 父进程          │
│     - ... (仅当前会话启用的工具)                    │
│                                                   │
│  2. 生成 script.py (用户代码)                      │
│                                                   │
│  3. 创建 Unix Domain Socket                       │
│     RPC 服务器线程 (_rpc_server_loop)              │
│     ├─ 接收工具调用请求                            │
│     ├─ 调用 handle_function_call()               │
│     ├─ 返回 JSON 结果                             │
│     └─ 工具调用次数限制 (防止无限循环)              │
│                                                   │
│  4. 启动子进程                                     │
│     env = {安全过滤后的环境变量}                    │
│     ├─ 移除秘密类变量 (API_KEY, TOKEN, SECRET)    │
│     ├─ 保留 env_passthrough 白名单                │
│     ├─ 保留安全前缀变量                            │
│     └─ terminal 工具: 去除 background/pty 参数    │
│                                                   │
│  5. 捕获 stdout/stderr                           │
│     ├─ 头部 + 尾部截断                            │
│     └─ 超时控制                                   │
│                                                   │
│  6. 中断: SIGTERM → 进程组                        │
│                                                   │
└──────────────────────────────────────────────────┘

SANDBOX_ALLOWED_TOOLS (默认可用):
  web_search, web_extract, read_file, write_file,
  search_files, patch, terminal
```

---

## 5. 模块依赖链

```
层级依赖 (从底层到高层):

Level 0: 无依赖 (基础设施)
  tools/registry.py          # ToolRegistry 单例
  hermes_constants.py        # 常量与路径
  utils.py                   # 通用工具函数

Level 1: 仅依赖 Level 0
  tools/*.py                 # 各工具实现 (import registry)
  agent/model_metadata.py    # 模型元数据
  agent/prompt_caching.py    # 缓存控制 (纯函数)

Level 2: 依赖 Level 0-1
  agent/auxiliary_client.py  # 辅助 LLM 客户端
  agent/context_compressor.py # 上下文压缩
  agent/prompt_builder.py    # 提示构建
  hermes_state.py            # SessionDB
  toolsets.py                # 工具集定义 (import registry)

Level 3: 依赖 Level 0-2
  model_tools.py             # 工具编排 (import registry, toolsets, tools/*)
  agent/display.py           # UI 显示

Level 4: 依赖 Level 0-3
  run_agent.py               # AIAgent (import model_tools, agent/*)

Level 5: 依赖 Level 0-4
  cli.py                     # HermesCLI (import run_agent)
  gateway/run.py             # GatewayRunner (import run_agent)
  batch_runner.py            # BatchRunner (import run_agent)

Level 6: 入口
  hermes_cli/main.py         # CLI 入口 (import cli, gateway/run)
```

**循环依赖避免策略：**

1. `tools/registry.py` 不导入任何工具文件或 `model_tools`
2. 各工具文件仅在模块级导入 `tools/registry`
3. `registry.dispatch()` 延迟导入 `model_tools._run_async`（仅在异步工具调用时）
4. `toolsets.py` 延迟导入 `tools.registry`（仅在插件工具集查询时）

---

## 6. 扩展性设计

### 6.1 添加新工具（3 文件修改）

```
1. tools/your_tool.py:
   - 定义 handler 函数 (返回 JSON 字符串)
   - 定义 check_fn (可用性检查)
   - 调用 registry.register()

2. model_tools.py:
   - 在 _discover_tools() 列表中添加导入

3. toolsets.py:
   - 添加到 _HERMES_CORE_TOOLS (所有平台)
   - 或创建新的工具集条目
```

### 6.2 添加新平台适配器

```
1. gateway/platforms/your_platform.py:
   - 继承 BasePlatformAdapter
   - 实现 connect(), disconnect(), send_message(), send_typing()
   - 实现消息接收 → MessageEvent 转换

2. gateway/config.py:
   - 在 Platform 枚举中添加

3. gateway/run.py:
   - 在 _create_adapter() 中添加 case

4. toolsets.py:
   - 添加 hermes-your-platform 工具集
```

### 6.3 添加新斜杠命令

```
1. hermes_cli/commands.py:
   - 添加 CommandDef 到 COMMAND_REGISTRY

2. cli.py:
   - 在 process_command() 中添加 elif 分支

3. gateway/run.py (可选):
   - 添加 gateway 处理器

自动生效:
  - CLI 帮助文本
  - 自动补全
  - Telegram BotCommand 菜单
  - Slack 子命令映射
```

### 6.4 MCP 插件扩展

```yaml
# ~/.hermes/config.yaml
mcp_servers:
  my_server:
    type: stdio                    # stdio | sse | streamable-http
    command: "node"
    args: ["path/to/server.js"]
    env:
      API_KEY: "..."
    tools:
      include: ["tool1", "tool2"]  # 可选过滤
      exclude: ["tool3"]

# 运行时自动:
#   1. 启动 MCP 服务器进程
#   2. 发现工具列表
#   3. 注册为 mcp_my_server_* 工具
#   4. 注入到当前平台工具集
```

### 6.5 主题/皮肤扩展

```yaml
# ~/.hermes/skins/mytheme.yaml
name: mytheme
description: 自定义主题

colors:
  banner_border: "#FF00FF"
  banner_title: "#00FFFF"
  response_border: "#FF1493"

spinner:
  thinking_verbs: ["分析中", "思考中", "推理中"]
  wings:
    - ["⟨⚡", "⚡⟩"]

branding:
  agent_name: "My Agent"
  response_label: " ⚡ Response "
  prompt_symbol: "λ"

tool_prefix: "▏"
tool_emojis:
  web_search: "🌐"
  terminal: "💻"
```

---

## 7. 安全设计

### 7.1 多层安全防护

```
┌─────────────────────────────────────────────────┐
│ Layer 1: 用户授权 (Gateway)                      │
│   _is_user_authorized()                         │
│   ├─ Telegram: TELEGRAM_ALLOWED_USERS           │
│   ├─ Discord: DISCORD_ALLOWED_USERS             │
│   └─ 未授权用户: 忽略 / 拒绝 / 自定义回复         │
├─────────────────────────────────────────────────┤
│ Layer 2: 命令安全 (approval.py)                  │
│   check_all_command_guards()                    │
│   ├─ Tirith 策略引擎 (可选)                      │
│   ├─ 危险命令正则模式检测                         │
│   └─ Smart/Manual 审批流程                       │
├─────────────────────────────────────────────────┤
│ Layer 3: 容器隔离 (terminal environments)        │
│   TERMINAL_ENV: docker | singularity | modal    │
│   ├─ 命令在容器内执行                             │
│   └─ 容器环境跳过危险命令检测                      │
├─────────────────────────────────────────────────┤
│ Layer 4: 代码沙箱 (code_execution_tool)          │
│   ├─ 环境变量过滤 (移除秘密)                      │
│   ├─ 工具调用次数限制                             │
│   ├─ terminal 参数限制 (禁止 background/pty)     │
│   └─ 输出截断                                    │
├─────────────────────────────────────────────────┤
│ Layer 5: 提示注入防护 (prompt_builder.py)         │
│   _scan_context_content()                       │
│   ├─ Unicode 零宽字符检测                        │
│   ├─ 常见注入模式正则匹配                         │
│   └─ 命中时替换为 [BLOCKED: ...] 占位符           │
├─────────────────────────────────────────────────┤
│ Layer 6: 文件路径安全 (file_tools.py)             │
│   ├─ 阻止读取 ~/.hermes/skills/.hub              │
│   ├─ 敏感内容脱敏 (redact_sensitive_text)         │
│   └─ 重复读取检测与限制 (防止循环)                 │
├─────────────────────────────────────────────────┤
│ Layer 7: 凭证保护                                │
│   ├─ ~/.hermes/.env 权限 0600                    │
│   ├─ auth.json 文件锁保护                        │
│   ├─ 输出中的 API 密钥脱敏                        │
│   └─ 代码沙箱环境变量清洗                         │
└─────────────────────────────────────────────────┘
```

### 7.2 子代理安全隔离

```
delegate_task 安全措施:
  1. 深度限制: MAX_DEPTH = 2 (防止递归失控)
  2. 工具阻止列表:
     - delegate_task  (防止子代理再委托)
     - clarify        (子代理不能与用户交互)
     - memory         (子代理不能修改持久记忆)
     - send_message   (子代理不能发送消息)
     - execute_code   (子代理不能启动沙箱)
  3. 临时上下文: 子代理使用 ephemeral_system_prompt
  4. 全局状态保护: 保存/恢复 _last_resolved_tool_names
  5. 并发限制: MAX_CONCURRENT_CHILDREN = 3
```

### 7.3 Web 安全

```
Web 工具安全措施:
  1. URL 安全检查 (website policy)
  2. 内容大小限制与截断
  3. LLM 内容压缩 (process_content_with_llm)
     - 分块处理大文档
     - 防止上下文窗口溢出
  4. 浏览器自动化:
     - 会话隔离 (per-task_id)
     - 空闲超时自动清理
     - Accessibility snapshot 替代原始 DOM
```

---

## 附录 A: 关键数据流总结

| 数据流 | 路径 | 说明 |
|--------|------|------|
| 用户输入 → LLM | CLI/Gateway → AIAgent → OpenAI API | 消息通过 conversation_history 累积 |
| 工具调用 | LLM → AIAgent → model_tools → registry → handler | JSON 字符串返回 |
| 持久化 | AIAgent → SessionDB (SQLite) | WAL 模式, 全量消息存储 |
| 搜索 | SessionDB → FTS5 | 全文搜索虚拟表 + rank 排序 |
| 压缩 | ContextCompressor → auxiliary_client → summary | 结构化摘要模板 |
| 缓存 | apply_anthropic_cache_control → API messages | 4 个 cache_control 断点 |
| 配置 | ~/.hermes/config.yaml → load_config() | YAML 合并 + 版本迁移 |
| 认证 | ~/.hermes/auth.json + .env → resolve_provider() | 文件锁 + OAuth 设备流 |
| 主题 | ~/.hermes/skins/*.yaml → SkinConfig | 数据驱动, 无代码修改 |

## 附录 B: 关键常量参考

| 常量 | 值 | 位置 | 说明 |
|------|-----|------|------|
| `max_iterations` | 90 | run_agent.py | 默认最大 API 循环次数 |
| `MAX_TOOL_WORKERS` | 8 | run_agent.py | 并行工具执行最大线程数 |
| `MAX_DEPTH` | 2 | delegate_tool.py | 子代理最大嵌套深度 |
| `MAX_CONCURRENT_CHILDREN` | 3 | delegate_tool.py | 并发子代理最大数 |
| `threshold_percent` | 0.50 | context_compressor.py | 压缩触发阈值 |
| `_SUMMARY_RATIO` | 0.20 | context_compressor.py | 摘要占压缩内容比例 |
| `_SUMMARY_TOKENS_CEILING` | 12,000 | context_compressor.py | 摘要 token 上限 |
| `CONTEXT_FILE_MAX_CHARS` | 20,000 | prompt_builder.py | 上下文文件最大字符 |
| `SCHEMA_VERSION` | 6 | hermes_state.py | 数据库 schema 版本 |
| `_config_version` | 10 | config.py | 配置文件版本 |
| `TERMINAL_TIMEOUT` | 120s | terminal_tool.py | 终端命令默认超时 |

## 附录 C: 支持的 LLM 提供商

| 提供商 | API 模式 | 认证方式 |
|--------|---------|---------|
| Anthropic | anthropic_messages / chat_completions | API Key / OAuth |
| OpenAI | chat_completions | API Key |
| OpenRouter | chat_completions | API Key |
| OpenAI Codex | codex_responses | Device Code OAuth |
| GitHub Copilot | chat_completions / codex_responses | Copilot ACP |
| Nous Research | chat_completions | Device Code OAuth |
| DeepSeek | chat_completions | API Key |
| ZAI (Z.ai) | chat_completions | API Key |
| Kimi | chat_completions | API Key |
| MiniMax | chat_completions | API Key |
| Alibaba (Qwen) | chat_completions | API Key |
| 自定义端点 | chat_completions | API Key |
| 本地 (Ollama/vLLM/LM Studio) | chat_completions | 无 |
