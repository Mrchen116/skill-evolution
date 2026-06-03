# 03 · Claude Code auto-dream — 后台反思的门控与输入装配

源码：`opensource-hub/claude-code/`。**已有完整笔记**（直接复用）：`opensource-hub/claude-code/笔记/auto-dream.md`（门控顺序、五层输入装配、四阶段 consolidation prompt 中英全文）。落地：**M3 (B 层)** 的调度门控 + 输入装配 + 整理纪律。

## ① 它是什么

CC 的后台**记忆整理**机制：不提取新记忆，而是攒够 session 后 fork 子 agent，对已有 `MEMORY.md` + topic 文件去重/纠错/修剪。我们把 `memory` 换成 `skill`、把"整理已有"扩成"重读轨迹挖共性问题改 skill"。

## ② 我们搬什么 / 不搬什么

| 机制 | 搬不搬 | 怎么落 |
|---|---|---|
| 多重门控（时间/量/锁/节流） | ✅ 搬 | B 层调度器的触发判据 |
| 门控阶段只 stat 不读 jsonl 正文 | ✅ 搬 | 省开销；触发后才窄读 |
| 五层输入装配（系统/上下文/全部资产/轨迹/任务指令） | ✅ 搬 | B 层子 agent 的输入组织 |
| 窄 grep 轨迹、不整文件读 | ✅ 搬 | 配合增量 tail（[04](./04-claude-mem.md)） |
| 整理纪律：相对日期转绝对、删被推翻的事实 | ✅ 搬 | MCP 知识注入防过期（design 决策7）+ Curator |
| 在 stopHook fire-and-forget 触发 | ⚠️ 借形不借栈 | 我们 hook 只投递；触发判定在 worker（hook 不能跑重活） |

## ③ 具体怎么运作（门控顺序，最便宜的先做）

```
Gate 0 全局开关（排除 KAIROS/Remote）
Gate 1 时间门：距上次整理 ≥ minHours（默认 24h）
Gate 1.5 扫描节流：时间门过但 session 不够时不前进 lock mtime，10min 最多扫一次
Gate 2 会话门：jsonl mtime > 上次整理时间 的 session 数 ≥ minSessions（默认 5，排除当前）
Gate 3 PID 锁：memory 目录下 .consolidate-lock（持有者 PID + mtime；死进程/1h 过期可 reclaim）
```
**门控阶段不读 jsonl 正文**，只 `readdir`+`stat` 各 `{uuid}.jsonl` 的 mtime。

→ **我们 B 层的调度器**直接复刻这套：
- 时间门 `minHours`（design 里写 ≥Xh，默认建议 **24h**，可调更短如 6h）。
- 会话量门 `minSessions`（默认 **5**，即"这段时间至少积累 5 个新 session 才值得跑批量"）。
- 单实例 PID 锁（worker 本就单实例，复用 supervisor 锁）。
- 节流避免空转。

## ④ 输入装配五层（B 层子 agent 输入参照）

auto-dream 给 fork 子 agent 的不是一条 prompt，是"完整 API 前缀 + 一条新 user message"：
A. systemPrompt（含格式/类型规范）｜B. userContext（全部资产 + currentDate）｜C. systemContext（git 等）｜D. 主 session 完整对话（共享 cache）｜E. 新 user message（四阶段任务指令 + session 列表）。

→ B 层子 agent 输入照此组织：**全部当前 skill 列表**（对应 A/B 层的"input 所有 skill"）+ **本批轨迹切片**（窄读）+ **任务指令**（minibatch analyst 指令）+ rejected_edits 负反馈。继承主 session cache 前缀省钱。

## ⑤ 整理纪律（搬给 MCP 注入 + Curator）

consolidation prompt 的几条纪律值得搬：
- **相对日期转绝对**（"昨天"→具体日期）——MCP 知识注入写 provenance 日期时照做。
- **删除被推翻的事实**——Curator 维护时，与现状矛盾的旧规则要修/删。
- **合并进已有而非建近重复**——对应 Codex reuse-before-create + Curator consolidate。

## ⑥ 关键参数默认值

| 参数 | auto-dream 默认 | B 层取值 |
|---|---|---|
| 时间门 minHours | 24h | 24h（可调 6–24h） |
| 会话门 minSessions | 5 | 5 |
| 扫描节流 | 10min | 10min |
| 锁过期 reclaim | 1h | 复用 supervisor 单实例锁 |
