# OpenCode 源代码说明

| 项目 | 内容 |
|------|------|
| 文档名称 | 源代码说明 |
| 模块路径 | `github.com/opencode-ai/opencode` |
| 统计基准日期 | 2026-09-20 |
| 统计范围 | 工作区全部受控文件（不含 `.git` 对象） |

---

## 1. 源代码整体介绍

### 1.1 项目定位

OpenCode 是用 **Go** 编写的终端 AI 编程助手。它在本地以单一可执行文件运行，通过 Cobra 提供 CLI，通过 Charmbracelet Bubble Tea 提供全屏 TUI，并在内部实现完整的 Agent 循环（流式 LLM 调用 + 工具执行 + SQLite 会话持久化）。

仓库 README 声明项目已归档，后续产品形态迁移至 Charm 团队的 Crush。本说明仅描述本仓库快照中的源代码。

### 1.2 编程语言

| 类别 | 语言 / 格式 | 用途 |
|------|-------------|------|
| 主语言 | Go 1.24.0（`go.mod` 的 `go 1.24.0`） | 几乎全部业务逻辑 |
| SQL | SQLite 方言 | sqlc 查询与 goose 迁移 |
| JSON | JSON / JSON Schema Draft-07 | 用户配置与 schema |
| YAML | GitHub Actions、Goreleaser、sqlc.yaml | CI/CD 与代码生成 |
| Shell | Bash | `install`、`scripts/release`、`scripts/snapshot`、隐藏字符检查 |
| Markdown | Markdown | README、命令模板（运行时用户侧） |

**没有** TypeScript/JavaScript 前端、没有 Python 服务端。TUI 全部在进程内用 Lipgloss/Bubble Tea 绘制。LSP 协议类型是从 TypeScript LSP 规范生成的 **Go** 代码（`internal/lsp/protocol/`）。

### 1.3 开发工具与生成器

| 工具 | 作用 |
|------|------|
| Go toolchain ≥ 1.24 | 编译、测试、`go install` |
| Cobra | CLI |
| Viper | 配置 |
| sqlc v1.29.0 | 从 `internal/db/sql/*.sql` 生成 `*.sql.go`、`models.go`、`querier.go` |
| goose v3 | 嵌入式迁移 |
| Goreleaser | 多平台发布（`.goreleaser.yml`） |
| GitHub Actions | `build.yml` / `release.yml` |
| Charmbracelet 套件 | bubbletea、bubbles、lipgloss、glamour |
| testify | 单元测试断言 |
| 编辑器 | 任意；项目含 `.opencode.json` 示例（为本工具自身配置 LSP gopls） |

主要第三方库（直接依赖，见 `go.mod`）：

- LLM：`anthropic-sdk-go`、`openai-go`、`google.golang.org/genai`、Azure Identity
- MCP：`mark3labs/mcp-go`
- 存储：`ncruces/go-sqlite3`（Wazero 解释 WASM SQLite）
- 终端：Charm 全家桶、`bubblezone`、`glamour`
- 其它：`doublestar`、`chroma`、`go-diff`、`html-to-markdown`、`goquery`、`fsnotify`、`uuid`

### 1.4 规模统计

统计命令等价于：对仓库内文件 `find` 后 `wc -l`（排除 `.git`）。

| 指标 | 数量 |
|------|------|
| 仓库文件总数（不含 `.git`） | **162** |
| Go 源文件（`*.go`） | **140** |
| SQL 文件 | **5**（2 个迁移 + 3 个 sqlc 查询） |
| 工作流 / Goreleaser YAML | **3** |
| Shell 脚本（含 `install`、`scripts/*`） | **4** |
| JSON（配置示例 + schema） | **2** |
| Markdown（README + cmd/schema/README + 协议 LICENSE 旁文档） | 少量文档；业务 md 不计入“运行时源码” |
| Go 代码总行数 | **42,176** |
| SQL 总行数 | **270** |
| 业务向“源文件”行数（Go+SQL+JSON+YAML+脚本等，不含 README/LICENSE/go.sum） | 约 **43,441** |

行数高度集中在 LSP 协议生成代码：

| 文件 | 行数 | 备注 |
|------|------|------|
| `internal/lsp/protocol/tsprotocol.go` | 6,907 | LSP 规范类型 |
| `internal/lsp/protocol/tsjson.go` | 3,072 | JSON 编解码 |
| 其余约 138 个 `.go` 文件合计 | ≈ 32,197 | 应用逻辑 + TUI + 工具 |

**若只计“手写业务代码”**（排除 `tsprotocol.go` 与 `tsjson.go`）：Go 约 **32,197** 行。

测试代码：4 个 `*_test.go`，合计约 706 行（`ls_test.go` 456 + `custom_commands_test.go` 105 + `theme_test.go` 88 + `prompt_test.go` 57）。

### 1.5 目录结构总览

```
.
├── main.go                 # 进程入口
├── go.mod / go.sum
├── sqlc.yaml
├── opencode-schema.json    # 配置 JSON Schema
├── .opencode.json          # 本仓库示例配置
├── .goreleaser.yml
├── install                 # 一键安装脚本
├── cmd/                    # CLI
│   ├── root.go
│   └── schema/             # schema 生成器
├── internal/               # 全部业务包
├── scripts/
└── .github/workflows/
```

`internal/` 是唯一的库代码根，保证外部模块不能 import 内部实现。

### 1.6 构建产物

```bash
go build -o opencode
# 或
go install github.com/opencode-ai/opencode@latest
```

版本字符串由 `internal/version.Version` 提供，发布时通过 ldflags 注入（Goreleaser）。

---

## 2. 源代码文件清单

下列清单按目录分组。**主要功能简介**描述该文件在系统中的职责，而非重复函数名（函数级说明见 `1DESIGN.md`）。

### 2.1 根目录

| 文件名 | 所在目录 | 主要功能简介 |
|--------|----------|--------------|
| `main.go` | `/` | 程序入口：panic 恢复后调用 `cmd.Execute()` |
| `go.mod` | `/` | 模块名、Go 版本、直接/间接依赖 |
| `go.sum` | `/` | 依赖校验和 |
| `sqlc.yaml` | `/` | sqlc 生成配置：引擎 SQLite，输入 `internal/db/sql`，输出 `internal/db` |
| `opencode-schema.json` | `/` | 用户配置的 JSON Schema（由 cmd/schema 生成） |
| `.opencode.json` | `/` | 示例：为 gopls 配置 LSP |
| `.goreleaser.yml` | `/` | 发布流水线：多 OS/ARCH 构建与打包 |
| `.gitignore` | `/` | Git 忽略规则 |
| `install` | `/` | curl \| bash 安装脚本，支持 `VERSION=` |
| `LICENSE` | `/` | MIT 许可证 |
| `README.md` | `/` | 产品说明、安装、配置、快捷键（部分与代码有出入） |
| `1DESIGN.md` | `/` | 设计说明书（本文档集之一） |
| `2HELP.md` | `/` | 使用说明书 |
| `3SOLUTION.md` | `/` | 后续优化方案 |
| `4SOURCE.md` | `/` | 本文件 |

### 2.2 `cmd/`

| 文件名 | 所在目录 | 主要功能简介 |
|--------|----------|--------------|
| `root.go` | `cmd/` | Cobra 根命令、flags、DB 连接、TUI/非交互分支、pubsub→tea 桥、MCP 预热 |
| `main.go` | `cmd/schema/` | 扫描 `SupportedModels` 打印配置 JSON Schema |
| `README.md` | `cmd/schema/` | schema 生成器用法 |

### 2.3 `internal/app/`

| 文件名 | 所在目录 | 主要功能简介 |
|--------|----------|--------------|
| `app.go` | `internal/app/` | `App` 组合根、主题初始化、非交互运行、优雅关闭 |
| `lsp.go` | `internal/app/` | 按配置启动/重启 LSP 客户端与 watcher |

### 2.4 `internal/config/`

| 文件名 | 所在目录 | 主要功能简介 |
|--------|----------|--------------|
| `config.go` | `internal/config/` | Viper 加载合并、提供商默认、校验、更新模型/主题、GitHub token |
| `init.go` | `internal/config/` | 是否显示项目初始化对话框、标记已初始化 |

### 2.5 `internal/db/`

| 文件名 | 所在目录 | 主要功能简介 |
|--------|----------|--------------|
| `connect.go` | `internal/db/` | 打开 SQLite、PRAGMA、执行迁移 |
| `db.go` | `internal/db/` | sqlc 生成的 `Queries` 基础（DBTX） |
| `models.go` | `internal/db/` | sqlc 生成：`Session`/`Message`/`File` |
| `querier.go` | `internal/db/` | sqlc 生成的 `Querier` 接口 |
| `sessions.sql.go` | `internal/db/` | 会话 CRUD Go 封装 |
| `messages.sql.go` | `internal/db/` | 消息 CRUD Go 封装 |
| `files.sql.go` | `internal/db/` | 文件快照 CRUD Go 封装 |
| `embed.go` | `internal/db/` | `//go:embed migrations` |
| `20250424200609_initial.sql` | `internal/db/migrations/` | 初始表、索引、触发器 |
| `20250515105448_add_summary_message_id.sql` | `internal/db/migrations/` | 会话增加 `summary_message_id` |
| `sessions.sql` | `internal/db/sql/` | sqlc 源：会话查询 |
| `messages.sql` | `internal/db/sql/` | sqlc 源：消息查询 |
| `files.sql` | `internal/db/sql/` | sqlc 源：文件查询 |

### 2.6 `internal/session/`、`message/`、`history/`、`permission/`、`pubsub/`

| 文件名 | 所在目录 | 主要功能简介 |
|--------|----------|--------------|
| `session.go` | `internal/session/` | 会话领域服务：根/任务/标题会话、pubsub |
| `message.go` | `internal/message/` | 消息服务：JSON parts 序列化 |
| `content.go` | `internal/message/` | 多态 ContentPart 与 Message 访问器 |
| `attachment.go` | `internal/message/` | 用户附件结构 |
| `file.go` | `internal/history/` | 文件版本快照服务 |
| `permission.go` | `internal/permission/` | 同步授权、会话级持久许可、自动批准 |
| `broker.go` | `internal/pubsub/` | 泛型事件总线 |
| `events.go` | `internal/pubsub/` | Event 类型与 created/updated/deleted |

### 2.7 `internal/llm/agent/`

| 文件名 | 所在目录 | 主要功能简介 |
|--------|----------|--------------|
| `agent.go` | `internal/llm/agent/` | Agent 主循环、流式处理、标题、压缩、计费、模型热切换 |
| `tools.go` | `internal/llm/agent/` | Coder/Task 工具集装配 |
| `agent-tool.go` | `internal/llm/agent/` | 嵌套 task agent 工具 |
| `mcp-tools.go` | `internal/llm/agent/` | MCP 发现、包装、调用 |

### 2.8 `internal/llm/provider/`

| 文件名 | 所在目录 | 主要功能简介 |
|--------|----------|--------------|
| `provider.go` | `internal/llm/provider/` | Provider 接口、工厂、消息清洗、选项函数 |
| `anthropic.go` | `internal/llm/provider/` | Anthropic Messages API、cache、thinking |
| `openai.go` | `internal/llm/provider/` | OpenAI Chat Completions、推理力度、Groq 等复用 |
| `gemini.go` | `internal/llm/provider/` | Gemini/Schema 转换与流式 |
| `copilot.go` | `internal/llm/provider/` | GitHub Copilot 端点与 token 交换 |
| `azure.go` | `internal/llm/provider/` | Azure OpenAI 客户端封装 |
| `bedrock.go` | `internal/llm/provider/` | AWS Bedrock（Claude） |
| `vertexai.go` | `internal/llm/provider/` | Vertex AI Gemini 封装 |

### 2.9 `internal/llm/models/`

| 文件名 | 所在目录 | 主要功能简介 |
|--------|----------|--------------|
| `models.go` | `internal/llm/models/` | `Model` 结构、流行度、合并 `SupportedModels` |
| `anthropic.go` | `internal/llm/models/` | Claude 系列目录与单价 |
| `openai.go` | `internal/llm/models/` | GPT/o 系列 |
| `gemini.go` | `internal/llm/models/` | Gemini 系列 |
| `groq.go` | `internal/llm/models/` | Groq Llama/Qwen/Deepseek |
| `azure.go` | `internal/llm/models/` | Azure 部署名映射 |
| `openrouter.go` | `internal/llm/models/` | OpenRouter 模型 |
| `xai.go` | `internal/llm/models/` | xAI Grok |
| `vertexai.go` | `internal/llm/models/` | Vertex Gemini |
| `copilot.go` | `internal/llm/models/` | Copilot 聊天模型 |
| `local.go` | `internal/llm/models/` | 本地 OpenAI 兼容端点发现 |

### 2.10 `internal/llm/prompt/`

| 文件名 | 所在目录 | 主要功能简介 |
|--------|----------|--------------|
| `prompt.go` | `internal/llm/prompt/` | 按 Agent 分发系统提示、读取 contextPaths |
| `coder.go` | `internal/llm/prompt/` | Coder 系统提示（Anthropic/OpenAI 两套）+ 环境信息 |
| `task.go` | `internal/llm/prompt/` | Task 子 agent 提示 |
| `title.go` | `internal/llm/prompt/` | 会话标题提示 |
| `summarizer.go` | `internal/llm/prompt/` | 压缩摘要提示 |
| `prompt_test.go` | `internal/llm/prompt/` | contextPaths 合并测试 |

### 2.11 `internal/llm/tools/`

| 文件名 | 所在目录 | 主要功能简介 |
|--------|----------|--------------|
| `tools.go` | `internal/llm/tools/` | `BaseTool`、`ToolInfo`、`ToolResponse`、上下文键 |
| `bash.go` | `internal/llm/tools/` | Shell 执行、黑名单、超时与输出截断 |
| `shell.go` | `internal/llm/tools/shell/` | 持久 bash 进程与命令队列 |
| `grep.go` | `internal/llm/tools/` | ripgrep 优先的内容搜索 |
| `glob.go` | `internal/llm/tools/` | doublestar 文件名搜索 |
| `ls.go` | `internal/llm/tools/` | 目录树、隐藏/pycache 过滤 |
| `ls_test.go` | `internal/llm/tools/` | LS 工具与树算法测试 |
| `view.go` | `internal/llm/tools/` | 分页读文件、行号、图片探测 |
| `edit.go` | `internal/llm/tools/` | 唯一字符串替换/创建/删除 |
| `write.go` | `internal/llm/tools/` | 整文件写入 |
| `patch.go` | `internal/llm/tools/` | 调用 diff.Patch 应用补丁 |
| `fetch.go` | `internal/llm/tools/` | HTTP 抓取与 HTML→Markdown |
| `sourcegraph.go` | `internal/llm/tools/` | 公开代码搜索 GraphQL |
| `diagnostics.go` | `internal/llm/tools/` | 聚合 LSP 诊断 |
| `file.go` | `internal/llm/tools/` | 读文件辅助记录 |

### 2.12 `internal/lsp/`

| 文件名 | 所在目录 | 主要功能简介 |
|--------|----------|--------------|
| `client.go` | `internal/lsp/` | LSP 进程生命周期、初始化、诊断存储 |
| `protocol.go` | `internal/lsp/` | 客户端协议辅助 |
| `methods.go` | `internal/lsp/` | 向 server 发送的 LSP 方法封装 |
| `handlers.go` | `internal/lsp/` | 处理 server 通知（如 publishDiagnostics） |
| `transport.go` | `internal/lsp/` | JSON-RPC 读写帧 |
| `language.go` | `internal/lsp/` | 语言/文件类型映射 |
| `watcher.go` | `internal/lsp/watcher/` | fsnotify 工作区监视并通知 LSP |
| `edit.go` | `internal/lsp/util/` | 应用 TextEdit |
| `tsprotocol.go` | `internal/lsp/protocol/` | 生成：LSP 全部结构体 |
| `tsjson.go` | `internal/lsp/protocol/` | 生成：LSP JSON 编解码 |
| `uri.go` | `internal/lsp/protocol/` | DocumentUri 处理 |
| `interface.go` | `internal/lsp/protocol/` | 符号/编辑结果接口 |
| `pattern_interfaces.go` | `internal/lsp/protocol/` | glob/相对模式接口 |
| `tsdocument-changes.go` | `internal/lsp/protocol/` | 文档变更联合类型 |
| `tables.go` | `internal/lsp/protocol/` | 协议常量表 |
| `LICENSE` | `internal/lsp/protocol/` | 生成代码许可证 |

### 2.13 `internal/tui/` 根与页面、布局、样式、主题

| 文件名 | 所在目录 | 主要功能简介 |
|--------|----------|--------------|
| `tui.go` | `internal/tui/` | 根 Model、全局快捷键、对话框调度、自动 compact |
| `chat.go` | `internal/tui/page/` | 聊天页：发送消息、布局三栏 |
| `logs.go` | `internal/tui/page/` | 日志页 |
| `page.go` | `internal/tui/page/` | PageID 常量 |
| `layout.go` | `internal/tui/layout/` | 布局接口 |
| `split.go` | `internal/tui/layout/` | 分栏 |
| `container.go` | `internal/tui/layout/` | 容器 |
| `overlay.go` | `internal/tui/layout/` | 对话框遮罩 |
| `styles.go` | `internal/tui/styles/` | 通用 Lipgloss |
| `markdown.go` | `internal/tui/styles/` | Glamour Markdown |
| `background.go` | `internal/tui/styles/` | 背景 |
| `icons.go` | `internal/tui/styles/` | 图标字符 |
| `theme.go` | `internal/tui/theme/` | Theme 接口与 BaseTheme |
| `manager.go` | `internal/tui/theme/` | 注册/切换/当前主题 |
| `theme_test.go` | `internal/tui/theme/` | 主题注册与切换测试 |
| `opencode.go` 等 9 个主题文件 | `internal/tui/theme/` | 各配色主题（opencode/catppuccin/gruvbox/monokai/dracula/flexoki/onedark/tokyonight/tron） |
| `images.go` | `internal/tui/image/` | TUI 内嵌图片资源处理 |
| `util.go` | `internal/tui/util/` | tea.Cmd 辅助、Clamp |

### 2.14 `internal/tui/components/`

| 文件名 | 所在目录 | 主要功能简介 |
|--------|----------|--------------|
| `chat.go` | `.../chat/` | 聊天区组合 |
| `editor.go` | `.../chat/` | 多行编辑器、发送、外部编辑器、附件 |
| `list.go` | `.../chat/` | 消息列表、流式渲染、工作中指示 |
| `message.go` | `.../chat/` | 单条消息与工具调用 UI |
| `sidebar.go` | `.../chat/` | 会话信息与文件 diff 统计 |
| `status.go` | `.../core/` | 底栏 token/cost/LSP |
| `session.go` | `.../dialog/` | 会话选择 |
| `models.go` | `.../dialog/` | 模型选择 |
| `theme.go` | `.../dialog/` | 主题选择 |
| `commands.go` | `.../dialog/` | 命令面板 |
| `custom_commands.go` | `.../dialog/` | 扫描用户/项目 Markdown 命令 |
| `custom_commands_test.go` | `.../dialog/` | `$NAME` 正则测试 |
| `arguments.go` | `.../dialog/` | 命令命名参数表单 |
| `permission.go` | `.../dialog/` | 权限允许/拒绝 |
| `filepicker.go` | `.../dialog/` | 附件文件选择 |
| `complete.go` | `.../dialog/` | `@` 补全对话框 |
| `help.go` | `.../dialog/` | 快捷键帮助 |
| `init.go` | `.../dialog/` | 项目初始化 OpenCode.md |
| `quit.go` | `.../dialog/` | 退出确认 |
| `table.go` | `.../logs/` | 日志表 |
| `details.go` | `.../logs/` | 日志详情 |
| `simple-list.go` | `.../util/` | 通用简单列表 |

### 2.15 基础设施其它包

| 文件名 | 所在目录 | 主要功能简介 |
|--------|----------|--------------|
| `diff.go` | `internal/diff/` | Unified diff 解析、行内高亮、并排渲染 |
| `patch.go` | `internal/diff/` | 补丁解析、模糊上下文匹配、应用 |
| `fileutil.go` | `internal/fileutil/` | rg/fzf 命令、隐藏跳过、doublestar glob |
| `format.go` | `internal/format/` | 非交互 text/json 输出 |
| `spinner.go` | `internal/format/` | `-p` 模式等待动画 |
| `logger.go` | `internal/logging/` | slog 封装、会话调试文件、RecoverPanic |
| `writer.go` | `internal/logging/` | 将日志写入 TUI 可订阅的 writer |
| `message.go` | `internal/logging/` | 日志消息结构 |
| `files-folders.go` | `internal/completions/` | `@` 文件/目录补全数据源 |
| `version.go` | `internal/version/` | 版本字符串 |

### 2.16 `scripts/` 与 `.github/`

| 文件名 | 所在目录 | 主要功能简介 |
|--------|----------|--------------|
| `check_hidden_chars.sh` | `scripts/` | CI：检测隐藏 Unicode |
| `release` | `scripts/` | 按 major/minor/patch 打 tag |
| `snapshot` | `scripts/` | Goreleaser snapshot 辅助 |
| `build.yml` | `.github/workflows/` | 推送时 snapshot 构建 |
| `release.yml` | `.github/workflows/` | Release 发布 |

---

## 3. 按层归类的文件统计

| 层 | 大约文件数 | 说明 |
|----|------------|------|
| 入口与 CLI | 3 | main + cmd |
| 配置/DB/领域服务 | ~20 | config/db/session/message/history/permission/pubsub |
| LLM Agent/Provider/Models/Prompt/Tools | ~45 | 核心智能 |
| LSP | ~16 | 含超大生成文件 2 个 |
| TUI | ~50 | 页面、组件、主题 |
| 基础设施 | ~10 | diff/logging/format/fileutil/completions/version |
| 工程配置 | ~10 | go.mod、CI、install、schema |

---

## 4. 阅读源代码的推荐顺序

1. `main.go` → `cmd/root.go` → `internal/app/app.go`  
2. `internal/config/config.go`、`internal/db/migrations/*`  
3. `internal/llm/agent/agent.go`（主循环）  
4. `internal/llm/tools/*.go`、`internal/permission/permission.go`  
5. `internal/llm/provider/provider.go` + 任一厂商文件  
6. `internal/tui/tui.go` → `page/chat.go` → `components/chat/*`  
7. `internal/lsp/client.go`（需要时再读 protocol 生成代码）

---

## 5. 生成代码与手写代码边界

| 生成 | 手写 |
|------|------|
| `internal/db/*.sql.go`、`models.go`、`querier.go`、`db.go` 部分 | `internal/db/sql/*.sql`、`connect.go`、`embed.go` |
| `internal/lsp/protocol/tsprotocol.go`、`tsjson.go` | `client.go`、`handlers.go`、`transport.go`、`watcher` |
| `opencode-schema.json` | `cmd/schema/main.go` |

修改数据库协议应先改 SQL 再跑 sqlc；不要手改 `*.sql.go`。

---

## 6. 测试与可执行入口对照

| 入口 | 路径 |
|------|------|
| 主程序 | `main.go` |
| Schema 小工具 | `cmd/schema/main.go` |
| 测试 | `internal/llm/tools/ls_test.go`、`internal/llm/prompt/prompt_test.go`、`internal/tui/theme/theme_test.go`、`internal/tui/components/dialog/custom_commands_test.go` |

运行：`go test ./...`

---

*统计基于当前工作区文件系统，行数随后续提交变化。*
