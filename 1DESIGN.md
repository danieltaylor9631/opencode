# OpenCode 设计说明书

| 项目 | 内容 |
|------|------|
| 文档名称 | OpenCode 系统设计说明书 |
| 项目名称 | OpenCode（`github.com/opencode-ai/opencode`） |
| 文档版本 | 1.0 |
| 对应代码 | 归档快照（后续演进见 Charm Crush） |
| 语言 | Go 1.24 |
| 运行形态 | 终端交互式 TUI + 非交互 CLI（`-p`） |
| 编制日期 | 2026-09-20 |

---

## 1. 文档目的与范围

本说明书依据仓库全部源代码编写，覆盖：

1. 整体架构与模块边界；
2. 每一个持久化数据模型、领域对象、配置对象、事件对象；
3. 每一个面向用户的功能与内部子系统；
4. 每一个公开函数、方法、关键内部函数的职责、输入、输出与副作用；
5. 每一个核心算法（Agent 循环、检索、补丁匹配、权限、压缩、流式协议等）；
6. 仓库中每一个测试用例及其断言意图。

LSP 协议生成代码（`internal/lsp/protocol/tsprotocol.go`、`tsjson.go`）包含数百个 LSP 规范类型，本说明书将其视为外部协议适配层，不逐字段展开，但列出入口与应用侧实际使用的子集。

---

## 2. 系统概述

OpenCode 是运行在开发者本机终端中的 **Agentic Coding Assistant**。用户用自然语言描述任务，系统通过多提供商大语言模型（LLM）进行推理，并调用一组本地工具（读文件、搜索、编辑、执行 shell、访问 MCP 服务器、查询 LSP 诊断等）完成编码工作。

系统同时支持：

- **交互模式**：Bubble Tea TUI，会话、权限对话框、主题、模型切换、自定义命令；
- **非交互模式**：`opencode -p "..."`，自动批准权限，将最终助手文本以 `text` 或 `json` 输出到 stdout。

核心设计原则：

| 原则 | 实现 |
|------|------|
| 单一二进制 | `main.go` → Cobra `cmd.Execute()` |
| 组合根 | `internal/app.App` 组装会话、消息、历史、权限、Coder Agent、LSP |
| 领域服务 + SQLite | sqlc 生成查询，goose 迁移 |
| 进程内事件总线 | 泛型 `pubsub.Broker[T]` 驱动 TUI 刷新 |
| 提供商无关 Agent 循环 | `provider.Provider` 抽象流式/非流式调用 |
| 工具即接口 | `tools.BaseTool`：`Info()` + `Run()` |
| 危险操作需授权 | `permission.Service.Request` 同步阻塞直到用户选择 |

---

## 3. 整体架构设计

### 3.1 逻辑分层

```
┌─────────────────────────────────────────────────────────────┐
│  展示层  cmd/  +  internal/tui  +  internal/format          │
│  Cobra CLI、Bubble Tea 页面/对话框、非交互 spinner/输出     │
└──────────────────────────────┬──────────────────────────────┘
                               │ pubsub 事件 / 直接方法调用
┌──────────────────────────────▼──────────────────────────────┐
│  应用层  internal/app                                        │
│  App 组合根：服务装配、非交互 Run、LSP 生命周期、主题初始化 │
└──────────────────────────────┬──────────────────────────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        ▼                      ▼                      ▼
┌───────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ 领域服务      │    │ Agent 运行时    │    │ 集成适配        │
│ session       │    │ llm/agent       │    │ llm/provider    │
│ message       │    │ llm/tools       │    │ llm/models      │
│ history       │    │ llm/prompt      │    │ lsp + watcher   │
│ permission    │    │ agent-tool/MCP  │    │ MCP (mcp-go)    │
└───────┬───────┘    └────────┬────────┘    └────────┬────────┘
        │                     │                      │
        ▼                     ▼                      ▼
┌─────────────────────────────────────────────────────────────┐
│  基础设施                                                    │
│  db (SQLite WAL + goose)  logging  config(viper)  pubsub    │
│  fileutil  diff/patch  completions  version                 │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 启动时序

1. `main.main` 注册 panic 恢复，调用 `cmd.Execute()`。
2. `rootCmd.RunE` 解析 flags：`-d/--debug`、`-c/--cwd`、`-p/--prompt`、`-f/--output-format`、`-q/--quiet`、`-v/--version`。
3. `os.Chdir(cwd)`（若指定），`config.Load(cwd, debug)` 合并全局/本地 JSON、环境变量、提供商默认模型。
4. `db.Connect()` 打开 `{data.directory}/opencode.db`，执行嵌入的 goose 迁移。
5. `app.New(ctx, conn)`：创建 session/message/history/permission 服务，后台 `initLSPClients`，构造 Coder Agent（含全部工具）。
6. `initMCPTools` 以 30 秒超时在后台 `ListTools`，填充全局 `mcpTools` 缓存。
7. 若 `prompt != ""`：`App.RunNonInteractive`（自动批准该会话权限）后退出。
8. 否则启动 Bubble Tea：`tea.WithAltScreen()`，`setupSubscriptions` 将 logging/session/message/permission/agent 事件桥接到 `program.Send`。
9. 退出时 `cleanup`：`app.Shutdown`（取消 watcher、5s 超时关闭 LSP）、取消订阅、等待 TUI handler。

### 3.3 运行时数据流（一次用户提问）

```
用户输入 (TUI editor / -p)
    → session.Create（如无当前会话）
    → agent.Run(sessionID, content, attachments)
        → 若会话已有 SummaryMessageID：截断历史并从摘要起作为 User
        → messages.Create(User)
        → 循环:
            provider.StreamResponse(history, tools)
            流式更新 Assistant 消息 (thinking/content/tool_use)
            若 FinishReason == tool_use:
                按顺序 tool.Run
                权限拒绝则取消后续工具
                messages.Create(Tool results)
                将 Assistant + Tool 追加进 history，继续循环
            否则结束
        → TrackUsage（token/cost）
        → 首次消息异步 generateTitle
    → TUI 订阅 AgentEvent / Message updated 渲染
    → 若 tokens ≥ 0.95 × contextWindow 且 autoCompact：Summarize
```

### 3.4 进程与外部依赖

| 依赖 | 用途 |
|------|------|
| SQLite（ncruces/go-sqlite3，wazero） | 会话、消息、文件快照 |
| 各 LLM HTTP API | Anthropic / OpenAI 兼容 / Gemini / Copilot 等 |
| ripgrep / fzf（可选） | grep/glob 加速 |
| 用户配置的 LSP 进程 | gopls 等 |
| 用户配置的 MCP 进程或 SSE 端点 | 动态工具 |
| 配置的 shell（默认 `$SHELL` 或 `/bin/bash -l`） | bash 工具持久会话 |

### 3.5 包职责一览

| 包 | 职责 |
|----|------|
| `main` | 入口与 panic 恢复 |
| `cmd` | Cobra 根命令、订阅桥、MCP 预热 |
| `cmd/schema` | 从 `SupportedModels` 生成 JSON Schema |
| `internal/app` | 组合根、非交互、LSP 启停 |
| `internal/config` | Viper 配置、校验、模型/主题持久化 |
| `internal/db` | sqlc Queries、Connect、embed 迁移 |
| `internal/session` | 会话 CRUD + pubsub |
| `internal/message` | 消息 CRUD、parts JSON |
| `internal/history` | 文件版本快照 |
| `internal/permission` | 同步授权 |
| `internal/pubsub` | 泛型 broker |
| `internal/llm/agent` | Agent 循环、标题、压缩、MCP 包装、子 Agent |
| `internal/llm/provider` | 多厂商适配 |
| `internal/llm/models` | 模型目录与计价 |
| `internal/llm/prompt` | 系统提示与项目上下文注入 |
| `internal/llm/tools` | 内置工具 |
| `internal/llm/tools/shell` | 持久 bash |
| `internal/lsp` | LSP JSON-RPC 客户端 |
| `internal/lsp/watcher` | 工作区文件监视 |
| `internal/tui` | 根 Model、快捷键、自动压缩 |
| `internal/logging` | slog、会话日志、RecoverPanic |
| `internal/diff` | unified diff 渲染与 patch 应用 |
| `internal/fileutil` | rg/fzf/doublestar |
| `internal/format` | 非交互输出 |
| `internal/completions` | `@` 路径补全 |
| `internal/version` | 构建版本字符串 |

---

## 4. 数据模型设计

### 4.1 持久化模型（SQLite / sqlc）

数据库路径：`{config.Data.Directory}/opencode.db`，默认目录 `.opencode`。连接使用 WAL。时间戳为 Unix 秒（SQL 中 `strftime('%s','now')`）。

#### 4.1.1 `sessions`

| 列 | 类型 | 约束 | 含义 |
|----|------|------|------|
| `id` | TEXT | PK | UUID；任务会话使用 tool call ID；标题会话为 `title-{parent}` |
| `parent_session_id` | TEXT | 可空 | 子任务/标题会话指向父会话 |
| `title` | TEXT | NOT NULL | 展示标题 |
| `message_count` | INTEGER | ≥0，触发器维护 | 消息条数 |
| `prompt_tokens` | INTEGER | ≥0 | 最近一轮输入侧 token（含 cache creation） |
| `completion_tokens` | INTEGER | ≥0 | 最近一轮输出侧 token（含 cache read） |
| `cost` | REAL | ≥0 | 累计美元成本 |
| `summary_message_id` | TEXT | 可空 | 压缩后摘要消息 ID |
| `created_at` / `updated_at` | INTEGER | NOT NULL | 创建/更新时间 |

触发器：

- `update_sessions_updated_at`：任意 UPDATE 后刷新 `updated_at`；
- `update_session_message_count_on_insert/delete`：messages 增删时维护 `message_count`。

列表查询 `ListSessions` 仅返回 `parent_session_id IS NULL` 的根会话，按 `created_at DESC`。

领域结构 `session.Session` 字段与上表一一对应（`sql.NullString` 解包为 `string`）。

#### 4.1.2 `messages`

| 列 | 类型 | 含义 |
|----|------|------|
| `id` | TEXT PK | UUID |
| `session_id` | TEXT FK → sessions ON DELETE CASCADE | 所属会话 |
| `role` | TEXT | `user` / `assistant` / `system` / `tool` |
| `parts` | TEXT JSON 数组 | 多态内容块，默认 `[]` |
| `model` | TEXT 可空 | 助手消息使用的 `models.ModelID` |
| `created_at` / `updated_at` | INTEGER | |
| `finished_at` | INTEGER 可空 | 完成时间 |

索引：`idx_messages_session_id`。

#### 4.1.3 `files`（文件历史快照）

| 列 | 类型 | 含义 |
|----|------|------|
| `id` | TEXT PK | UUID |
| `session_id` | TEXT FK CASCADE | |
| `path` | TEXT | 绝对路径 |
| `content` | TEXT | 该版本全文 |
| `version` | TEXT | `initial` 或 `v1`、`v2`… |
| `created_at` / `updated_at` | INTEGER | |

唯一约束：`(path, session_id, version)`。索引：`session_id`、`path`。

`ListLatestSessionFiles`：按 path 取 `MAX(created_at)` 再回表，用于侧边栏。

> 注：`files.sql` 中 `ListNewFiles` 引用 `is_new` 列，初始迁移未创建该列；该查询当前未在 Go 服务层调用。

#### 4.1.4 sqlc 生成参数类型

- `CreateSessionParams`：`ID, ParentSessionID, Title, MessageCount, PromptTokens, CompletionTokens, Cost`
- `UpdateSessionParams`：`Title, PromptTokens, CompletionTokens, SummaryMessageID, Cost, ID`
- `CreateMessageParams` / `UpdateMessageParams`：对应 messages 列
- `CreateFileParams` / `UpdateFileParams`：对应 files 列
- `Querier` 接口聚合全部查询方法，便于测试替换

### 4.2 消息内容模型（`internal/message`）

`ContentPart` 接口以私有方法 `isPart()` 做封闭联合。JSON 持久化通过 `marshallParts` / `unmarshallParts` 带类型标签编码。

| 类型 | JSON 关键字段 | 用途 |
|------|---------------|------|
| `TextContent` | `text` | 用户/助手可见文本 |
| `ReasoningContent` | `thinking` | 推理模型思考增量 |
| `ImageURLContent` | `url`, `detail` | 图 URL |
| `BinaryContent` | `Path`, `MIMEType`, `Data` | 附件；发给 OpenAI 时转 `data:` URL，其它提供商为 raw base64 |
| `ToolCall` | `id,name,input,type,finished` | 助手发起的工具调用 |
| `ToolResult` | `tool_call_id,name,content,metadata,is_error` | 工具返回 |
| `Finish` | `reason,time` | 结束原因与 Unix 时间 |

`FinishReason` 枚举：`end_turn`、`max_tokens`、`tool_use`、`canceled`、`error`、`permission_denied`、`unknown`。

`MessageRole`：`assistant`、`user`、`system`、`tool`。

`message.Attachment`：`FilePath, FileName, MimeType, Content []byte`，TUI 文件选择器附加到下一条用户消息。

`CreateMessageParams`：`Role, Parts, Model`。

### 4.3 配置模型（`internal/config`）

| 类型 | 字段 | 说明 |
|------|------|------|
| `MCPType` | `stdio` / `sse` | MCP 传输 |
| `MCPServer` | `Command, Env, Args, Type, URL, Headers` | 一个 MCP 服务器 |
| `AgentName` | `coder` / `summarizer` / `task` / `title` | 四类 Agent |
| `Agent` | `Model, MaxTokens, ReasoningEffort` | 每 Agent 模型与限额；`ReasoningEffort` 用于 OpenAI：low/medium/high |
| `Provider` | `APIKey, Disabled` | 提供商开关 |
| `Data` | `Directory` | 数据目录，默认 `.opencode` |
| `LSPConfig` | `Disabled`（JSON tag 为 `"enabled"`，语义为禁用）、`Command, Args, Options` | LSP |
| `TUIConfig` | `Theme` | 默认 `opencode` |
| `ShellConfig` | `Path, Args` | 默认 `$SHELL` 或 `/bin/bash`，args `["-l"]` |
| `Config` | 上述全部 + `WorkingDir, Debug, DebugLSP, ContextPaths, AutoCompact` | 全局单例 `cfg` |

默认 `contextPaths`：

`.github/copilot-instructions.md`、`.cursorrules`、`.cursor/rules/`、`CLAUDE.md`、`CLAUDE.local.md`、以及多种大小写的 `opencode.md` / `OpenCode.md`。

### 4.4 LLM 与 Agent 模型

#### `models.Model`

| 字段 | 含义 |
|------|------|
| `ID` | 配置中使用的逻辑 ID，如 `claude-3.7-sonnet` |
| `Name` | 展示名 |
| `Provider` | `copilot/anthropic/openai/gemini/groq/openrouter/bedrock/azure/vertexai/xai/local/__mock` |
| `APIModel` | 发给厂商的模型名 |
| `CostPer1MIn/Out`、`CostPer1MInCached/OutCached` | 每百万 token 美元单价 |
| `ContextWindow` | 上下文窗口 |
| `DefaultMaxTokens` | 默认生成上限 |
| `CanReason` | 是否支持 reasoning/thinking |
| `SupportsAttachments` | 是否接受图片等附件 |

`ProviderPopularity` 决定模型对话框中提供商排序。`SupportedModels` 在 `init()` 中 `maps.Copy` 合并各厂商表。

#### `agent.AgentEvent`

| 字段 | 含义 |
|------|------|
| `Type` | `error` / `response` / `summarize` |
| `Message` | 完成时的助手消息 |
| `Error` | 错误 |
| `SessionID` | 压缩完成时的会话 |
| `Progress` | 压缩进度文案 |
| `Done` | 是否结束 |

`agent` 结构：嵌入 `pubsub.Broker[AgentEvent]`，持有 `sessions/messages/tools/provider/titleProvider/summarizeProvider`，`activeRequests sync.Map` 存 `sessionID → context.CancelFunc`（压缩键为 `sessionID+"-summarize"`）。

错误哨兵：`ErrRequestCancelled`、`ErrSessionBusy`。

#### Provider 事件

`EventType`：`content_start`、`tool_use_start`、`tool_use_delta`、`tool_use_stop`、`content_delta`、`thinking_delta`、`content_stop`、`complete`、`error`、`warning`。

`TokenUsage`：`InputTokens, OutputTokens, CacheCreationTokens, CacheReadTokens`。

`ProviderResponse`：`Content, ToolCalls, Usage, FinishReason`。

`ProviderEvent`：`Type, Content, Thinking, Response, ToolCall, Error`。

`maxRetries = 8`。

### 4.5 工具模型

| 类型 | 字段 |
|------|------|
| `ToolInfo` | `Name, Description, Parameters map[string]any, Required []string` |
| `ToolResponse` | `Type`（text/image）、`Content, Metadata, IsError` |
| `ToolCall` | `ID, Name, Input`（JSON 字符串） |
| `BaseTool` | `Info()`、`Run(ctx, ToolCall)` |

各工具参数结构：

| 结构 | 字段 |
|------|------|
| `BashParams` | `Command, Timeout` |
| `BashPermissionsParams` | 同左 |
| `BashResponseMetadata` | `StartTime, EndTime` |
| `GrepParams` | `Pattern, Path, Include, LiteralText` |
| `grepMatch` | `path, modTime, lineNum, lineText` |
| `GrepResponseMetadata` | `NumberOfMatches, Truncated` |
| `GlobParams` | `Pattern, Path` |
| `GlobResponseMetadata` | 截断等信息 |
| `LSParams` | `Path, Ignore []string` |
| `TreeNode` | `Name, Path, Type, Children` |
| `ViewParams` | `FilePath, Offset, Limit` |
| `EditParams` | `FilePath, OldString, NewString` |
| `EditPermissionsParams` | `FilePath, Diff` |
| `EditResponseMetadata` | `Diff, Additions, Removals` |
| `WriteParams` | `FilePath, Content` |
| `PatchParams` | 补丁文本相关 |
| `FetchParams` | `URL, Format, Timeout` |
| `SourcegraphParams` | `Query, Count, ContextWindow, Timeout` |
| `AgentParams` | `Prompt` |
| `fileRecord` | 工具内部读缓存 |

Context keys：`SessionIDContextKey`、`MessageIDContextKey`。

### 4.6 权限模型

`CreatePermissionRequest` / `PermissionRequest`：

`ID, SessionID, ToolName, Description, Action, Params, Path`

`Path` 在 `Request` 中被规范为目录（`filepath.Dir`；`.` 则用工作目录）。持久授权匹配四元组：`(SessionID, ToolName, Action, Path)`。

### 4.7 历史与 Diff 模型

`history.File`：`ID, SessionID, Path, Content, Version, CreatedAt, UpdatedAt`。

Diff：

| 类型 | 含义 |
|------|------|
| `LineType` | 增/删/上下文等 |
| `Segment` | 行内高亮片段 |
| `DiffLine` | 单行 diff |
| `Hunk` | hunk |
| `DiffResult` | 解析结果 |
| `linePair` | 并排配对 |
| `ParseConfig` / `SideBySideConfig` | 解析与渲染选项 |

Patch：

| 类型 | 含义 |
|------|------|
| `ActionType` | 更新/新增/删除 |
| `FileChange` | 单文件变更 |
| `Commit` | 一组文件变更 |
| `Chunk` | 补丁块 |
| `PatchAction` | 解析出的动作 |
| `Patch` | 完整补丁 |
| `DiffError` | 带路径/上下文的错误 |
| `Parser` | 行扫描解析器 |

### 4.8 PubSub

`EventType`：`created` / `updated` / `deleted`。

`Event[T]`：`Type, Payload T`。

`Broker[T]`：订阅者 map、buffer 64、非阻塞发送（满则丢弃）。

### 4.9 日志模型

`logging.LogMessage` 等：供 TUI 日志页展示；`Writer` 实现 slog handler 输出并发布事件。

### 4.10 TUI 关键状态模型

| 类型 | 职责 |
|------|------|
| `appModel` | 根：当前 page、dialogs、keyMap、自动压缩触发 |
| `chatPage` | 聊天页：editor + messages + sidebar |
| `messagesCmp` | 消息列表、渲染缓存、Agent 工作指示 |
| `editorCmp` | 多行输入、附件、外部编辑器 |
| `sidebarCmp` | 会话信息与文件变更统计 |
| `statusCmp` | token/cost/LSP 诊断 |
| `PermissionDialogCmp` | 允许 / 本会话允许 / 拒绝 |
| `SessionDialog` / `ModelDialog` / `ThemeDialog` / `CommandDialog` | 选择器 |
| `filepickerCmp` | 附件选择 |
| `completionDialogCmp` | `@` 补全 |
| `MultiArgumentsDialogCmp` | 自定义命令 `$NAME` 填参 |
| `InitDialogCmp` | 首次初始化 OpenCode.md |
| `helpCmp` / `quitCmp` | 帮助与退出确认 |
| 主题 `*Theme` | 实现 `theme.Theme` 接口的配色 |

### 4.11 其它

- `format.OutputFormat`：`text` / `json`
- `version.Version`：ldflags 注入，默认开发串
- `completions.CompletionItem`：标签与值
- LSP `Client`：进程、capabilities、diagnostics 缓存、handlers
- `WorkspaceWatcher`：fsnotify + LSP `didChangeWatchedFiles`

---

## 5. 功能设计

### 5.1 交互式编码助手（主功能）

用户在 TUI 编辑器输入自然语言（可带图片附件），发送后 Coder Agent 在当前会话中循环调用 LLM 与工具，直到模型给出非 tool_use 结束原因。侧边栏展示本会话修改过的文件及增删行数；状态栏展示 token 占用与费用。

### 5.2 非交互 Prompt 模式

`opencode -p` 创建标题为 `Non-interactive: {prompt前100字}` 的会话，`AutoApproveSession`，等待 `AgentEvent` 完成后 `format.FormatOutput` 打印。`-q` 关闭 spinner。

### 5.3 会话管理

- 新建：`Ctrl+N`（聊天页）或无会话时发送自动创建；
- 切换：快捷键打开会话对话框（实现为 `ctrl+s`，README 曾写 `Ctrl+A`，以代码为准）；
- 列表仅根会话；删除走 `session.Delete`（级联消息与文件）；
- 任务子会话不出现在列表，成本回写父会话。

### 5.4 多模型 / 多提供商

`Ctrl+O` 打开模型对话框，按 `ProviderPopularity` 分组。选择后 `CoderAgent.Update` → `config.UpdateAgentModel` 写回用户 JSON。Agent 忙碌时禁止切换。

默认模型探测顺序（`setProviderDefaults`）：Copilot → Anthropic → OpenAI → Gemini → Groq → OpenRouter → Bedrock → Azure → VertexAI → Local。

### 5.5 工具执行与权限

写文件、bash（非只读）、fetch、MCP execute、patch 等调用 `permissions.Request`。TUI 弹出权限对话框：

- `a` 允许一次；
- `A` 本会话同 tool/action/path 目录持久允许；
- `d` 拒绝 → Agent 将后续工具标为取消，FinishReason `permission_denied`。

非交互会话自动批准。

### 5.6 文件检索与阅读

`glob`、`grep`、`ls`、`view`、`sourcegraph` 供 Agent 探索代码。Grep 优先 ripgrep，失败则 Go 正则遍历。结果按 mtime 排序，上限 100。

### 5.7 文件修改

- `edit`：唯一子串替换；空 `old_string` 创建文件；空 `new_string` 删除内容；
- `write`：整文件覆盖；
- `patch`：专用补丁语法，模糊上下文匹配后应用；
- 成功后 `history.Create`/`CreateVersion`，并可通知 LSP 文档变更。

### 5.8 Shell

`bash` 使用 `PersistentShell` 单例，工作目录、环境变量跨调用保持。超时默认 60s，最大 10 分钟；输出截断 30000 字符。禁止 `curl/wget/nc` 等；只读命令可跳过部分权限提示（实现中仍可能请求权限，以 `Run` 内判断为准）。

### 5.9 子 Agent（Task）

`agent` 工具创建 `AgentTask`（仅 glob/grep/ls/sourcegraph/view），会话 ID = tool call ID，父会话累计 cost。用于不确定路径的探索，不可改文件。

### 5.10 MCP

配置 `mcpServers`。启动时 ListTools，工具名 `{server}_{tool}`。每次 Run 新建客户端：Initialize → CallTool → Close。stdio 与 SSE 两种传输。

### 5.11 LSP 与诊断

配置 `lsp` 映射启动语言服务器。`diagnostics` 工具聚合 `publishDiagnostics`。watcher 将文件系统事件转为 LSP 通知。`debugLSP` 控制详细日志。

### 5.12 上下文注入与项目记忆

系统提示并行读取 `contextPaths`（`sync.Once`）。首次运行可弹出 Init 对话框生成/更新 `OpenCode.md`。

### 5.13 会话压缩（Compact）

手动命令或自动：当 `promptTokens+completionTokens ≥ 0.95 * ContextWindow` 且 `autoCompact`。`Summarize` 用 summarizer 提供商生成摘要，写入同会话 Assistant 消息，设置 `SummaryMessageID`；下次 Run 从该消息截断并将 role 改为 User。

### 5.14 自定义命令

扫描：

- `$XDG_CONFIG_HOME/opencode/commands/` 或 `~/.opencode/commands/` → `user:` 前缀；
- `{project}/.opencode/commands/` → `project:` 前缀。

Markdown 文件名（含子目录）构成 ID。占位符 `$NAME`（`[A-Z][A-Z0-9_]*`）去重后弹出参数对话框替换。

内置命令：Initialize Project、Compact Session。

### 5.15 主题、帮助、日志、附件

主题：`opencode`、`catppuccin`、`gruvbox`、`monokai`、`dracula`、`flexoki`、`onedark`、`tokyonight`、`tron`。`Ctrl+T` 切换并 `UpdateTheme` 持久化。

帮助：`Ctrl+?` / `Ctrl+H`。日志页：`Ctrl+L`。附件：`Ctrl+F` 文件选择器；编辑器 `Ctrl+E` 打开 `$EDITOR`。

### 5.16 `@` 路径补全

`internal/completions` 提供文件/目录补全，供编辑器 `@` 触发。

---

## 6. 函数与方法说明书

下列覆盖仓库中全部业务 Go 函数（不含 LSP 协议生成文件中的编解码样板）。签名按源码，语义按实现。

### 6.1 `main`

| 函数 | 说明 |
|------|------|
| `main()` | `defer logging.RecoverPanic("main", ...)` 后 `cmd.Execute()` |

### 6.2 `cmd`

| 函数 | 说明 |
|------|------|
| `Execute()` | 执行 `rootCmd` |
| `rootCmd.RunE` | 见 §3.2 |
| `initMCPTools(ctx, app)` | 30s 超时 goroutine 调用 `agent.GetMcpTools` |
| `setupSubscriptions(app, ctx)` | 合并多个服务 Subscribe 到 `tea.Msg` channel |
| `attemptTUIRecovery(program)` | panic 后 `program.Quit()` |
| `init()` | 注册 flags |

`cmd/schema.main`：`generateSchema()` 遍历 `SupportedModels` 输出 Draft-07 JSON Schema 到 stdout。

### 6.3 `internal/app`

| 函数 | 说明 |
|------|------|
| `New(ctx, conn)` | 装配服务、主题、后台 LSP、CoderAgent |
| `initTheme()` | `theme.SetTheme(cfg.TUI.Theme)` |
| `RunNonInteractive(ctx, prompt, outputFormat, quiet)` | 创建会话、自动批准、Run、格式化输出 |
| `Shutdown()` | 取消 watcher、关闭 LSP |
| `initLSPClients` / `createAndStartLSPClient` 等（`lsp.go`） | 按配置拉起客户端与 watcher |

### 6.4 `internal/config`

| 函数 | 说明 |
|------|------|
| `Load(workingDir, debug)` | 单例加载；合并全局+本地；校验；强制 title agent maxTokens=80 |
| `configureViper()` | 配置名 `.opencode`、JSON、HOME/XDG 路径、`OPENCODE_` 环境前缀 |
| `setDefaults(debug)` | data.directory、contextPaths、theme、autoCompact、shell |
| `setProviderDefaults()` | 按凭证填充 providers 与默认 agent 模型 |
| `hasAWSCredentials` / `hasVertexAICredentials` / `hasCopilotCredentials` | 凭证探测 |
| `readConfig` / `mergeLocalConfig` | 读入并 merge `workingDir/.opencode.json` |
| `applyDefaultValues` | 补全空字段 |
| `validateAgent` / `Validate` | 模型存在、提供商未禁用、maxTokens 与窗口关系 |
| `getProviderAPIKey` | 从 cfg 或环境取 key |
| `setDefaultModelForAgent` | 按流行度选第一个可用模型 |
| `updateCfgFile` | 读改写用户配置文件 |
| `Get` / `WorkingDirectory` | 访问单例 |
| `UpdateAgentModel` / `UpdateTheme` | 热更新并落盘 |
| `LoadGitHubToken` | 从 env、cfg 或 `~/.config/github-copilot/{hosts,apps}.json` 读取 |
| `ShouldShowInitDialog` / `MarkProjectInitialized`（`init.go`） | 项目初始化标记 |

### 6.5 `internal/db`

| 函数 | 说明 |
|------|------|
| `Connect()` | 打开 SQLite、WAL、goose Up |
| `New(db)` | 构造 `Queries` |
| `Queries` 各方法 | 与 `sql/*.sql` 同名：`CreateSession`、`GetSessionByID`、`ListSessions`、`UpdateSession`、`DeleteSession`；`CreateMessage`、`GetMessage`、`ListMessagesBySession`、`UpdateMessage`、`DeleteMessage`、`DeleteSessionMessages`；`CreateFile`、`GetFile`、`GetFileByPathAndSession`、`ListFilesBySession`、`ListFilesByPath`、`UpdateFile`、`DeleteFile`、`DeleteSessionFiles`、`ListLatestSessionFiles`、`ListNewFiles` |
| embed | `//go:embed migrations` |

### 6.6 `internal/session`

| 方法 | 说明 |
|------|------|
| `NewService(q)` | 带 Broker 的服务 |
| `Create(ctx, title)` | UUID 根会话 |
| `CreateTaskSession(ctx, toolCallID, parent, title)` | ID=toolCallID |
| `CreateTitleSession(ctx, parent)` | ID=`title-{parent}` |
| `Get` / `List` / `Save` / `Delete` | CRUD + 发布事件 |
| `fromDBItem` | Null 字段转换 |

### 6.7 `internal/message`

服务：`NewService`、`Create`、`Update`、`Get`、`List`、`Delete`、`DeleteSessionMessages`、`fromDBItem`、`marshallParts`、`unmarshallParts`。

`Message` 方法：`Content`、`ReasoningContent`、`ImageURLContent`、`BinaryContent`、`ToolCalls`、`ToolResults`、`IsFinished`、`FinishPart`、`FinishReason`、`IsThinking`、`AppendContent`、`AppendReasoningContent`、`FinishToolCall`、`AppendToolCallInput`、`AddToolCall`、`SetToolCalls`、`AddToolResult`、`SetToolResults`、`AddFinish`、`AddImageURL`、`AddBinary`。

各 `ContentPart.String` / `isPart` 见 §4.2。

### 6.8 `internal/history`

`NewService(q, db)`、`Create`（version=`initial`）、`CreateVersion`（`vN`，UNIQUE 冲突重试）、`createWithVersion`、`Get`、`GetByPathAndSession`、`ListBySession`、`ListLatestSessionFiles`、`Update`、`Delete`、`DeleteSessionFiles`、`fromDBItem`。

### 6.9 `internal/permission`

`NewPermissionService`、`Request`（自动批准 / 持久匹配 / 阻塞 channel）、`Grant`、`GrantPersistant`（拼写按源码）、`Deny`、`AutoApproveSession`。

### 6.10 `internal/pubsub`

`NewBroker`、`NewBrokerWithOptions`、`Subscribe`、`Publish`、`Shutdown`、`GetSubscriberCount`。

### 6.11 `internal/llm/agent`

| 函数 | 说明 |
|------|------|
| `NewAgent` | 按 agentName 创建 provider；coder 额外 title+summarizer |
| `Model` | 当前模型 |
| `Cancel` | 取消 Run 与 summarize |
| `IsBusy` / `IsSessionBusy` | 扫描 activeRequests |
| `generateTitle` | titleProvider.SendMessages，去换行后 Save 标题 |
| `err` | 包装 AgentEvent error |
| `Run` | 忙则 `ErrSessionBusy`；丢弃不支持附件；goroutine `processGeneration` |
| `processGeneration` | 标题、摘要截断、用户消息、stream 循环 |
| `createUserMessage` | 持久化 user parts |
| `streamAndHandleEvents` | 流式落库 + 顺序执行工具 |
| `finishMessage` | AddFinish + Update |
| `processEvent` | 映射 ProviderEvent 到消息字段与 TrackUsage |
| `TrackUsage` | 四段计价累加 Cost，更新 token 字段 |
| `Update` | 忙则失败；更新配置并重建 provider |
| `Summarize` | 异步摘要流水线 |
| `createAgentProvider` | 模型/提供商/系统提示/推理选项工厂 |
| `CoderAgentTools` / `TaskAgentTools` | 工具列表 |
| `mcpTool.Info/Run`、`runTool`、`NewMcpTool`、`getTools`、`GetMcpTools` | MCP |
| `agentTool.Info/Run`、`NewAgentTool` | 子 Agent |

### 6.12 `internal/llm/provider`

公共：`NewProvider`、`cleanMessages`（去掉空 parts）、`SendMessages`、`StreamResponse`、`Model`、以及 `WithAPIKey/WithModel/WithMaxTokens/WithSystemMessage/With*Options`。

各客户端共同模式：`convertMessages`、`convertTools`、`finishReason`、`send`、`stream`、`shouldRetry`、`toolCalls`、`usage`。

Anthropic 特有：`preparedMessages`（对尾部消息设 ephemeral cache）、`DefaultShouldThinkFn`、`WithAnthropicShouldThinkFn`、`WithAnthropicBedrock`、`WithAnthropicDisableCache`。

OpenAI：`WithOpenAIBaseURL`、`WithReasoningEffort`、Groq/OpenRouter/xAI/Local 均复用 openaiClient 改 base URL。

Copilot：`exchangeGitHubToken`、Anthropic 模型走不同消息转换。

Gemini：JSON Schema → `genai.Schema` 递归转换。

Azure / VertexAI / Bedrock：薄封装对应客户端。

### 6.13 `internal/llm/models`

各文件定义 `*Models` map 与 `Provider*` 常量；`init` 合并到 `SupportedModels`。`local.go` 可从 `LOCAL_ENDPOINT` 动态发现。

### 6.14 `internal/llm/prompt`

`GetAgentPrompt(name, provider)` 分发 coder/task/title/summarizer。

`CoderPrompt`：Anthropic/OpenAI 两套基座 + `getEnvironmentInfo` + `lspInformation`。

`getContextFromPaths` / `processContextPath`：读 contextPaths。

`TitlePrompt`、`TaskPrompt`、`SummarizerPrompt`：短系统提示。

### 6.15 `internal/llm/tools`

公共：`NewTextResponse`、`NewTextErrorResponse`、`WithResponseMetadata`、`GetContextValues`。

每个工具：`NewXxxTool`、`Info`、`Run`。

辅助：

- bash：`bashDescription`、禁止列表匹配；
- grep：`escapeRegexPattern`、`searchFiles`、`searchWithRipgrep`、`searchFilesWithRegex`、`fileContainsPattern`、`globToRegex`；
- glob：doublestar 搜索；
- ls：`shouldSkip`、`createFileTree`、`printTree`、`listDirectory`；
- view：`addLineNumbers`、`readTextFile`、`isImageFile`、`LineScanner`；
- edit/write/patch：权限、diff 统计、history、LSP notify；
- fetch：`extractTextFromHTML`、`convertHTMLToMarkdown`；
- sourcegraph：GraphQL + `formatSourcegraphResults`；
- diagnostics：汇总 LSP 诊断；
- file.go：读文件记录辅助。

`shell.GetPersistentShell`、`Exec`、`Close`、`processCommands`、`execCommand`、`killChildren`、`shellQuote` 等。

### 6.16 `internal/diff`

解析与渲染：`ParseUnifiedDiff`、`HighlightIntralineChanges`、`pairLines`、`SyntaxHighlight`、`RenderSideBySideHunk`、`FormatDiff`、`GenerateDiff`、样式辅助函数。

补丁：`TextToPatch`、`IdentifyFilesNeeded/Added`、`getUpdatedFile`、`PatchToCommit`、`AssembleChanges`、`LoadFiles`、`ApplyCommit`、`ProcessPatch`、`OpenFile`、`WriteFile`、`RemoveFile`、`ValidatePatch`、`findContext*`、`peekNextSection`、`Parser.Parse` 等。

### 6.17 `internal/fileutil`

`GetRgCmd`、`GetFzfCmd`、`SkipHidden`、`GlobWithDoublestar`。

### 6.18 `internal/format`

`OutputFormat.String`、`Parse`、`IsValid`、`GetHelpText`、`FormatOutput`、`formatAsJSON`。

`Spinner.Start/Stop` 及 tea Model 三件套。

### 6.19 `internal/logging`

`Info/Debug/Warn/Error` 及 `*Persist`；`RecoverPanic`；`GetSessionPrefix`；`AppendToSessionLogFile`；`WriteRequestMessageJson`、`WriteRequestMessage`、`AppendToStreamSessionLog*`、`WriteChatResponseJson`、`WriteToolResultsJson`；`NewWriter`。

### 6.20 `internal/lsp`（应用侧）

`Client` 连接、Initialize、Shutdown、文档同步、诊断存储、`methods.go` 中各 LSP 方法封装、`handlers.go` 处理 server 通知、`transport.go` JSON-RPC、`language.go` 语言映射、`watcher` 递归监视。

`util/edit.go`：将 LSP TextEdit 应用到缓冲区。

### 6.21 `internal/tui`

根：`New`、`Init`、`Update`、`View`、`RegisterCommand`、`findCommand`、`moveToPage`。`Update` 处理全局键、权限、agent 完成、自动 compact。

页面与组件均实现 Bubble Tea `Init/Update/View`，以及 `SetSize/GetSize` 布局接口。关键业务方法：

- `chat.sendMessage` → `CoderAgent.Run`；
- `editorCmp.send` / `openEditor`；
- `messagesCmp.renderView` / `working` / `IsAgentWorking`；
- `sidebarCmp.loadModifiedFiles` / `processFileChanges` / `findInitialVersion`；
- `statusCmp` token 百分比；
- dialog 选择后发出 `ModelSelectedMsg`、`CommandRunCustomMsg` 等。

`theme`：`RegisterTheme`、`SetTheme`、`GetTheme`、`CurrentTheme`、`AvailableThemes`、各 `NewXxxTheme`。

`layout`：split、overlay、container。

`util.CmdHandler`、`ReportError/Info/Warn`、`Clamp`。

### 6.22 `internal/completions`

文件/文件夹补全 provider：列出相对工作目录的路径供 `@` 使用。

### 6.23 `internal/version`

`init` 设置默认 Version。

---

## 7. 算法设计

### 7.1 Agent 主循环

**输入**：sessionID、用户文本、附件。  
**输出**：`AgentEvent`（成功含最终 Assistant Message）。

伪代码：

```
if IsSessionBusy: return ErrSessionBusy
store cancel in activeRequests
msgs = List(session)
if empty: async generateTitle
if SummaryMessageID: slice from that index; msgs[0].Role = User
userMsg = Create(User)
history = msgs + userMsg
loop:
  if ctx cancelled: return canceled
  assistant, toolMsg, err = streamAndHandleEvents(history)
  if err: return
  if assistant.FinishReason == tool_use and toolMsg != nil:
    history += assistant + toolMsg
    continue
  return Response(assistant)
```

工具执行算法：对 `assistant.ToolCalls()` **顺序** 执行；找不到工具 → 错误结果；`ErrorPermissionDenied` → 当前与后续全部错误结果并 `FinishReasonPermissionDenied`；取消 → 剩余工具 “canceled by user”。

**复杂度**：循环次数由模型 tool_use 轮数决定；每轮一次 LLM 调用 + O(T) 工具，T 为该轮 tool call 数。无并行 tool execution。

### 7.2 流式事件归并

Provider 推送 delta：

- `EventThinkingDelta` → `AppendReasoningContent` + Update DB  
- `EventContentDelta` → `AppendContent` + Update DB  
- `EventToolUseStart` → `AddToolCall`  
- `EventToolUseStop` → `FinishToolCall`  
- `EventComplete` → `SetToolCalls`、`AddFinish`、`TrackUsage`  
- `EventError` → 上抛（Canceled 转 context.Canceled）

每次 delta 写库，TUI 通过 message updated 事件重绘。这是简单的 **append-only 归并**，不是差分压缩。

### 7.3 费用计算

```
cost = CostPer1MInCached/1e6 * CacheCreationTokens
     + CostPer1MOutCached/1e6 * CacheReadTokens
     + CostPer1MIn/1e6 * InputTokens
     + CostPer1MOut/1e6 * OutputTokens
sess.Cost += cost
sess.CompletionTokens = OutputTokens + CacheReadTokens
sess.PromptTokens     = InputTokens + CacheCreationTokens
```

注意：token 字段被 **覆盖为最近一轮**，不是会话累计（cost 才累计）。自动压缩阈值使用这两个字段之和。

### 7.4 自动压缩触发

在 TUI 收到 Agent 完成事件后：

```
tokens = session.PromptTokens + session.CompletionTokens
if tokens >= 0.95 * model.ContextWindow && config.AutoCompact:
    CoderAgent.Summarize(sessionID)
```

Summarize 算法：List 全量消息 + 固定英文摘要指令 → summarizer `SendMessages`（无工具）→ 新建 Assistant 文本消息 → 设置 `SummaryMessageID`，`PromptTokens=0`，累加摘要 cost。

下次对话：从摘要消息截断，并把该条 role 改为 User，以满足部分 API「首条须为 user」的约束。

### 7.5 标题生成

会话第一条用户消息时异步调用 title agent（maxTokens=80），将回复去换行 `TrimSpace` 后写入 `session.Title`。失败只记日志。

### 7.6 Grep

1. 若 `literal_text`：按 `\.+*?()[]{}^$|` 顺序转义（先转义反斜杠）；  
2. `searchFiles(pattern, path, include, limit=100)`：  
   - 优先 `searchWithRipgrep`（`fileutil.GetRgCmd`）；  
   - 失败则 `filepath.Walk` + `regexp`，跳过隐藏文件；  
3. 按 `modTime` 降序；截断超限。

`globToRegex` 将 include glob 转为正则过滤文件名。

### 7.7 Glob

`fileutil.GlobWithDoublestar`：doublestar 匹配，跳过隐藏路径，按 mtime 排序，limit 100，返回 truncated 标志。

### 7.8 LS 树构建

`listDirectory` BFS/递归收集路径，`shouldSkip`：

- 任意路径分段以 `.` 开头；
- 包含 `__pycache__`；
- 匹配 ignore glob。

`createFileTree` 按路径分段插入 `TreeNode`；`printTree` 缩进文本。超过条目上限设 truncated。

### 7.9 View 分页读

`readTextFile`：`LineScanner` 逐行读，从 `offset` 起最多 `limit` 行；`addLineNumbers` 生成 `     N|content` 形式。图片扩展名走 `isImageFile`，不按文本读。可附加 LSP 诊断片段。

### 7.10 Edit 唯一替换

读入全文：

- `old=="" && new!=""`：创建文件（需权限）；  
- `new==""`：删除 `old` 出现处内容；  
- 否则：`strings.Count(old)` 必须为 1，否则错误；`strings.Replace` 一次。

生成 unified diff 统计 additions/removals，写入 history 版本，可选 LSP didChange。

### 7.11 Patch 模糊上下文匹配

`findContextCore` 三级 matcher：

1. 精确相等 fuzz=0；  
2. `TrimRight` 空白 fuzz=1；  
3. `TrimSpace` fuzz=100。

EOF 块先从文件尾匹配，失败再从 `start` 匹配并 +10000 fuzz。`Parser` 扫描 `*** Begin Patch` 风格分节，产出 `PatchAction`，`ApplyCommit` 调 write/remove 回调。

这是 **带 fuzz 代价的线性扫描匹配**，最坏 O(n*m)（文件行 × 上下文行）。

### 7.12 Diff 并排渲染

`ParseUnifiedDiff` 按 `@@` hunk 头解析；`HighlightIntralineChanges` 用 LCS/sergi-diff 在配对的删/增行上标 Segment；`pairLines` 将删行与增行配对；Chroma 按文件名语法高亮；Lipgloss 主题色输出左右栏。

`GenerateDiff`：对 before/after 调 go-udiff 或等价库生成 unified 文本，并计数增删。

### 7.13 Persistent Shell

单例按 workingDir 创建。内部命令队列 + 一个长期 bash 进程：

- `Exec` 投递 `commandExecution`，等待 `commandResult` 或 ctx/timeout；  
- `execCommand` 写入 stdin，读取 stdout/stderr 直到完成标记；  
- 超时 `killChildren`；  
- `shellQuote` 防注入拼接。

状态（cwd、env）留在子进程中，故跨工具调用持久。

### 7.14 Bash 安全策略

- 命令子串命中 `bannedCommands`（curl/wget/nc 等）拒绝；  
- `safeReadOnlyCommands` 前缀视为只读（git status、go test 等）；  
- 仍对写操作走权限；  
- `Timeout` 钳制到 `[0, MaxTimeout]`，默认 60000ms；  
- 输出 > 30000 字符截断并提示。

### 7.15 Fetch

HTTP GET，超时可配。`format`：

- 原始文本；  
- `extractTextFromHTML`（goquery 去脚本）；  
- `convertHTMLToMarkdown`。

需权限 action 通常为 fetch/execute。

### 7.16 Sourcegraph

向 Sourcegraph GraphQL 发搜索；`count`、`context_window` 限制命中与代码片段行窗；超时。结果格式化为路径+行号+窗口。

### 7.17 MCP 发现与调用

`GetMcpTools`：若全局切片非空直接返回（进程级缓存，不热更新）。否则遍历配置：stdio/SSE 建客户端 → Initialize → ListTools → 包装 `mcpTool`。

`Run`：权限 execute → 再新建客户端（不复用发现连接）→ Initialize → CallTool → 拼接 TextContent。

### 7.18 子 Agent

同步 `<-done` 等待 task agent 整段循环结束，只把最终 Assistant 文本返回父模型。父 `Cost += child.Cost`。无流式中间结果给父模型。

### 7.19 权限匹配

```
if session in autoApprove: allow
pathDir = Dir(path) or WorkingDirectory
if exists grantPersistent with same (session, tool, action, pathDir): allow
else publish PermissionRequest and wait channel
```

无超时：TUI 不响应则工具永久阻塞（设计风险）。

### 7.20 PubSub 投递

RLock 复制订阅者列表后，对每个 channel `select { case ch <- ev: default: }`。慢订阅者丢事件。TUI 桥另有发送超时（约 2s，见 `cmd/root.go` 订阅循环）。

### 7.21 配置合并与默认模型

Viper：环境 `OPENCODE_*` 自动映射；先读 HOME/XDG 全局文件，再 merge 项目 `.opencode.json`。`setProviderDefaults` 检查各厂商 env/文件凭证，未禁用则写入 `cfg.Providers`，并为四个 Agent 选择第一个可用模型。`Validate` 确保引用的 ModelID 存在且提供商启用。

### 7.22 提供商重试

`shouldRetry`：对 429/5xx/网络错误指数退避，最多 8 次。具体 backoff 各客户端略有差异（Copilot 含 token 刷新）。

### 7.23 Anthropic Prompt Cache

`preparedMessages` 对接近末尾的消息设置 `cache_control: ephemeral`，以降低长对话输入成本。`TrackUsage` 用 CacheCreation/CacheRead 单价计费。

### 7.24 自定义命令参数提取

正则 `\$([A-Z][A-Z0-9_]*)` 全局匹配，用 map 去重保持出现顺序，对话框收集后再 `strings.ReplaceAll("$NAME", value)`。

### 7.25 文件历史版本号

`CreateVersion`：读最新 version，若 `initial` 则 `v1`，否则解析 `vN` 得 `vN+1`。UNIQUE 冲突则递增重试，避免并发写同一 path。

### 7.26 `@` 补全

扫描工作区文件/目录（跳过隐藏），fuzzy 过滤编辑器当前 token，返回路径 CompletionItem。

### 7.27 LSP 诊断聚合

客户端 map 按 URI 存 `[]Diagnostic`。`diagnostics` 工具按可选 file_path 过滤，格式化 severity+行+消息。watcher 防抖后发 `workspace/didChangeWatchedFiles`。

---

## 8. 测试设计与用例清单

仓库测试文件 4 个，均以 `testing` + `testify` 为主。无 Agent 循环、无 Provider 集成测试（`ProviderMock` 注释为测试用但未实现完整行为）。

### 8.1 `internal/llm/tools/ls_test.go`

| 用例 | 意图 |
|------|------|
| `TestLsTool_Info` | Name 为 LS 工具名；Description 非空；Parameters 含 `path`、`ignore`；Required 含 `path` |
| `TestLsTool_Run/lists directory successfully` | 临时树中可见 dir/file 出现；隐藏项与 `__pycache__` 不出现 |
| `.../handles non-existent path` | 响应包含 `path does not exist` |
| `.../handles empty path parameter` | 空 path 仍返回非空 Content（回落到工作目录） |
| `.../handles invalid parameters` | Input 非 JSON → `error parsing parameters` |
| `.../respects ignore patterns` | ignore `file1.txt`、`dir1` 后树中无 `- file1.txt`、`- dir1/` |
| `.../handles relative path` | chdir 到父目录后用 base name 仍列出内容 |
| `TestShouldSkip/hidden file` | `.hidden_file` → true |
| `.../hidden directory` | `.hidden_dir` → true |
| `.../pycache directory` | `__pycache__/file.pyc` → true |
| `.../node_modules directory` | 纯路径含 node_modules **不** skip（expected false） |
| `.../normal file` / `normal directory` | false |
| `.../ignored by pattern` | `ignore_*.txt` 匹配 → true |
| `.../not ignored by pattern` | `keep_me.txt` → false |
| `TestCreateFileTree` | 四路径合成单根 `path/to`，children 3 个，`dir1` 下 2 子节点 |
| `TestPrintTree` | 缩进树字符串含 `/root/`、`dir1/`、嵌套 file |
| `TestListDirectory/lists files with no limit` | 可见路径在、隐藏不在、truncated=false |
| `.../respects limit and returns truncated flag` | limit=2 → len=2 且 truncated |
| `.../respects ignore patterns` | `*.txt` 无 txt 后缀；dir1 仍在 |

### 8.2 `internal/llm/prompt/prompt_test.go`

| 用例 | 意图 |
|------|------|
| `TestGetContextFromPaths` | `config.Load(tmp)` 后设置 ContextPaths 为 `file.txt` 与 `directory/`；创建四文件；`getContextFromPaths` 输出按 `# From:{abs}` 拼接的固定字符串（顺序：file.txt，directory 下 a/b/c） |

辅助 `createTestFiles`：以 `/` 结尾 mkdir，否则写 `{path}: test content`。

### 8.3 `internal/tui/theme/theme_test.go`

| 步骤 | 断言 |
|------|------|
| `TestThemeRegistration` | `AvailableThemes` 含 catppuccin、gruvbox、monokai；`GetTheme` 非 nil；`SetTheme("gruvbox")` 后 `CurrentThemeName` 为 gruvbox；再切 monokai；最后恢复 original |

### 8.4 `internal/tui/components/dialog/custom_commands_test.go`

`TestNamedArgPattern` 表驱动：

| input | expected 去重名 |
|-------|-----------------|
| `$ARGUMENTS` | ARGUMENTS |
| `$FOO` and `$BAR` | FOO, BAR |
| `$FOO_BAR` `$BAZ123` | FOO_BAR, BAZ123 |
| 无占位符 | 空 |
| `$FOO` 两次 | 仅 FOO |
| `$1INVALID` | 空（数字开头非法） |

`TestRegexPattern`：合法 `$FOO/$BAR/$FOO_BAR/$BAZ123/$ARGUMENTS` 匹配；非法 `$foo/$1BAR/$_FOO/FOO/$` 不匹配。

### 8.5 建议但尚未存在的测试（缺口，非当前代码）

Agent 取消与忙锁、权限四元组、edit 唯一性失败、patch fuzz 三级、TrackUsage 公式、摘要截断、Grep rg 回退、MCP 命名、配置 Validate。实现这些不属于本说明书范围，仅标明覆盖空洞。

---

## 9. 接口与协议

### 9.1 CLI

```
opencode [--debug|-d] [--cwd|-c DIR] [--prompt|-p TEXT]
         [--output-format|-f text|json] [--quiet|-q] [--version|-v]
```

### 9.2 配置 JSON Schema

`opencode-schema.json` 由 `go run ./cmd/schema` 生成。`$schema` 可在 `.opencode.json` 引用以便 IDE 校验。

### 9.3 MCP

客户端实现 MCP Initialize / ListTools / CallTool；stdio 与 SSE。ClientInfo Name=`OpenCode`，Version=`version.Version`。

### 9.4 LSP

JSON-RPC 2.0 stdio。应用侧使用子集：initialize、initialized、shutdown/exit、textDocument/didOpen|didChange|didClose、workspace/didChangeWatchedFiles、textDocument/publishDiagnostics。协议类型生成自 TypeScript LSP spec。

### 9.5 LLM

- Anthropic Messages Streaming + tools + thinking + cache_control  
- OpenAI Chat Completions（含 Groq/OpenRouter/Azure/xAI/Local/Copilot 变体）  
- Gemini `google.golang.org/genai`  
- Bedrock 上的 Anthropic 模型 ID  

---

## 10. 存储、并发与可靠性

- SQLite 单连接；sqlc 方法接受 `context.Context`。  
- Agent 每会话互斥（`activeRequests`）；多会话可并行。  
- 工具顺序执行，无内部超时包装（依赖工具自身与 ctx）。  
- Panic：main、agent.Run、MCP goroutine、TUI handler 均 `RecoverPanic`。  
- 关闭：LSP 5s shutdown；shell `Close`；broker `Shutdown`。  

---

## 11. 安全设计

| 机制 | 行为 |
|------|------|
| 权限对话框 | 写/执行类工具默认人工确认 |
| bash 黑名单 | 拦截常见下载/网络客户端 |
| 工作目录 | 工具路径相对 `config.WorkingDirectory` |
| API Key | 配置文件与环境变量，不入库 |
| 非交互自动批准 | 文档明确，适合受控脚本，不适合不可信 prompt |
| MCP | 每次调用仍走权限 |

已知限制：无 OS sandbox、无网络策略、权限 channel 无超时、项目 `.opencode.json` 可覆盖全局（恶意仓库配置风险）。

---

## 12. 构建、发布与质量门禁

- Go 1.24，`go build -o opencode`  
- `.goreleaser.yml` 多平台发布  
- `scripts/release` semver tag；`scripts/snapshot` 快照  
- GitHub Actions：`build.yml`（snapshot）、`release.yml`  
- `scripts/check_hidden_chars.sh` 隐藏 Unicode 检查  
- `sqlc.yaml` 生成 `internal/db`  

---

## 13. 设计约束与归档说明

README 标明仓库已归档，演进迁移至 Charm **Crush**。本说明书描述的是本树源代码的实际行为（例如会话压缩发生在同一 session 上设置 `SummaryMessageID`，而 README 有一处“创建新 session”的过时表述；会话切换快捷键以 `internal/tui/tui.go` 的 `ctrl+s` 为准）。

---

## 14. 模块依赖关系（编译）

```
main → cmd → app, config, db, tui, agent, format, logging, version
app  → session, message, history, permission, agent, lsp, theme, format
agent → provider, tools, prompt, models, session, message, permission, pubsub
tools → permission, history, lsp, diff, fileutil, shell, config
tui   → app 的服务与 agent 事件
provider → models, message, tools
```

禁止 UI 直接写 SQL；禁止 tools 依赖 tui。该分层在源码中总体成立。

---

*（完）本文与源码同步于工作区当前 `main` 快照。*
