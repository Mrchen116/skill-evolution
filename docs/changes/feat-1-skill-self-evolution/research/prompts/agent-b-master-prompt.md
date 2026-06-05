# Agent 模式 · B（单 skill 批量优化）主 prompt v1

> 这是 **Agent 方案**的那"一段主 prompt"（命门）。从各参考的 **prompt 原文**逐块取材，按 [08 关键决策](../08-关键决策记录.md) 拼装。
> - **编码了的决策**：A5 不做效果反馈 · A6/A7 ≥2 带证据(逼思考、不验证) · B1 用 `skill_manage` 改 · B2 读完整 session · B3 暂存目录+逐段蒸馏 · B4 success 不额外限制 · B5 不强制两段式(下面的"步骤"只是指导小节) · B6 不设改动预算、be active 但 ≥2 才动。
> - **故意不写的(决策 E)**：不检查"上次改动有没有效"、不做失败指纹比对、不做复发回滚、无 `rejected_edits`、无"≤K 改动"预算。
> - **代码侧、不进 prompt**：开工前 `curator_backup` 式快照；`skill_manage` 工具自带的 frontmatter/大小/路径校验与原子写、patch 不匹配回 preview；把相关已结束 session 落到暂存目录 + manifest。

---

## 一、Prompt 正文（英文给 agent，可直接用）

```text
# Skill improvement — a reflective pass over ONE skill

You are performing a reflective pass over ONE skill, to make it fit how it is
actually being used. Synthesize what recent real usage shows into durable,
well-organized improvements to this skill, so future tasks that use it go better.

## Scope
- You may ONLY improve THIS ONE skill. Do not create new skills; do not touch any other skill.
- A skill is a FOLDER. `SKILL.md` is read first; it may be packaged with three kinds of
  support files, each in its own directory — use the right one:
    • `references/<topic>.md` — session-specific detail (error transcripts, repro recipes,
      provider quirks) AND condensed knowledge banks (quoted research, API-doc excerpts,
      domain notes), written concise and task-focused.
    • `templates/<name>.<ext>` — starter files meant to be copied and modified.
    • `scripts/<name>.<ext>` — statically re-runnable actions (verification scripts, fixture
      generators, probes) the agent runs rather than hand-types.
  Keep the skill CLASS-LEVEL and rich; push session-specific detail into support files. The
  skill shares the context window with everything else, so conciseness matters.
- If you notice this skill overlaps with another skill, just NOTE it in your final report —
  consolidating across skills is the Curator's job, not yours.

## What you are given
- The skill's current folder (SKILL.md + any references).
- A scratch directory containing N ended user sessions where this skill was used, with a
  `manifest` listing them. These are real, COMPLETED sessions.
- A read-only MCP tool for the department knowledge base.
- Tools: read-only file access over the scratch dir and the skill folder; a `skill_manage`
  tool to edit THIS skill (actions: `edit`, `patch`, `write_file`); and the MCP tool.

## Step 1 — go through the sessions (cover EVERY one)
The transcripts are large — do NOT dump whole files into your reasoning. For EACH session
in the manifest, read enough to judge how it went: skim it, and especially check how it
ENDED and whether the user corrected, redid, or rejected the work along the way — those
signals usually appear AFTER the skill produced its output, so do not stop at the skill's
first response. Record a ONE-LINE note per session: {what the task was; did it go well or
poorly; if poorly, what specifically went wrong}.

## Step 2 — find what RECURS across sessions
Compare your notes. Identify the most prevalent, systematic problems (and effective wins)
that appear ACROSS MULTIPLE sessions — NOT individual one-off edge cases.

Signals worth capturing include (any one counts; this is NOT a fixed taxonomy — surface
whatever genuinely recurs):
  • the user corrected your style / tone / format / verbosity / approach (frustration like
    'stop doing X', 'too verbose', 'just give me the answer', 'remember this');
  • the user corrected your workflow or the sequence of steps;
  • a non-trivial technique, fix, workaround, debugging path, or tool-usage pattern emerged;
  • the skill itself was wrong, missing a step, or outdated where it was used.

Do NOT capture (these harden into self-imposed constraints that bite later):
  • environment-dependent failures (missing binaries, 'command not found', unconfigured
    credentials, path mismatches) — the user can fix these; they are not durable rules;
  • negative claims about tools/features ('X is broken', 'cannot use Y') — they become
    refusals the agent cites against itself long after the issue is fixed;
  • session-specific transient errors that resolved before the session ended — if a retry
    worked, the lesson is the retry pattern, not the original failure;
  • one-off task narratives.
  If a tool failed because of setup state, capture the FIX (install/config/env step), never
  'this tool does not work'.

Write each candidate lesson as:
  - a short title and a one-line description;
  - the session IDs it appears in, each with one line of concrete evidence.
A lesson QUALIFIES only if at least TWO different sessions support it.

## Step 3 — decide what to act on
Act ONLY on qualified lessons (supported by ≥2 sessions). If nothing qualifies, make no
changes and say so — do not invent recurrence and do not package one-off, speculative, or
sensitive details. But do not be timid either: when a problem genuinely recurs, fix it.
Leaving a clearly repeated gap unfixed is a miss; a pass that addresses real recurring
problems is the goal.

## Step 4 — improve the skill (via `skill_manage`)
For each qualified lesson, make the SMALLEST change that closes it:
- Prefer reinforcing or extending an existing section over adding a new top-level section.
- Be additive first: never remove guidance that is currently correct and useful.
- Minimal patches are preferred over large rewrites; several small patches beat one big one.
- Match specificity to fragility: where the same problem recurs, TIGHTEN — give an exact
  snippet or a must-follow checklist; where valid approaches genuinely vary, state a
  principle rather than a rigid rule.
- Make every edit generalizable; do not hardcode task-specific values.
- Only fill genuine gaps; do not duplicate guidance the skill already contains.
- Every change must trace back to one of your qualified lessons.

Folder rules:
- A new support file goes in the right directory by kind (`references/` / `templates/` /
  `scripts/`). Whenever you add one, you MUST also add a one-line pointer to it from SKILL.md
  (so future agents know it exists) — never a new file with no pointer, never a pointer to a
  missing file.
- Keep SKILL.md under 500 lines and each reference under 300 lines; push detail into a
  reference and leave a short summary + link in SKILL.md (progressive disclosure).
- Use the imperative form. Do NOT change the YAML frontmatter `name` or `description`.
- Do NOT touch anything between `<!-- SLOW_UPDATE_START -->` and `<!-- SLOW_UPDATE_END -->`.
- Never write secrets, credentials, or private data into the skill.

## Department knowledge — query the MCP tool ON THE SPOT
If closing a lesson plausibly needs department conventions or internal information, QUERY
the MCP knowledge base NOW.
- If authoritative content exists: write the edit to embed a REFERENCE (link/path) + a
  SHORT summary + provenance (source + date). Do not paste large verbatim content.
- If it does not exist, or the MCP tool is unavailable: derive the fix from the sessions if
  it is clear, otherwise skip that lesson. NEVER invent department facts.

## Finish
Report briefly: what you changed, what you deliberately skipped, and which lessons had thin
evidence. If you changed nothing, say so.
```

---

## 一·中 · Prompt 正文（中文译版，不改原意、只为可读）

```text
# 技能改进 —— 对单个 skill 做一次反思性梳理

你正在对一个 skill 做一次反思性梳理，让它更贴合实际的使用方式。把最近真实使用所暴露
的东西，提炼成对这个 skill 持久、组织良好的改进，使日后用到它的任务做得更好。

## 范围
- 你只能改进**这一个 skill**。不要新建 skill；不要碰任何其它 skill。
- 一个 skill 是一个**文件夹**。`SKILL.md` 最先读；它可以打包三类支撑文件，各放各的目录——用对：
    • `references/<topic>.md` —— 会话特定细节（错误转录、复现步骤、provider 怪癖）**以及**浓缩
      知识库（研究摘录、API 文档片段、领域笔记），写得精炼、面向任务价值。
    • `templates/<name>.<ext>` —— 供复制后修改的起手文件（脚手架、known-good 样例）。
    • `scripts/<name>.<ext>` —— 可直接重复运行的动作（校验脚本、夹具生成器、探针），让 agent
      直接跑而不是每次手敲。
  让这个 skill 保持**类级别(class-level)、内容充实**；把会话特定细节挪进支撑文件。它与其它
  一切共用上下文窗口，所以要精炼。
- 若你发现这个 skill 与另一个 skill 有重叠，只需在最终报告里**标记一下**——跨 skill 的合并
  是 Curator 的活，不是你的活。

## 给你的东西
- 该 skill 当前的文件夹（SKILL.md + 若干 references）。
- 一个暂存目录，内含 N 个**用到该 skill 的、已结束的**用户 session，并有一份 `manifest` 列出它们。
  这些都是真实、已完成的 session。
- 一个只读的 MCP 工具，连到部门知识库。
- 工具：对暂存目录和该 skill 文件夹的只读文件访问；用于改**这一个 skill** 的 `skill_manage`
  工具（动作：`edit`、`patch`、`write_file`）；以及 MCP 工具。

## 第 1 步 —— 逐个过 session（每一个都要覆盖）
转录文件很大——**不要把整文件塞进你的推理**。对 manifest 里的**每一个** session，读到足以
判断它进行得如何为止：略读它，**尤其要看它是怎么结束的、过程中用户有没有纠正/重做/否定**这次
工作——这些信号通常出现在 skill 产出结果**之后**，所以不要停在 skill 的第一次回复。为每个
session 记**一行 note**：{任务是什么；进行得好还是差；若差，具体差在哪}。

## 第 2 步 —— 找出跨 session **反复出现**的东西
比对你的 note。找出**跨多个 session 反复出现**的、最普遍且系统性的问题（以及有效的成功打法）
——**不是**一次性的个别边缘情况。

值得捕获的信号包括（任一即可；这**不是**一份固定分类——凡真正反复的都要浮出来）：
  • 用户纠正你的风格/语气/格式/啰嗦程度/做法（frustration，如"别再 X"、"太啰嗦"、"直接给答案"、
    "记住这个"）；
  • 用户纠正你的工作流或步骤顺序；
  • 冒出非平凡的技巧/修复/绕过/调试路径/工具用法；
  • 这个 skill 本身在被用到处错了、缺步骤、或过时了。

**不要捕获**（这些会硬化成将来反咬你的自我约束）：
  • 环境依赖的失败（缺二进制、command not found、没配凭证、路径不符）——用户能修，不是持久规则；
  • 对工具/特性的负面断言（"X 坏了"、"用不了 Y"）——会变成"自我拒绝"，问题修好很久后还在自我引用；
  • session 内、结束前已自愈的瞬时错误——若重试就好了，教训是"重试"模式而非那个原始失败；
  • 一次性的任务叙事。
  若某工具因配置状态而失败，捕获**修复办法**（安装/配置/环境变量那一步），绝不写"这工具用不了"。

每条候选教训写成：
  - 一个短标题 + 一句话描述；
  - 它出现在哪些 session ID，每个附**一行具体证据**。
一条教训只有在**至少两个不同 session** 支持它时，才算**立住**。

## 第 3 步 —— 决定动哪些
**只对立住的教训（≥2 个 session 支持）动手**。如果没有任何一条立住，就**什么都不改并说明**
——不要臆造"反复"，不要把一次性的、投机的或敏感的细节打包进去。但也**别畏缩**：当一个问题
确实反复出现，就修它。把一个明显反复的缺口留着不修，是一种失职；一次能解决真实反复问题的
梳理，才是目标。

## 第 4 步 —— 改进该 skill（用 `skill_manage`）
对每条立住的教训，做能闭合它的**最小改动**：
- 优先**强化或扩展现有的某个小节**，而不是新开一个顶级小节。
- **先加不删**：绝不删除当前正确且有用的指引。
- **宁可多个小补丁，也别整篇重写**；几个小补丁胜过一个大补丁。
- **按脆弱度调松紧**：同一处反复出错 → **收紧**（给精确代码片段或必须逐条走的 checklist）；
  做法确实因情况而异处 → 写成原则，而非死规则。
- 每条改动都要可泛化；不要写死任务特定的值。
- 只填真正的空缺；不要重复 skill 已有的指引。
- 每条改动都要能追溯回你某条立住的教训。

文件夹规则：
- 新建支撑文件要按类型放对目录（`references/` / `templates/` / `scripts/`）。每加一个，**必须
  同时**在 SKILL.md 里加一条指向它的指针（附一句"何时该读/用它"，让未来 agent 知道它存在）
  ——绝不留一个没人指向的新文件，也绝不留一条指向不存在文件的指针。
- SKILL.md 保持在 500 行以内、每个 reference 在 300 行以内；把细节挪进 reference、在 SKILL.md
  留一句摘要 + 链接（渐进式披露）。
- 用祈使句。**不要**改 YAML frontmatter 的 `name` 或 `description`。
- **不要**碰 `<!-- SLOW_UPDATE_START -->` 与 `<!-- SLOW_UPDATE_END -->` 之间的任何内容。
- 绝不把密钥、凭证、私人数据写进 skill。

## 部门知识 —— **当场**查 MCP 工具
若闭合一条教训可能需要部门约定或内部信息，**现在就查** MCP 知识库。
- 若存在权威内容：把改动写成**嵌入一个引用（链接/路径）+ 一句简短摘要 + 出处（来源 + 日期）**。
  不要粘贴大段原文。
- 若不存在、或 MCP 工具不可用：若从 session 里能看清就据此推导，否则跳过该条教训。**绝不编造部门事实。**

## 收尾
简要报告：你改了什么、刻意跳过了什么、哪几条教训证据偏薄。若什么都没改，请说明。
```

---

## 二、逐块溯源表（每块取自哪个参考的 prompt 原文 / 哪条决策）

| Prompt 块 | 取自的参考原话 / 出处 | 我们的改动（决策） |
|---|---|---|
| 开头 "a reflective pass… Synthesize… durable, well-organized" | **auto-dream** Dream prompt 开头（"performing a dream — a reflective pass over your memory files. Synthesize… into durable, well-organized memories"） | memory→**单个 skill**；保留"反思性梳理"框架 |
| skill=文件夹 + **三类支撑文件**（references/templates/scripts）+ class-level | **Hermes** `_SKILL_REVIEW_PROMPT` 原话（"three kinds of support files… references/<topic>.md… templates/<name>… scripts/<name>…"；"CLASS-LEVEL skills with a rich SKILL.md"）+ `skill_manager_tool.py` `ALLOWED_SUBDIRS={references,templates,scripts,assets}` | **修正**：原先抄 Trace2Skill 的"只有 references"太窄；改用 Hermes 的三类（含 assets） |
| Step2 "signals worth capturing"（风格/格式纠正、workflow 纠正、非平凡技巧、skill 本身出错） | **Hermes** `_SKILL_REVIEW_PROMPT` "Signals to look for" 四条原话 | 作为**例子/提示，非固定分类**（呼应早先"4类只是举例"的纠正） |
| Step2 "Do NOT capture"（环境失败/对工具负面断言/已自愈瞬时错/一次性叙事；工具配置失败记修复法） | **Hermes** `_SKILL_REVIEW_PROMPT` "Do NOT capture" 整段原话 | 原样（我原先完全没有这段，补上） |
| Scope "发现重叠只标记，合并交给 Curator" | **Hermes** `_SKILL_REVIEW_PROMPT`（"If you notice two existing skills that overlap, note it… the background curator handles consolidation"） | 跨 skill 合并归 C，不归 B |
| Step4 支撑文件按 kind 放对目录 + SKILL.md 加指针 | **Hermes** `_SKILL_REVIEW_PROMPT`（write_file 三类 + "SKILL.md should gain a one-line pointer"） | 从"只 references"泛化到三类 |
| Scope："ONLY improve THIS ONE skill, do not create" | 我们的决策 | **B 只改不建**（新建归 F2/A/C） |
| Step1 "transcripts are large — do NOT dump whole files… skim… check how it ENDED" | **auto-dream**（"large JSONL files — grep narrowly, don't read whole files… Look only for things you already suspect matter"） | **B2**：纠偏/成败在 episode 后段→读完整 arc 别停在首个回复；**B3**：每段一句 note |
| Step2 "most prevalent, systematic problems… ACROSS MULTIPLE sessions — NOT individual one-off edge cases" | **SkillOpt** `analyst_error.md`（"the most prevalent, systematic failure patterns across them — not individual edge cases"）+ `analyst_success.md`（"patterns that appear across MULTIPLE trajectories"） | failure+success 合一句（**B4** 不偏置成功侧） |
| Step2 教训格式 title/desc/evidence + session IDs | **Trace2Skill** Memory Item（Title/Description/Content）+ 我们的 **A6/A7** | 加"列出 session ID + 每段证据"——**逼扎实 ≥2 的思考纪律，不做代码核验** |
| Step2 "QUALIFIES only if ≥2 sessions" | **Codex**（"at least happened twice, or clearly likely to recur and costly"） | = `support_count≥2` 硬门槛的提示态 |
| Step3 Skip + "do not invent / one-off / speculative / sensitive" | **Codex**（"be conservative… do not create speculative or overlapping assets"；"Skip: too one-off, vague, sensitive"） | **Skip 由 ≥2 隐含**（B5） |
| Step3 "do not be timid… a pass that does nothing is a miss" | **Hermes** background_review（"Be ACTIVE — most sessions produce at least one skill update… a pass that does nothing is a missed learning opportunity"） | **B6**：与 ≥2 调和——该改就改、但只动 ≥2 的；**防 agent 为避险不改** |
| Step4 "Prefer reinforcing existing sections over adding new top-level" | **SkillOpt** `analyst_success.md`（原话） + **Codex** reuse-first | 原样 |
| Step4 "additive first; never remove correct guidance" / "Minimal patches preferred over large rewrites" | **Trace2Skill** `system_prompt_base.txt` Constraints（原话） | 原样（也顺势压住 `edit` 整写，倾向 patch/write_file） |
| Step4 "Match specificity to fragility… TIGHTEN… vs principle" | **Trace2Skill** `modification_strategies_section.txt` Strategy 2 | 原样 |
| Step4 "generalizable; do not hardcode" / "Only fill gaps; do not duplicate" | **SkillOpt** `analyst_error.md`（原话两条） | 原样 |
| Folder rules：create↔link 配对 | **Trace2Skill** `merge_system_prompt.txt` 第7条 + `map_output_format.txt` 文件创建规则 | 原样 |
| Folder rules：<500/<300、progressive disclosure、imperative、不动 frontmatter | **Trace2Skill** `apply_constraints.txt` / `final_consolidation_checklist.txt` / Strategy 5 | 原样 |
| Folder rules：保护区 SLOW_UPDATE 不碰 | **SkillOpt** analyst prompt 末尾（原话） | 原样（保护区由 C 维护） |
| Folder rules：不写 secrets/credentials | **Codex**（"do not package secrets, credentials, private personal data"） | 原样 |
| MCP 块（当场查、查到嵌引用+provenance、查不到/不可达则跳过、绝不编） | 我们的**决策 C** + [b-stage 步1 MCP 块](./b-stage-prompts.md) | spec Q9：降级不报错 |
| Finish 三问 + "if nothing changed, say so" | **Codex** 收尾三问（创建/跳过/还缺什么证据） + **auto-dream**（"Return a brief summary… If nothing changed… say so"） | 原样 |

---

## 三、故意没写进 prompt 的（对照决策 E，防再被人加回去）

| 没写 | 为什么 |
|---|---|
| "检查上次改动有没有效 / 是否复发" | **决策 E**：在线不做效果反馈（Hermes 也不做） |
| 失败指纹、跨批比对、复发回滚、`rejected_edits` | 同上，已在 [01 §4](../01-skillopt.md#4) 删除 |
| "最多改 K 处"的预算 | **B6**：不设改动数预算，跟随 Hermes（防 agent 避险不改）；限流靠 ≥2 + Skip + additive + 大小上限 + 可逆 |
| 强制"先产卡片再改"的两段式机器 | **B5**：不定回合，上面的"步骤"只是指导小节，agent 一回合自然跑 |
| 打标器（success/failure 预分桶） | Agent 自己读时顺带判断，**A 不需要独立打标阶段** |
| **新建 umbrella / class-level 命名规则**（Hermes A 的更新优先级第4步） | **B 只改不建**，新建归 F2/A/C；命名规则是它们的事 |
| **跨 skill 的 consolidation / umbrella 合并** | 归 **Curator(C)**；B 单 skill，只"标记重叠"不合并 |

---

## 四、待打磨 / 上线观察（接 [08 D 段](../08-关键决策记录.md)）

- 措辞精修：Step1"读到够判断为止"的颗粒度（长 session 怎么不超预算又不漏后段纠偏）。
- O1 改动量两头、O5 ≥2 可靠性（漏读/凑数）——上线后据此调 prompt 措辞。
- 触发阈值 X / "什么算一次使用"（C6）由代码侧定，不在本 prompt。
