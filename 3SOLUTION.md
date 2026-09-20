# OpenCode 后续优化方案

| 项目 | 内容 |
|------|------|
| 文档名称 | 基于 2026 年代 Agent 技术趋势的优化方案 |
| 对象系统 | OpenCode（约 2025 年架构的终端编码 Agent，本仓库已归档） |
| 编制日期 | 2026-09-20 |
| 对标产品 | Claude Code、Cursor Agent/Cloud Agents、Codex CLI、Charm Crush、OpenAI Agents SDK、MCP 生态、A2A |

---

## 1. 背景：为什么“去年的 Agent”在 2026 年显得过时

OpenCode 的骨架仍然正确：**本地工具 + 多提供商 LLM + 会话持久化 + 人机权限**。这是 2024–2025 年编码 Agent 的标准形态（ReAct 循环、function calling、MCP 初版、TUI）。

2025 下半年至 2026 年，行业重心从“能调用工具的聊天机器人”转向：

1. **长时程、可恢复的 Agent**：任务以小时/天计，可暂停、可分叉、可在云端/本机切换；
2. **上下文工程（Context Engineering）** 取代单纯 Prompt 堆砌：检索、摘要、记忆分层、工具结果预算成为一等公民；
3. **多 Agent 编排与协议**：子 Agent 不再是“同步黑盒再返回一段文本”，而是有 mailbox、A2A、共享工件；
4. **MCP 深化 + 技能包（Skills）**：工具发现、鉴权、采样、资源、根目录、OAuth；可安装的 Agent Skill（流程+脚本+策略）；
5. **并行工具、流式工具、推测执行**：一轮发出数十个只读调用；写操作沙箱化；
6. **评测与可观测性**：Agent eval、轨迹回归、OpenTelemetry for Agents、成本归因；
7. **规格驱动开发（Spec-driven）**：先写规格/计划/测试，再改代码，减少“乱改仓库”；
8. **身份、权限与供应链**：最小权限、审计、不可信 MCP、提示注入隔离；
9. **混合运行时**：本机 TUI + 云端后台 Agent + CI 中的无头 Agent。

OpenCode 源码中已经出现若干“早期正确方向”（Task 子 Agent、MCP、autoCompact、权限对话框、LSP diagnostics），但实现深度停在 2025 年原型级。下列方案按优先级给出可落地的演进路径。即使本仓库归档，对 Crush 或任何 fork 同样适用。

---

## 2. 目标架构（演进后）

```
                    ┌─────────────┐
                    │ Spec/Plan   │  规格、验收标准、待办
                    └──────┬──────┘
                           ▼
┌──────────┐  事件/轨迹   ┌─────────────────────┐
│ TUI/CLI  │◄────────────►│ Orchestrator        │
│ IDE/CI   │              │ (图/状态机, 可恢复) │
└──────────┘              └──────────┬──────────┘
      ▲                              │
      │ HITL                         ▼
┌─────┴──────┐            ┌─────────────────────┐
│ Permission │            │ Worker Agents       │
│ + Policy   │            │ coder / explorer /  │
│ Engine     │            │ reviewer / tester   │
└────────────┘            └──────────┬──────────┘
                                     │
              ┌──────────────────────┼──────────────┐
              ▼                      ▼              ▼
        Tool Runtime            Memory Fabric    Model Router
        (sandbox, parallel,     (episodic /     (capability,
         MCP, computer-use)      semantic /      cost, latency)
                                 procedural)
              │                      │
              ▼                      ▼
        Git worktree /          Session store +
        snapshot FS             vector/log/otel
```

关键变化：Agent 循环从 `for { stream; run tools sequentially }` 升级为 **带检查点的编排器**；工具从“进程内 Go 函数”升级为 **带策略的运行时**；上下文从“全量消息 + 一次摘要”升级为 **分层记忆**。

---

## 3. 方案一：编排器与可恢复长任务（P0）

### 现状

`agent.processGeneration` 是单 goroutine 死循环，状态只在 SQLite 消息表和内存 `activeRequests`。进程退出即丢失“正在跑”的语义；没有步骤图、没有重试策略、没有人工插入点（除权限）。

### 2026 趋势对齐

Cloud Agent / background agent 把一次任务建模为 **Run**：有状态（queued/running/waiting_human/succeeded/failed）、有 checkpoint、可在另一台机器续跑。

### 建议

1. 引入 `Run` / `Step` 表：每次用户提交创建一个 Run；每轮 LLM、每个工具调用是 Step，写入 input/output hash、token、错误。
2. `waiting_human` 状态覆盖权限、澄清问题、计划确认（Plan mode）。
3. 崩溃恢复：启动时扫描未完成 Run，从最后成功 Step 续执行。
4. 取消粒度：取消当前工具、取消 Run、取消子 Agent 树。
5. 与 Git 集成：每个 Run 绑定 worktree 或 stash 点，失败可一键回滚。

### 验收

杀死进程后重启，未完成的“重构 X 并跑测试”任务从下一个工具调用继续，而不是重头生成。

---

## 4. 方案二：并行工具与工具结果预算（P0）

### 现状

工具 **严格顺序** 执行。探索类任务（多 glob + 多 view）延迟线性叠加。工具输出无统一预算，bash 仅截断 30k 字符，大文件 view 可迅速撑爆上下文。

### 建议

1. 在同一 assistant 消息内，对 **只读工具**（glob/grep/ls/view/sourcegraph/diagnostics）并行执行，写工具仍串行且排在只读之后。
2. 引入 `ToolResultBudget`：按模型 `ContextWindow` 的百分比（如 15%）分配；超限时存完整结果到磁盘/对象存储，只把摘要+指针回给模型，并提供 `tool_result_read` 分页工具。
3. 支持 **流式工具输出**（MCP 已有 progress/log）：TUI 显示进度，不必等 bash 结束。
4. 允许模型在工具执行中 **中断并改计划**（speculative tool calls + cancel）。

### 算法要点

依赖分析：根据工具副作用标注 `readonly` / `writes(path)` / `exec`。无冲突的 readonly 用 `errgroup` 并行；写同一 path 的调用合并或串行化。

---

## 5. 方案三：上下文工程与分层记忆（P0）

### 现状

- 系统提示一次性注入若干 markdown 路径（`sync.Once`，进程内永不刷新）；
- 压缩是“整段对话丢给 summarizer，同一 session 打 SummaryMessageID”；
- session.PromptTokens 被 **每轮覆盖**，自动压缩阈值统计并不真正等于上下文占用；
- 无向量检索、无代码地图、无跨会话记忆。

### 建议：四层记忆

| 层 | 内容 | 更新策略 |
|----|------|----------|
| Procedural | OpenCode.md / Skills / 用户规则 | 文件监视热加载 |
| Semantic | 仓库代码块、文档、issue 切片 | 启动与 git hook 增量索引 |
| Episodic | 本 Run 的决策、失败尝试、用户偏好 | 每步写入；压缩时保留结构化条目而非散文摘要 |
| Working | 当前模型窗口 | 检索 + 引证 + 摘要，禁止无界 append |

具体技术：

1. **真实 token 计量**：用与提供商一致的 tokenizer 或 API 返回的 `usage` **累计上下文快照**，而不是覆盖 PromptTokens。
2. **结构化压缩**：摘要 JSON schema（目标、已改文件、测试状态、禁区、下一步），不要只存一段散文。
3. **代码检索工具**：基于 tree-sitter 分块 + embedding（或 BM25 先上）的 `code_search`，替代“盲目 grep + 塞进上下文”。
4. **引用（citations）**：助手回答附 file:line，TUI 可跳转。
5. **ContextPaths 热更新**：去掉错误的进程级 `Once`，改 fsnotify。

---

## 6. 方案四：Plan Mode、规格与测试闭环（P0）

### 现状

Coder 系统提示要求“一直做到完”，没有强制的计划确认，也没有把测试作为任务完成门禁。容易出现未运行测试的“完成”。

### 建议

1. **Plan mode**（默认对“跨文件重构/新功能”开启）：先产出计划（影响面、风险、测试命令），用户批准后再进入执行态。
2. **Definition of Done**：Run 结束前必须执行项目检测命令（`go test`、`npm test` 等，来自 OpenCode.md 或自动推断），失败则自动再循环，上限 N 次。
3. **规格文件一等公民**：`.opencode/specs/*.md` 作为任务输入，Agent 对照验收清单打勾。
4. **只读探索 / 写入执行** 分阶段切换工具集，减少计划阶段误改文件。

这与 2026 年“spec-driven agents”和“test-gated agents”一致，也是减少权限疲劳的办法：计划阶段几乎不弹权限。

---

## 7. 方案五：多 Agent 体系从“假分身”升级（P1）

### 现状

`agent` 工具同步等待子 Agent，子 Agent **不能写文件**，父模型只得到最终字符串，用户不可见中间过程。无法并行多个探索 Agent 真正加速（提示里写了 concurrent，实现是父模型一轮多个 tool_use，但父侧仍顺序 `Run`）。

### 建议

1. 子 Agent 并行：父消息中多个 `agent` 调用真正 `errgroup`。
2. 角色分化：
   - **Explorer**：只读，广搜；
   - **Implementer**：写入 + bash；
   - **Reviewer**：diff + diagnostics + 测试日志，只评论不改（或只提 patch）；
   - **Researcher**：fetch/MCP/文档。
3. 共享 **Artifact Store**（补丁、测试输出、计划），用 ID 引用，避免把大文本在 Agent 间复制。
4. 对齐 **A2A（Agent-to-Agent）** 或至少内部 mailbox：子 Agent 可提问、可上报进度事件到 TUI。
5. 用户可见“子任务树”，可单独取消/重跑。

---

## 8. 方案六：MCP 2.0 级集成与 Skills（P1）

### 现状

- 仅 stdio/SSE，进程级 ListTools 缓存，不刷新；
- 每次调用重新 Initialize+Close，延迟高；
- 只映射 Tools，忽略 Resources、Prompts、Sampling、Roots、OAuth、elicitation；
- 工具名 `{server}_{tool}` 易碰撞且不稳。

### 建议

1. **长连接会话**：按 server 维持 client，心跳与重连；配置变更热加载。
2. 实现 MCP Resources（把仓库文件/数据库以 resource 暴露）、Prompts（可调用的提示模板）。
3. **Roots**：把工作目录以 MCP root 形式声明，减少路径越界。
4. **鉴权**：SSE/HTTP MCP 的 OAuth 与 secret 从系统钥匙串读取，不进 git。
5. **Sampling**：允许 MCP 服务器反向请求模型时走同一权限与预算。
6. **Skills 目录**：类似“可安装技能包”（YAML + 脚本 + 说明），用户 `opencode skill add ...`，比散落的 markdown 命令更可分发、可版本化。
7. 对不可信 MCP：**工具输出视为 untrusted**，与用户文本隔离，缓解提示注入。

---

## 9. 方案七：安全架构重构（P0/P1）

### 现状缺口

- 非交互模式整会话自动批准；
- 权限 channel **无超时**；
- bash 黑名单极易绕过（`python -c urllib`、`bash -c curl`）；
- 无文件系统沙箱、无 netns；
- 项目内 `.opencode.json` 可覆盖全局，存在恶意仓库投毒；
- MCP 与 fetch 可成为数据外泄通道。

### 建议

1. **策略引擎**（替代布尔 auto-approve）：
   - 规则示例：`allow glob,grep,ls,view`；`ask bash,edit,write`；`deny fetch` 在 CI 以外；
   - 路径 ACL：仅工作区 + 显式 extra mounts；
   - 环境变量脱敏。
2. **沙箱执行**：优先用 OS 机制（Linux bubblewrap/landlock、macOS sandbox-exec、或专用 VM/容器）。bash/MCP stdio 默认进沙箱；`--privileged` 才宿主机。
3. **批准分类**：读操作默认静默；工作区内编辑可“会话记住”；工作区外、网络、密钥文件永远确认。
4. **配置信任分级**：全局配置 > 用户配置 > 项目配置（项目不能静默改 API key、不能加任意 MCP）。
5. **提示注入防护**：外部 fetch/MCP/issue 正文放入明确 delimiters；禁止其内容直接扩展工具权限。
6. 审计日志：每次工具调用签名写入 append-only log，便于事故回溯。

---

## 10. 方案八：模型路由、成本与缓存（P1）

### 现状

四个 Agent 名绑定固定模型；无按任务路由；Anthropic cache 只打在尾部消息；本地模型发现较粗。

### 建议

1. **Router**：explorer/title 用小模型；实现用强模型；review 用另一家模型交叉检查。
2. **能力协商**：根据 `SupportsAttachments`、`CanReason`、上下文窗口自动降级或切模型，而不是丢附件。
3. **Prompt caching 标准化**：对稳定系统提示 + 工具 schema + 项目记忆打 cache breakpoint（Anthropic/OpenAI 均已支持类似语义）。
4. **费用预算**：用户设日/会话上限，超限转本地模型或停止。
5. **真实用量仪表**：累计 input/output/cache，修正当前“每轮覆盖 token 字段”的错误。
6. 跟进 2026 年常见 API：Responses API / 长上下文窗口（1M+）/ 延迟优化批次，但保持 Provider 接口稳定。

---

## 11. 方案九：可观测性、Eval 与回归（P1）

### 现状

debug 时把 JSON 轨迹写到 `.opencode/messages`；几乎无单测覆盖 Agent；ProviderMock 未完成。

### 建议

1. **OpenTelemetry**：Run/Step/LLM/Tool 自动 span；导出用户可选。
2. **轨迹格式**：采用开源轨迹 schema（或 OpenTelemetry gen-ai 语义约定），可回放。
3. **Eval 集**：
   - 单元：edit 唯一匹配、patch fuzz、权限四元组、token 累计；
   - 集成：用本地小模型或录制 HTTP 做 golden 轨迹；
   - 质量：HumanEval 式仓库任务、内部 fixture 仓库。
4. **失败聚类**：工具错误、权限拒绝、循环空转（同工具同参数重复 N 次熔断）。
5. CI 中跑 `go test ./...` 外加 `opencode eval --suite smoke`。

没有 eval，任何“换模型/改提示”都是蒙眼飞行——这是 2025 原型与 2026 产品的分水岭。

---

## 12. 方案十：运行时形态（TUI + Headless + 后台）（P1）

### 现状

只有本地前台 TUI 和一次性 `-p`。

### 建议

1. **稳定 Headless API**：JSON 行协议或本地 HTTP（仅 loopback + token），供 IDE/CI 嵌入。
2. **后台任务**：TUI 可 detach；`opencode ps / attach / logs`。
3. **IDE 协议**：LSP 已具备诊断，可反向做“轻量语言服务器”给编辑器展示 Agent diff。
4. **Computer-use 可选模块**：对 UI 测试、浏览器应用，走隔离浏览器（Playwright）而非给模型原始桌面控制，除非用户显式安装 skill。
5. 与 GitHub/GitLab：PR 上评论触发只读审查 Agent（密钥与仓库权限严格分离）。

---

## 13. 方案十一：TUI/UX 现代化（P2）

对齐 2026 年终端 Agent 体验：

1. 工具调用 **时间线** 而非仅聊天气泡；显示并行分组。
2. Diff 审阅：对每次 edit 可 `y/n/e`（接受/拒绝/手改），而不是事后 git。
3. 快捷键与 README 对齐；可配置 keymap（Vim/Emacs）。
4. 会话搜索、书签、导出 Markdown/JSONL。
5. 图像附件走剪贴板；终端图形协议（Kitty/iTerm）预览。
6. 无障碍：减少动画、屏幕阅读器友好。
7. 主题跟随终端 light/dark（已有 AdaptiveColor，需默认自动）。

---

## 14. 方案十二：工程与供应链（P2）

1. 模块化：`internal/llm/tools` 可做成 plugin hashicorp/go-plugin 或 WASM，降低主二进制攻击面。
2. 依赖治理：定期升级 LLM SDK；锁定 MCP 协议版本。
3. 生成代码与手写隔离（已部分做到）。
4. 发布 SBOM 与签名（cosign）。
5. 明确 fork 策略：若主仓归档，文档应指向 Crush 并说明行为差异。

---

## 15. 分阶段路线图

### 阶段 A — 正确性与安全基线（侵入中等，收益最高）

- 修复 token 累计与 autoCompact 触发；
- ContextPaths 热加载；
- 权限超时与策略文件；
- 项目配置信任分级；
- 只读工具并行；
- 补齐核心单测（agent 忙锁、edit、patch、permission）。

### 阶段 B — 产品级 Agent

- Run/Step 检查点；
- Plan mode + 测试门禁；
- 结构化压缩 + 简单 BM25 代码搜索；
- MCP 长连接与热更新；
- Headless JSONL；
- OTel 轨迹。

### 阶段 C — 平台级

- 沙箱；
- Skills 市场/本地目录；
- 多角色并行 + artifact store；
- Eval 套件与模型路由；
- 云端/本机 Run 迁移。

不建议一上来做“自主多 Agent 社交网络”。2026 年验证有效的路径是：**可恢复的单用户任务 + 强权限 + 强评测 + 上下文预算**。

---

## 16. 与当前代码的映射（改哪里）

| 方案 | 主要改动点 |
|------|------------|
| 编排/检查点 | 新包 `internal/run`；改 `agent.Run` 为驱动 Step；新表 goose 迁移 |
| 并行工具 | `streamAndHandleEvents` 工具循环；工具元数据加 `SideEffect` |
| 记忆 | `prompt.getContextFromPaths` 去 Once；新 `internal/memory`；改 `Summarize` |
| Plan/DoD | TUI 命令 + agent 状态机；`OpenCode.md` schema |
| 子 Agent | `agent-tool.go` errgroup；TUI 树 |
| MCP | `mcp-tools.go` 连接池；config 热更新 |
| 安全 | `permission` 策略语言；bash 进 sandbox 包 |
| 路由/成本 | `TrackUsage` 累计；`createAgentProvider` 路由表 |
| 可观测 | `logging` 旁路 OTel；eval 目录 |
| Headless | `cmd` 子命令 `agent run/attach` |
| UX | `internal/tui` 时间线与 diff 审阅 |

---

## 17. 明确不做什么（避免 2026 年常见陷阱）

1. **不要**为了“Agent 框架热词”引入过重的图编排 DSL，先把检查点表做对。
2. **不要**把完整桌面 Computer Use 设为默认，攻击面不可接受。
3. **不要**在无沙箱时扩大 bash 能力（去掉黑名单却不隔离）。
4. **不要**用单一散文摘要当作长期记忆。
5. **不要**让项目级 MCP 配置在用户未确认时启动。
6. **不要**用“再加一个系统提示段落”代替检索与预算——那是 2024 年的办法。

---

## 18. 成功指标

| 指标 | 基线（现状） | 阶段 B 目标 |
|------|--------------|-------------|
| 只读探索回合墙钟时间 | 顺序工具求和 | 下降 40%+ |
| 长任务进程被杀后可续跑 | 否 | 是 |
| 越权写工作区外文件 | 仅靠对话框 | 策略默认拒绝 |
| Agent 相关测试 | 4 个文件、无循环测试 | 循环/权限/压缩有单测 + smoke eval |
| 上下文溢出事故 | 靠 0.95 启发式且 token 统计偏差 | 真实 occupancy + 结构化压缩 |
| MCP 调用延迟 | 每次冷启动进程 | 热连接 p50 显著下降 |
| 用户权限疲劳 | 几乎每个 bash 都问 | 只读静默，写操作可会话记忆 |

---

## 19. 结论

OpenCode 作为 2025 年的终端编码 Agent，模块划分清晰，提供商抽象与权限意识在当时是加分项。站在 **2026 年**看，差距不在“再接入一个模型品牌”，而在：

**可恢复的任务模型、并行且有预算的工具运行时、分层记忆、可执行的计划/测试闭环、以及默认安全的策略与沙箱。**

按阶段 A→B→C 推进，可以把归档原型演进为与同年商业 Agent 可比较的工程系统；若选择迁移 Crush，上述清单可作为能力差距对照表使用。
