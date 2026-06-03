# 02 · Hermes Agent — 运行期两层进化 + skill 工具 + provenance

源码：`self-evolution/hermes-agent/`。**已有代码级深挖笔记**（直接复用，别重挖）：
- `notes/hermes-agent-features/01-autonomous-skill-creation.md`（review prompt 全文结构、4 级偏好顺序、Do-NOT-capture 反模式、class-level 命名）
- `notes/hermes-agent-features/02-skill-self-improvement.md`（per-turn 触发计数、fork 配置、skill_manage 6 动作、Curator 全套、provenance）

落地：**M2 (A 层 per-turn)** + **M4 (C 层 Curator)** + SkillStore 工具语义。

## ① 它是什么

skill 在使用过程中被改。两条独立路径：**Per-turn Review Fork**（快、局部、单 session）+ **Curator**（慢、全局、跨 session 维护）。代码层**没有** diff engine / 执行轨迹比对器——改不改、怎么改全由一个 fork 出来的 review agent 读 prompt + 对话历史自主决定，约束来自 prompt 而非代码。

## ② 我们搬什么 / 不搬什么

| 机制 | 搬不搬 | 在 CC 外怎么落 |
|---|---|---|
| Per-turn 触发语义（iter 计数→turn 末 review） | ✅ 搬语义 | 改不了 CC loop → 用 PostToolUse 计数 + Stop 触发重建（见 §1） |
| fork review agent + 工具白名单 + 危险命令 auto-deny + 继承 prompt cache | ✅ 搬 | 用 headless `claude -p`/Agent SDK 子进程等价替代（[04](./04-claude-mem.md) 给运行载体） |
| skill_manage：patch(fuzzy)/edit/write_file/remove_file/delete + 写坏安全扫描即回滚 | ✅ 搬 | SkillStore.apply/create/archive 的语义来源 |
| Curator：idle 触发、archive-not-delete、stale/archive 生命周期、pinned 跳过、只动 agent-created | ✅ 搬 | M4 整体 |
| provenance：区分 user/agent-created | ✅ 搬 | 扩成 seed/agent/user 三态（[design 决策8]） |
| review prompt 内核（4 级偏好、Do-NOT-capture、class-level 命名） | ✅ 搬 | A 层 system prompt 基底 |
| Hermes 自由 review 当"挖共性"的脑 | ❌ 不用 | 挖共性交给 SkillOpt minibatch（[01](./01-skillopt.md)）；A 层才用自由 review |

## ③ 具体怎么运作（关键旋钮，详见 notes/）

### 3.1 Per-turn 触发（A 层基底）
- 计数器 `_iters_since_skill`，阈值 `_skill_nudge_interval` 默认 **10**。
- 每个 tool iteration +1；一旦调了 `skill_manage` 归零；turn 末 ≥阈值则 fork review。
- **我们的重建**：worker 按 `session_id` 维护计数；每个 `PostToolUse` +1；`Stop` 时 ≥10 触发 A 层；A 层动了 skill 后归零。（design 决策2）

### 3.2 fork review agent 配置（A 层子 agent 参照）
`max_iterations=16`、`quiet_mode=True`、memory+skill nudge 设 0 防递归、`skip_memory=True`、工具白名单仅 memory+skills、继承 `_cached_system_prompt`+`session_id` 命中 prefix cache（实测 ~26% 成本↓）、`_bg_review_auto_deny` 危险命令自动 deny。→ 我们的 headless 子 agent 照此配：限轮数、quiet、工具白名单=SkillStore+MCP只读、禁危险 Bash、继承 cache。

### 3.3 skill_manage 6 动作（SkillStore 语义来源）
`create` / `edit`(整写) / `patch`(fuzzy 局部替换，`fuzzy_find_and_replace`，不匹配回 preview 让模型自纠) / `write_file`(references/templates/scripts 白名单) / `remove_file` / `delete`(带 `absorbed_into` 声明合并)。**安全扫描**：patch/edit 后扫描，被 block 则 `_atomic_write_text` 原子回滚到原内容。→ SkillStore.apply 直接采用：先快照、fuzzy 改、安全扫描、fail 回滚。

### 3.4 Curator（M4 全套）<a id="4"></a>
- 触发：`maybe_run_curator()` —— idle ≥ `DEFAULT_MIN_IDLE_HOURS`(默认 **2h**) 且 距上次 ≥ `DEFAULT_INTERVAL_HOURS`(默认 **7天**)。
- 生命周期：`STALE_AFTER_DAYS`=**30**、`ARCHIVE_AFTER_DAYS`=**90**。
- 严格不变量：**只动 agent-created**（我们扩展：seed/user 都不动）、**从不 hard-delete 只 archive（可恢复）**、**pinned 跳过所有 auto-transition**、走 auxiliary client 不污染主 cache。
- 动作：pin / archive(stale) / consolidate(合并近重复) / patch。
- 状态文件 `.curator_state`：last_run_at / run_count / paused 等。→ M4 照搬这套阈值与不变量；并接管 [01 §5](./01-skillopt.md#5) 的 SLOW_UPDATE 保护区维护。

## ④ 关键参数默认值

| 参数 | 默认 | 取值 |
|---|---|---|
| skill nudge interval（A 触发） | 10 | 沿用 10 |
| review fork max_iterations | 16 | 沿用 |
| Curator idle 门 | 2h | 沿用 |
| Curator 间隔 | 7天 | 沿用 |
| stale / archive | 30天 / 90天 | 沿用 |
| 回滚机制 | 安全扫描 fail 即原子回滚 | 扩为"每改必快照 + 可显式 rollback" |
