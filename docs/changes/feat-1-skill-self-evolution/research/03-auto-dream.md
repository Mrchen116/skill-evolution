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

→ B 层子 agent 输入照此组织：**全部当前 skill 列表**（对应 A/B 层的"input 所有 skill"）+ **本批轨迹切片**（窄读）+ **任务指令**（minibatch analyst 指令）。（注：早期还列了 rejected_edits 负反馈——**已按决策 E 删除**，在线不做效果反馈；另：我们在 CC 外，**拿不到主 session cache 便宜**，需自付 token，见 §⑦ 三类。）

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

## ⑦ 可参考 / 不可参考速查（出方案时直接查这张）<a id="7"></a>

> 同 [01-skillopt §8](./01-skillopt.md#8) / [07-trace2skill §⑥](./07-trace2skill.md#6) 的拆法。
>
> **先记住 auto-dream 的特殊定性**：它和 SkillOpt/Trace2Skill **性质不同**——那两个是"records→skill 的**引擎**"，auto-dream **不是引擎、是外壳**：它 fork **一个**子 agent，给一段 prompt + 只读工具，让 agent 自己在一个回合里多步完成。
> 1. ⭐ **它就是「Agent 派（方案 A）」的现成模板**（一个子 agent + 一段主 prompt + 工具）。
> 2. **它对 B 的价值是"外壳"不是"脑"**：触发门控 + 输入装配 + 只读安全。它自己那个"脑"（4 阶段 Orient/Gather/Consolidate/Prune）做的是**整理已有 MEMORY.md**，不挖轨迹找新缺口——那套脑其实对应我们的 **C（Curator）**，不是 B 的"找共性改 skill"。

### 一类 · 能学，且是「纪律/内容」（进 prompt：喂 B 的调查 agent + C/Curator）

| 名字 | 是什么 | 为什么值得学 | 怎么用 · 出处 |
|---|---|---|---|
| **整理纪律** | 几条改记忆/文档的卫生规则：相对日期（"昨天/上周"）转成绝对日期；今日调查推翻的旧事实从源头删；新信息合并进已有文件、不建近似重复；索引文件（MEMORY.md）只放一行一条的指针、不塞正文。 | 这正是 **C（Curator）维护 skill 库**要的纪律；"别建近重复"也用于 B/A 的新建判定，防库膨胀、防过期。 | C 的 prompt 条款 + B/A 新建判定｜consolidationPrompt Phase 3/4 |
| **窄读纪律** | 明确告诉 agent："JSONL 很大，**窄关键词 grep、别整文件读**；只查你已经怀疑重要的，别穷尽式读"。 | 任何要读真实轨迹（jsonl）的 agent/步骤都得这样，否则上下文爆炸、又慢又贵。 | B 的调查 agent / 打标步 prompt｜Phase 2 |
| **只读工具约束** | 调查阶段把 Bash 限死在只读命令（`ls/find/grep/cat/stat/wc/head/tail`），任何写入/重定向/改状态一律拒，并在 prompt 里明说"据此规划、别试探边界"。 | 给"调查/分析"阶段一道硬安全边界——分析时本就不该写盘，写盘只发生在受控的 apply 步。 | B 调查阶段工具白名单 + prompt 声明｜`## Additional context` |

### 二类 · 能学，但是「调度底盘 / 工程外壳」（与 W/A 引擎无关，两套都要）

| 名字 | 是什么 | 为什么值得学 | 落到哪 · 出处 |
|---|---|---|---|
| **三重门控（最便宜先做）** | 按"越便宜越先做"排：① 时间门（距上次整理 ≥minHours）→ ② 10min 扫描节流（时间到但 session 不够时，不前进 lock、避免每轮 stat 全目录）→ ③ 会话门（**只 stat 各 jsonl 的 mtime、不读正文**，新 session ≥minSessions）→ ④ PID 锁（死进程/1h 过期可 reclaim）。 | 一套现成的"后台何时才值得真跑"的判据，省开销、防并发、防空转。**B 调度器直接复刻。** | B 调度器｜`autoDream.ts` + `consolidationLock.ts` |
| **五层输入装配** | 给 fork 子 agent 的不是一条 prompt，是"完整 API 前缀 + 一条新 user message"：系统(格式/类型规范)+上下文(全部资产+currentDate)+git+主对话+新 user message(任务指令+session 列表+工具约束)。 | 它是"喂 B 的 agent/分析器**到底该带哪些料**"的现成清单（全部 skill + 相关轨迹 + 任务 + 约束）。 | B 的 agent/analyst 输入清单｜`forkedAgent.ts` 五层 |
| **fork 子 agent + 只读工具 的执行外壳** | 后台起一个子 agent、给只读工具、跑完返回一段摘要，全程不打扰主流程。 | **这正是方案 A（Agent 派）的执行模板**——B 走 A 路时照此搭外壳。 | 方案 A 的执行外壳｜`forkedAgent.ts` + stopHook fire-and-forget |

### 三类 · 不能直接学 / 必须改

| 名字 | 是什么 | 为什么不能直接学 | 我们怎么改 · 出处 |
|---|---|---|---|
| **它的"脑"是整理、不是挖掘** | 4 阶段（Orient→Gather→Consolidate→Prune）做的是 tidy 已有 MEMORY.md：去重、纠错、修剪、修索引。 | 这是**维护**，不是 B 要的"读轨迹找反复缺口、改 skill"。把它当 B 的分析脑会跑偏。 | 这套脑对应 **C（Curator）**；B 的脑换成 SkillOpt/Trace2Skill 的分析｜consolidationPrompt |
| **触发钥匙不同** | auto-dream **全局**触发：距上次 ≥24h 且新 session ≥5。 | 我们 B 是**按单个 skill 的使用计数**触发（某 skill 用过 >X 次）——计数维度不一样。 | 借门控**机制**，把触发**钥匙**换成 per-skill 使用计数｜Gate 1/2 |
| ⭐**继承主 session cache 的便宜** | auto-dream 在 CC **内部** fork，能白嫖主 session 已缓存的 prompt 前缀（systemPrompt/userContext/对话），近乎零额外成本带上全部上下文。 | 我们在 CC **外**（claude-mem 底盘，改不了 CC 内部），**拿不到这个内部特权**。 | 得**自己从 jsonl 重组上下文、按全价付 token**——诚实承认这笔成本｜`forkedAgent.ts:530` |
| **fire-and-forget 挂在 CC stopHook** | 每轮对话结束，CC 在自己的 stopHook 里顺手触发 auto-dream。 | CC 的内部钩子我们改不了。 | 借"形"（stop 时投递一个触发信号），**触发判定放我们自己的 worker**（claude-mem 式）｜`stopHooks.ts` |

### 一句话

auto-dream 给 B 的是**外壳三件套**（何时触发 / 喂什么 / 怎么安全跑），以及 **Agent 派的执行模板**；它的"脑"归 **C（Curator）**，不归 B。**别把它当 B 的分析引擎**——B 的引擎在 SkillOpt / Trace2Skill。
