# 04 · claude-mem — 在 CC 外旁挂的集成底盘

源码：`self-evolution/claude-mem/`（thedotmack/claude-mem）。落地：**M1 chassis** 的整体范式（hook 接入 + worker + 增量 tail + SQLite + MCP）。**这是"怎么挂上 CC"的蓝本，不是算法。**

## ① 它是什么

纯靠 **hook + 读 transcript** 在 CC 外实现的持久记忆系统，完全不改 CC 内核。证明了"内部改不了、读 jsonl + 用 hook"这条路可行且稳定（多平台 adapter：claude-code/codex/openclaw/cursor）。

## ② 我们搬什么（M1 蓝图清单）

| 能力 | claude-mem 实现 | 我们 M1 落地 |
|---|---|---|
| **hook 当瘦分发器** | `plugin/hooks/hooks.json` 每个 hook 只 `node bun-runner worker-service hook <event>` | 插件 hooks.json：每 hook 投递事件给 worker 后秒回 |
| **常驻 worker + supervisor** | `worker-service.ts` + `supervisor/`（保活、信号处理、env 消毒） + `ProcessManager`(pid 文件/spawnDaemon/stale 清理/ownership 校验) + `HealthMonitor`(port/health/readiness) | worker daemon + supervisor + pid 锁 + 健康检查（对应 design Runbook 的 daemon CLI） |
| **增量 tail jsonl** | `transcripts/watcher.ts` `FileTailer`：`fsWatch`+`createReadStream`+offset；`state.ts` 持久化 `{offsets: Record<path, number>}` | TrajectoryTailer：按 path 记 offset 增量读，size 变小则重置 |
| **自带 SQLite** | `bun:sqlite`，`storage/sqlite/schema.ts`：projects/sessions/agent_events/memory_items/memory_sources/audit_log + 一堆索引 | 我们的 7 张表（design 接口§5），索引按 (skill, ts) / (session_id) |
| **MCP 客户端** | `@modelcontextprotocol/sdk` `Client`+`StdioClientTransport` 在 worker 内连 MCP | McpKnowledge 用同款 SDK 连【部门 MCP】只读检索 |
| **回灌通道** | MCP server + SessionStart 注入 context | v1 **不需要**（静默、改动直落 skill 文件即下个 turn 生效）；保留 MCP server 接口供未来 |
| **失败降级** | worker 连不上 → hook fallback 不阻断 CC | hook 投递失败即静默跳过，CC 照常（对应 design 风险段降级路径） |

## ③ 具体怎么运作（M1 要复刻的几个点）

### 3.1 hook 事件清单（`hooks.json` 实证）
claude-mem 挂的：`Setup`(版本检查) / `SessionStart`(startup\|clear\|compact → 启 worker + 注入) / `UserPromptSubmit`(session-init) / `PostToolUse(*)`(observation) / `PreToolUse(Read)`(file-context) / `Stop`(summarize)。
→ 我们挂：`SessionStart`(保活 worker) / `UserPromptSubmit`(turn 边界 + 抓 user 原话作纠偏信号) / `PostToolUse(*)`(计数 + tool 信号) / `Stop`(A 层触发点) / `SessionEnd`(flush)。**每个 hook 命令 = 读 stdin → 投递事件 → 输出 `{"continue":true,"suppressOutput":true}` 退出。**

### 3.2 worker 的"瘦 hook、重 worker"边界
hook 命令本质只是 `worker-service hook <platform> <event>`——把事件丢给常驻进程。**所有 LLM 级活（review/reflect/curator）在 worker 异步做**，hook 端口超时（claude-mem 配 60–120s）只覆盖"投递"，不覆盖"分析"。这是 design 决策1/2 的硬依据。

### 3.3 增量 tail（`watcher.ts` + `state.ts`）
`FileTailer` 用 `fsWatch` 监听文件、`createReadStream({start: offset})` 只读新增字节、按行解析、回调处理、更新 offset；offset 存一个 JSON（`{offsets: {<path>: <bytes>}}`）。我们 TrajectoryTailer 照此，offset 进 SQLite `session_state.last_offset`。

### 3.4 worker 子模块（claude-mem 的拆法，可借组织）
`worker/DatabaseManager` / `SessionManager` / `SSEBroadcaster` / `ClaudeProvider`(调 LLM) / supervisor。→ 我们对应：Storage / SessionState / （无需 SSE）/ ReviewAgent(headless claude) / Supervisor。

## ④ 关键工程取舍（直接抄）

- **单实例 pid 锁** + stale pid 清理 + ownership 校验：多 CC 窗口共用一个 worker。
- **env 消毒**：spawn daemon 时清理敏感 env（`env-sanitizer.ts`）。
- **健康检查三态**：port 占用 / health / readiness，供 Runbook 的 `daemon status`。
- **平台 adapter 分离**（`adapters/claude-code/mapper.ts`）：核心引擎与 hook 适配解耦——对应用户"CC/Codex/OpenClaw 生态"，v1 只做 claude-code adapter，但按此解耦留口。
