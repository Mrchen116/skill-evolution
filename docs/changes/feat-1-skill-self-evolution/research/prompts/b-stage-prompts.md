# B（F4·单 skill 批量优化）各步 prompt 草案 v0

> 在 SkillOpt 原文（[skillopt-prompts.md](./skillopt-prompts.md)）基础上改造，落地 spec/design 的 4 个决策：
> A=轻量 LLM 标注器 · B=开放式找缺口（**不做固定分类**，4 例子仅作提示）· C=**内联 MCP**（analyst 自带 MCP 工具当场查，避免"需要/有没有"错配）· D=success 侧保留但弱化。
> **E（2026-06 修正）=不做效果反馈**：对齐 Hermes（最成熟的在线方案，它不衡量"改动有没有效"）——质量靠**入口 ≥2 门槛 + 可逆 + 持续进化收敛 + C 维护**，**删掉了"失败指纹比对/复发回滚/outcome-rejected_edits"**（那是离线 gate 的山寨）。下文凡涉及"步6 自纠/指纹/rejected_edits"的旧草稿均已据此修订。
> prompt 正文用英文（目标 agent 读），中文是注解。**草案,待逐步精修。**

## 管线回顾（哪步是 LLM 调用）

```
0 打标(LLM 标注器)  → 每段轨迹切片 → success/failure/unclear（只为分桶，不抽指纹）
1 analyst_error(LLM+MCP) → failure 桶找共性缺口 → edits（跨会话反复≥2 才采纳）
2 analyst_success(LLM)   → success 桶找共性赢法 → edits（弱化）
3 aggregate(LLM)         → 多 minibatch 合并(失败优先)；只有 1 个 minibatch 时跳过
4 ranking(LLM)           → 选 top-L + support_count≥2 门槛 + reuse-first
5 apply(确定性)          → 先快照→fuzzy 改→安全扫描→fail 回滚
（无"步6 自纠"）质量靠：入口≥2门槛 + 可逆(快照/archive/provenance) + 持续进化收敛 + C维护
                ❌ 不做效果打分/失败指纹比对/复发回滚/outcome-rejected_edits（对齐 Hermes，见 01 §4）
```

**自适应**：若某 skill 触发时 failure 切片总量能塞进上下文（~几条），令 minibatch = 整桶 → analyst 一次看全、直接产 support_count，**跳过 aggregate**。SkillOpt 切 minibatch+aggregate 是为 40 题大 batch；我们单 skill 量小，能合则合。

---

## 步 0 · 轨迹标注器（NEW，SkillOpt 无——它用数据集 ground-truth）

**第一性原理**：在线无 label，要把该 skill 的每段轨迹切片判成 success/failure，**只为给 analyst 分桶**。（早期草稿还让它抽"失败指纹"做复发比对——**已按决策 E 删除**，我们不做效果反馈。）

```text
You are a trajectory tagger for an online skill-evolution system.

You are given:
- The skill being evaluated (name + current SKILL.md).
- ONE trajectory slice: the portion of a real, ENDED user session where this skill
  was used — the user's messages, the agent's tool calls and observations, and how it ended.

Decide whether, in THIS slice, the task this skill was meant to help with went WELL or
POORLY, from the USER's point of view:
- POORLY (failure): the user corrected / redid / rejected the work ("no", "redo",
  "that's not what I meant", "you should use X"), showed frustration, the agent hit
  repeated tool errors/retries, or the task was abandoned.
- WELL (success): completed smoothly, no rework, no correction.
If you cannot tell, answer "unclear" rather than guess.

Respond ONLY with JSON (no fences):
{ "label": "success" | "failure" | "unclear",
  "evidence": "<one line: what in the slice led to this label>" }
```
注：`unclear` 的不进任一桶（不污染分析）。**不再抽取/存储失败指纹**（决策 E）。

---

## 步 1 · failure 分析（gap-finder，SkillOpt `analyst_error.md` 改造）

**改了什么**（对照 [skillopt-prompts.md] 的 `analyst_error.md`）：
1. **单 skill 上下文**：明确"你只能改这一个 skill"，给它全文。
2. **开放式找缺口**（决策 B）：把"找共性失败模式"重述成"比对 skill 现状 vs 轨迹实际所需，找反复缺口"；4 例子作**提示**，明令"别局限于此"。
3. **内联 MCP**（决策 C）：给它 MCP 只读工具，指示"疑似缺部门知识就**当场查**，查到才嵌引用、查不到就跳过或走轨迹推导"——解决错配。
4. **support_count 直接产**：analyst 看整个 minibatch，每条缺口直接给 support_count（SkillOpt 是 merge 才打；单 skill 量小，提前产更省一次 merge 推断）。
5. ~~rejected_edits 负反馈注入~~ **（决策 E 删除）**：在线不做效果反馈，无"被否改动"来源。
6. 保留 SkillOpt 原文约束：≤L、generalizable、不重复已有、不碰保护区。

```text
You are a skill-improvement analyst for ONE specific skill in an online system.
You may ONLY edit this one skill.

You are given:
- The skill's current full SKILL.md (the only document you may edit).
- A MINIBATCH of real trajectories where this skill was used and the task went POORLY,
  taken from one user's recent ENDED sessions.
- TOOLS: read-only access to the department knowledge base via MCP.

GOAL: find the most important COMMON gaps across the batch — places where the skill's
current guidance was insufficient and caused the SAME kind of problem to recur across
MULTIPLE trajectories (not one-off edge cases) — and propose a concise set of edits that
close them.

HOW TO FIND GAPS (first principles): compare what the skill CURRENTLY says against what
the trajectories show the user actually needed. A gap may show up as a missing convention,
an unhandled edge/empty/failure case, an incomplete or missing workflow step, or missing
department/domain knowledge — but do NOT limit yourself to these categories; surface
whatever genuinely recurs.

DEPARTMENT KNOWLEDGE (use MCP on the spot): if closing a gap plausibly needs
department-specific conventions/internal information, QUERY the MCP knowledge base NOW to
check whether authoritative content actually exists.
- If it exists: write the edit so it embeds a REFERENCE (link/path) + a SHORT SUMMARY +
  provenance (source + date). Do NOT paste large verbatim content.
- If it does NOT exist: either propose a trajectory-derived edit if the fix is clear from
  the trajectories alone, or skip that gap. NEVER invent department facts.

CONSTRAINTS:
- Only address gaps NOT already covered; do not duplicate existing skill content.
- Edits must be generalizable; no task-specific hardcoding.
- Produce AT MOST L edits (you will be told L), highest-impact first; fewer is fine.
- Do NOT target or modify anything between <!-- SLOW_UPDATE_START --> and
  <!-- SLOW_UPDATE_END --> (a protected section maintained elsewhere).

Respond ONLY with JSON (no fences):
{ "batch_size": <n>,
  "common_gaps": [ {"gap": "<one line>", "support_count": <#trajectories showing it>} ],
  "patch": { "reasoning": "<why these edits close the batch's common gaps>",
    "edits": [
      { "op": "append|insert_after|replace|delete",
        "target": "<exact anchor text in SKILL.md, for insert_after/replace/delete>",
        "content": "<markdown>",
        "support_count": <#trajectories this edit addresses>,
        "knowledge_ref": "<MCP source+date if this edit used department knowledge, else empty>" } ] } }
"edits" may be empty if no warranted gap.
```

## 步 2 · success 分析（弱化，SkillOpt `analyst_success.md` 改造）

**改了什么**：同样给单 skill 上下文；目标=找**反复有效的赢法**去强化现有 section（"prefer reinforcing existing sections over adding new"，SkillOpt 原话保留）。**弱化**=预算更小（如 L_success=1）、且 ranking 阶段 failure 补丁优先。一般不需要 MCP。

```text
You are a success-pattern analyst for ONE specific skill.

Given the skill's current SKILL.md and a MINIBATCH of trajectories where this skill was
used and the task went WELL, identify generalizable behavior patterns COMMON across the
batch that are worth encoding to make good outcomes more reliable.

RULES:
- Only propose patches for patterns NOT already covered.
- Focus on patterns appearing across MULTIPLE trajectories; must generalize beyond
  specific tasks.
- Prefer REINFORCING existing sections over adding new top-level sections.
- AT MOST L_success edits (you'll be told the budget). Fewer is fine; empty if already covered.
- Do NOT touch the <!-- SLOW_UPDATE_* --> protected section.

Respond ONLY with JSON: same schema as the failure analyst (common_gaps→success_patterns,
each with support_count; edits with op/target/content/support_count).
```

## 步 3 · aggregate（SkillOpt `merge_failure/success/final.md`，基本照搬）

**何时用**：每桶 > 1 个 minibatch 时才需要（把多个 minibatch 的 patch 合成一个）。**失败优先于成功**（SkillOpt `aggregate.py` 原则）。只有 1 个 minibatch 时**跳过**，直接进 ranking。
**改动**：几乎无；只需把我们 analyst 多出的 `knowledge_ref` 字段在合并时**透传保留**。

## 步 4 · ranking（SkillOpt `ranking.md` + 两处加法）

**改了什么**：
1. **support_count≥2 硬门槛**（Codex"≥2 次"）：先丢弃 support_count<2 的 edit，再排序选 top-L。
2. **reuse-first / 不重复**（Codex + skill-creator 纪律）：明确"优先强化/扩展 skill 现有 section，而非堆新顶级 section；与现有内容重复的降到最低"。
其余排序准则照 SkillOpt 原文（systematic impact > complementarity > generality > actionability）。

```text
You receive ONE skill's current document and a pool of proposed edits (each with a
support_count = how many trajectories support it). First DROP every edit with
support_count < 2. Then RANK the remaining edits and select the top L.

Ranking criteria (priority order):
1. Systematic impact — edits fixing widespread, recurring problems rank highest.
2. Complementarity — edits filling gaps (not duplicating existing skill content) rank higher;
   prefer reinforcing/extending existing sections over adding new top-level sections.
3. Generality — general principles over entity-specific advice.
4. Actionability — concrete guidance over vague advice.

Respond ONLY with JSON:
{ "reasoning": "...", "dropped_low_support": [<indices dropped for support<2>],
  "selected_indices": [<0-based indices of top edits, priority order>] }
```

## ~~步 6 · 自纠~~ → 已删除（决策 E）：在线不做效果反馈

早期草稿在这里设计了"失败指纹比对 + 复发回滚 + rejected_edits 回灌"——这是把 SkillOpt 的离线 gate/buffer 山寨到在线。**已删除**，理由（一手确认，详见 [01 §4](../01-skillopt.md#4)）：

- 最成熟的在线方案 **Hermes 刻意不衡量"改动有没有效"**（usage 只记次数、Curator 只按时间衰减、A 改完不回头看）。
- 在线无 gate → 无"被否改动"来源；失败指纹是 LLM 自由文本，跨批比对两端都模糊、无 ground truth。

**取而代之的在线质量保障**（apply 前后只剩这些，全部确定性、无新 prompt）：
- 每次 apply 前 `SkillStore.snapshot`（可逆底座）；archive 不硬删；provenance 护 seed/user。
- 质量挡在**入口**：步 4 的 `support_count≥2` 门槛 + reuse-first。
- 坏改动不长期生效，靠**持续进化自然收敛**（真实需求反复产证据，下一轮继续朝它改）+ 可随时 archive 回退，**不靠系统主动测效果回滚**。

---

## 待精修 / 待定（下一轮）

- [ ] 标注器对"多轮长 session 切片"的边界（一段 session 多次用同一 skill 怎么切）。
- [ ] `L`、`L_success`、minibatch M、support_count 阈值的具体默认值（暂用 L=4 / L_success=1 / M=8 / 阈值2）。
- [ ] MCP 查询的 query 怎么由 analyst 构造（自由 query vs 受控检索接口）。
- [ ] 保护区（SLOW_UPDATE）由 C 维护的 prompt（slow_update.md 改造）——属 C，不在本文。
