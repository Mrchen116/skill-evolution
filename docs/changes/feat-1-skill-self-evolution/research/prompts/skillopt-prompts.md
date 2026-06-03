# SkillOpt Prompts — 逐字提取（Batch Optimizer 参考）

> 来源（只读）：`/Users/czj/Repos/opensource-hub/self-evolution/SkillOpt/skillopt/prompts/`
> 本文档逐字（VERBATIM）抄录所有 prompt 文件，按 **pipeline 阶段**分组，使同一阶段的 patch / rewrite / full_rewrite 三种变体差异可见。所有 prompt 文本保持英文原样，未做翻译/改写/截断。

---

## 0. prompt 加载机制与 update_mode 映射

### 加载机制（`__init__.py` 的 `load_prompt`）

- prompt 以 `.md` 文件存储，运行时按名加载。
- `load_prompt(name, env)` 的查找顺序：
  1. `skillopt/envs/{env}/prompts/{name}.md`（若提供 env，env-specific override）
  2. `skillopt/prompts/{name}.md`（generic 默认 fallback）
  3. 都不存在则抛 `FileNotFoundError`。
- 文件内容带进程级 `_cache`（`clear_cache()` 可清空，测试用）。
- 关键：**prompt 名（`name`）就是文件名去掉 `.md`**。没有额外的 naming map / 别名表——文件名即 key。变体通过命名后缀区分：`*`（patch）、`*_rewrite`、`*_full_rewrite`。

### update_mode 三个变体（`optimizer/update_modes.py`）

三个常量及其归一化别名：

| update_mode 常量 | 值 | 命名后缀 | 别名（`normalize_update_mode`） | payload key（`payload_key`） |
|---|---|---|---|---|
| `PATCH_MODE` | `"patch"` | 无后缀（plain name） | `patch`, `edits` | `edits` |
| `REWRITE_MODE` | `"rewrite_from_suggestions"` | `*_rewrite` | `rewrite`, `rewrite_from_suggestions`, `suggestions`, `rewrite_suggestions` | `revise_suggestions` |
| `FULL_REWRITE_MINIBATCH_MODE` | `"full_rewrite_minibatch"` | `*_full_rewrite` | `full_rewrite`, `full_rewrite_minibatch`, `minibatch_full_rewrite`, `skill_rewrite_minibatch` | `skill_candidates` |

- 默认 / 兜底为 `patch`（未知字符串归一化为 `PATCH_MODE`）。
- payload 标签（`payload_label`）：patch→"edit(s)"，rewrite→"suggestion(s)"，full_rewrite→"skill candidate(s)"。
- `describe_item` / `short_item_summary` 针对三种 mode 分别提取字段：
  - patch 项字段：`op` / `target` / `content`（+ `support_count`）。
  - rewrite 项字段：`type` / `title` / `instruction`（+ `priority_hint` / `support_count`）。
  - full_rewrite 项字段：`title` / `change_summary` / `new_skill`（+ `source_type` / `support_count`）。

### 阶段 → 文件 → update_mode 总览表

| Pipeline 阶段 | patch（plain） | rewrite（`*_rewrite`） | full_rewrite（`*_full_rewrite`） |
|---|---|---|---|
| Reflect-failure-analyst | `analyst_error.md` | `analyst_error_rewrite.md` | `analyst_error_full_rewrite.md` |
| Reflect-success-analyst | `analyst_success.md` | `analyst_success_rewrite.md` | `analyst_success_full_rewrite.md` |
| Aggregate-merge-failure | `merge_failure.md` | `merge_failure_rewrite.md` | `merge_failure_full_rewrite.md` |
| Aggregate-merge-success | `merge_success.md` | `merge_success_rewrite.md` | `merge_success_full_rewrite.md` |
| Aggregate-merge-final | `merge_final.md` | `merge_final_rewrite.md` | `merge_final_full_rewrite.md` |
| Select-ranking | `ranking.md` | `ranking_rewrite.md` | —（无 full_rewrite 变体） |
| Update-rewrite-skill | —（仅 rewrite 路径有此阶段） | `rewrite_skill.md` | — |
| SlowUpdate | `slow_update.md`（单一，mode 无关） | — | — |
| MetaSkill | `meta_skill.md`（单一，mode 无关） | — | — |
| LR-autonomous | `lr_autonomous.md`（单一，mode 无关） | — | — |

> 注意：`rewrite_skill.md` 是 rewrite 模式专属的"用 suggestions 重写整篇 skill"阶段；full_rewrite 模式不需要它，因为 full_rewrite 的 analyst/merge 阶段已直接产出完整 `new_skill`。`ranking` 没有 `*_full_rewrite` 变体（full_rewrite 每个 analyst/merge 都强制只返回一个 candidate，无需 ranking 选择）。

### 几条关键 verbatim 规则的出处速查

- **AT MOST L edits/suggestions（编辑预算 L）**：出现在 patch 与 rewrite 的两个 analyst（`analyst_error*` / `analyst_success*`），原文 `Produce AT MOST L edits/suggestions`。full_rewrite analyst 不提 L，改为 `Return exactly one item in "skill_candidates"`。
- **COMMON patterns across a minibatch（找共性而非边缘个案）**：`analyst_error*` 的 `address the COMMON patterns — not individual edge cases`；`analyst_success*` 的 `patterns that appear across MULTIPLE trajectories`。
- **JSON 输出 schema `op/target/content`**：patch 链路（`analyst_error.md` / `analyst_success.md` / `merge_failure.md` / `merge_success.md` / `merge_final.md`）的 `edits[]` 项。merge 阶段额外带 `support_count` 与 `source_type`。
- **`<!-- SLOW_UPDATE_START/END -->` 受保护区规则**：出现在所有 patch 与 full_rewrite prompt，以及 rewrite analyst（`analyst_error_rewrite.md`）与 `rewrite_skill.md`。patch 链路措辞为 "Do NOT propose/merge any edits that target content within these markers"；full_rewrite/rewrite 措辞为 "keep that block unchanged" / "Do not modify ... except to keep it intact"。值得注意：**rewrite 的 merge 阶段**（`merge_failure_rewrite.md` / `merge_success_rewrite.md` / `merge_final_rewrite.md`）与 `ranking*` 都**未**提及该保护区规则。

---

## 1. Reflect — Failure Analyst（`analyst_error*`）

每个 trajectory 失败后的逐 step 失败分析。三个变体差异：patch 产出 `edits[]`（op/target/content），rewrite 产出 `revise_suggestions[]`，full_rewrite 直接产出完整 `new_skill`（且只一个 candidate、不提 L 预算）。

### 1a. `analyst_error.md` — Reflect-failure-analyst / **patch**

```
You are an expert failure-analysis agent for AI agent tasks.

You will be given MULTIPLE failed agent trajectories from a single minibatch
and the current skill document.
Your job is to identify the most important COMMON failure patterns across
the batch and propose a concise set of skill edits.

## Analysis Process
1. Read ALL trajectories in the minibatch.
2. Identify the most prevalent, systematic failure patterns across them.
3. For each pattern, classify its failure type.
4. Propose skill edits that address the COMMON patterns — not individual edge cases.
5. Edits must be generalizable; do not hardcode task-specific values.
6. Only patch gaps in the skill — do not duplicate existing content.

You will be told the maximum number of edits (the budget L). Produce AT MOST L edits,
focusing on the highest-impact patterns. You may produce fewer if warranted.

Respond ONLY with a valid JSON object (no markdown fences, no extra text):
{
  "batch_size": <number of trajectories analysed>,
  "failure_summary": [
    {"failure_type": "<type>", "count": <int>, "description": "<one-line>"}
  ],
  "patch": {
    "reasoning": "<why these edits address the batch's common failures>",
    "edits": [
      {"op": "append",       "content": "<markdown to add at end of skill>"},
      {"op": "insert_after", "target": "<exact heading/text to insert after>", "content": "<markdown>"},
      {"op": "replace",      "target": "<exact text to replace>",              "content": "<replacement>"},
      {"op": "delete",       "target": "<exact text to remove>"}
    ]
  }
}
Only include edits that are needed. "edits" can be an empty list if no patch is warranted.

IMPORTANT: The skill document may contain a section between
<!-- SLOW_UPDATE_START --> and <!-- SLOW_UPDATE_END --> markers.
This is a PROTECTED section managed by a separate slow-update process.
Do NOT propose any edits that target, modify, or delete content within
these markers.
```

### 1b. `analyst_error_rewrite.md` — Reflect-failure-analyst / **rewrite**

```
You are an expert failure-analysis agent for AI agent tasks.

You will be given MULTIPLE failed agent trajectories from a single minibatch
and the current skill document.
Your job is to identify the most important COMMON failure patterns across
the batch and propose a concise set of skill-revision suggestions.

## Analysis Process
1. Read ALL trajectories in the minibatch.
2. Identify the most prevalent, systematic failure patterns across them.
3. For each pattern, classify its failure type.
4. Propose revision suggestions that address the COMMON patterns, not individual edge cases.
5. Suggestions must be generalizable and should help a later optimizer rewrite the full skill document.
6. Do not hardcode task-specific values.

You will be told the maximum number of suggestions (the budget L). Produce AT MOST L suggestions,
focusing on the highest-impact patterns. You may produce fewer if warranted.

Respond ONLY with a valid JSON object (no markdown fences, no extra text):
{
  "batch_size": <number of trajectories analysed>,
  "failure_summary": [
    {"failure_type": "<type>", "count": <int>, "description": "<one-line>"}
  ],
  "patch": {
    "reasoning": "<why these suggestions address the batch's common failures>",
    "revise_suggestions": [
      {
        "type": "add_rule|remove_rule|merge_rules|reorganize|compress|clarify",
        "title": "<short title>",
        "motivation": "<why this matters>",
        "instruction": "<what the rewriting optimizer should change in the skill>",
        "priority_hint": "high|medium|low"
      }
    ]
  }
}
"revise_suggestions" may be an empty list if no revision is warranted.

IMPORTANT: The skill document may contain a section between
<!-- SLOW_UPDATE_START --> and <!-- SLOW_UPDATE_END --> markers.
This is a PROTECTED section managed by a separate slow-update process.
Do NOT propose suggestions that target, modify, or delete content within
these markers.
```

### 1c. `analyst_error_full_rewrite.md` — Reflect-failure-analyst / **full_rewrite**

```
You will be given several failed agent trajectories from one minibatch and the current skill document.

Summarize the lessons from these trajectories into one complete replacement skill document.

When rewriting from a minibatch, use the current trajectories as the primary
evidence for updates. Preserve essential task-format instructions, but avoid mechanically carrying over
stale, redundant, or conflicting rules. Prefer a concise, coherent replacement
skill over a long document with weakly supported guidance.

Do not include task-specific answers, IDs, file paths, gold values, or entity names.
If the skill contains a protected block between <!-- SLOW_UPDATE_START --> and
<!-- SLOW_UPDATE_END -->, keep that block unchanged.

Respond ONLY with a valid JSON object:
{
  "batch_size": <number of trajectories analysed>,
  "failure_summary": [
    {"failure_type": "<type>", "count": <int>, "description": "<one-line>"}
  ],
  "patch": {
    "reasoning": "<brief summary of the rewrite>",
    "skill_candidates": [
      {
        "title": "<short title>",
        "change_summary": ["<short change 1>", "<short change 2>"],
        "new_skill": "<complete rewritten skill document>"
      }
    ]
  }
}

Return exactly one item in "skill_candidates".
```

---

## 2. Reflect — Success Analyst（`analyst_success*`）

成功 trajectory 的共性模式提炼。三变体与 failure analyst 对称，但用 `success_patterns` 替换 `failure_summary`，且更强调保守加固（reinforce existing behavior）。

### 2a. `analyst_success.md` — Reflect-success-analyst / **patch**

```
You are an expert success-pattern analyst for AI agents.

You will be given MULTIPLE successful agent trajectories from a single minibatch
and the current skill document. Your job is to identify generalizable behavior
patterns that are COMMON across the batch and worth encoding in the skill.

## Rules
- Only propose patches for patterns NOT already covered in the skill.
- Focus on patterns that appear across MULTIPLE trajectories in the batch.
- Be concise. Patterns must generalize beyond specific tasks.
- Prefer reinforcing existing sections over adding new top-level sections.

You will be told the maximum number of edits (the budget L). Produce AT MOST L edits,
focusing on the most broadly applicable patterns. You may produce fewer if warranted.

Respond ONLY with a valid JSON object:
{
  "batch_size": <number of trajectories analysed>,
  "success_patterns": ["<pattern 1>", "<pattern 2>"],
  "patch": {
    "reasoning": "<why these patterns are worth encoding>",
    "edits": [
      {"op": "append",       "content": "<markdown>"},
      {"op": "insert_after", "target": "<heading/text>", "content": "<markdown>"},
      {"op": "replace",      "target": "<old text>",     "content": "<new text>"},
      {"op": "delete",       "target": "<exact text to remove>"}
    ]
  }
}
"edits" may be empty if the skill already covers all observed patterns.

IMPORTANT: The skill document may contain a section between
<!-- SLOW_UPDATE_START --> and <!-- SLOW_UPDATE_END --> markers.
This is a PROTECTED section managed by a separate slow-update process.
Do NOT propose any edits that target, modify, or delete content within
these markers.
```

### 2b. `analyst_success_rewrite.md` — Reflect-success-analyst / **rewrite**

```
You are an expert success-pattern analyst for AI agent tasks.

You will be given MULTIPLE successful agent trajectories from a single minibatch
and the current skill document. Your job is to identify broadly useful patterns
worth preserving in a later full-skill rewrite.

## Rules
- Only propose revise_suggestions for patterns NOT already covered in the skill.
- Focus on patterns that appear across MULTIPLE trajectories in the batch.
- Keep suggestions general, concise, and rewrite-friendly.
- Prefer guidance that improves organization, clarity, or reusable behavior.

You will be told the maximum number of suggestions (the budget L). Produce AT MOST L suggestions,
focusing on the most broadly applicable patterns. You may produce fewer if warranted.

Respond ONLY with a valid JSON object:
{
  "batch_size": <number of trajectories analysed>,
  "success_patterns": ["<pattern 1>", "<pattern 2>"],
  "patch": {
    "reasoning": "<why these suggestions are worth encoding>",
    "revise_suggestions": [
      {
        "type": "add_rule|remove_rule|merge_rules|reorganize|compress|clarify",
        "title": "<short title>",
        "motivation": "<why this matters>",
        "instruction": "<what the rewriting optimizer should change in the skill>",
        "priority_hint": "high|medium|low"
      }
    ]
  }
}
"revise_suggestions" may be empty if the skill already captures all useful patterns.
```

> 注意：success_rewrite **未**提及 SLOW_UPDATE 保护区（与 error_rewrite 不同）。

### 2c. `analyst_success_full_rewrite.md` — Reflect-success-analyst / **full_rewrite**

```
You will be given several successful agent trajectories from one minibatch and the current skill document.

Summarize any useful lessons from these trajectories into one complete replacement skill document.

When rewriting from a minibatch, use the current trajectories as the primary
evidence for updates. Preserve essential task-format instructions, but avoid mechanically carrying over
stale, redundant, or conflicting rules. Prefer a concise, coherent replacement
skill over a long document with weakly supported guidance.

Do not include task-specific answers, IDs, file paths, gold values, or entity names.
If the skill contains a protected block between <!-- SLOW_UPDATE_START --> and
<!-- SLOW_UPDATE_END -->, keep that block unchanged.

Respond ONLY with a valid JSON object:
{
  "batch_size": <number of trajectories analysed>,
  "success_patterns": ["<pattern 1>", "<pattern 2>"],
  "patch": {
    "reasoning": "<brief summary of the rewrite>",
    "skill_candidates": [
      {
        "title": "<short title>",
        "change_summary": ["<short change 1>", "<short change 2>"],
        "new_skill": "<complete rewritten skill document>"
      }
    ]
  }
}

Return exactly one item in "skill_candidates".
```

---

## 3. Aggregate — Merge Failure（`merge_failure*`）

合并多个独立的 FAILURE 分析产出为一个连贯 patch/suggestion-set/candidate。强调去重、冲突解决、保留独特纠错洞见、prevalent-pattern bias、估计 `support_count`。

### 3a. `merge_failure.md` — Aggregate-merge-failure / **patch**

```
You are a skill-edit coordinator. You receive multiple independently-proposed patches
from FAILURE analysis of agent trajectories. Merge them into ONE coherent, non-redundant patch.

Merge guidelines:
1. **Deduplicate**: keep the best-worded version of similar edits.
2. **Resolve conflicts**: if patches contradict on the same point,
   choose the one with stronger justification or synthesize both.
3. **Preserve unique insights**: include all non-redundant corrective edits.
4. **Prevalent-pattern bias**: edits appearing consistently across multiple patches
   address systematic failures — preserve them with HIGH priority.
   Edits from only one patch may be discarded if task-specific.
5. **Independence**: no two edits in the merged patch may target the same text region.
6. **Support count**: for each merged edit, estimate how many source patches support it.
7. **PROTECTED SECTION**: The skill may contain a section between
   <!-- SLOW_UPDATE_START --> and <!-- SLOW_UPDATE_END --> markers.
   Do NOT merge or produce any edits that target content within these markers.

Respond ONLY with a valid JSON object:
{
  "reasoning": "<summary of key consolidation decisions>",
    "edits": [
    {
      "op": "append|insert_after|replace|delete",
      "target": "<if insert_after or replace or delete>",
      "content": "<markdown>",
      "support_count": <integer>,
      "source_type": "failure"
    }
  ]
}
```

### 3b. `merge_failure_rewrite.md` — Aggregate-merge-failure / **rewrite**

```
You are a skill-revision coordinator. You receive multiple independently-proposed
revision suggestion sets from FAILURE analysis of agent trajectories. Merge them
into ONE coherent, non-redundant set of revise_suggestions.

Merge guidelines:
1. Deduplicate overlapping suggestions.
2. Resolve conflicts by keeping the more general, better-justified direction.
3. Preserve unique high-impact corrective insights.
4. Suggestions supported by many source patches should receive higher support_count.
5. The output suggestions should help a later optimizer rewrite the full skill.

Respond ONLY with a valid JSON object:
{
  "reasoning": "<summary of consolidation decisions>",
  "revise_suggestions": [
    {
      "type": "add_rule|remove_rule|merge_rules|reorganize|compress|clarify",
      "title": "<short title>",
      "motivation": "<why this matters>",
      "instruction": "<what the rewriting optimizer should change in the skill>",
      "priority_hint": "high|medium|low",
      "support_count": <integer>,
      "source_type": "failure"
    }
  ]
}
```

### 3c. `merge_failure_full_rewrite.md` — Aggregate-merge-failure / **full_rewrite**

```
You will be given complete skill candidates written from failed trajectories and the current skill document.

Combine them into one complete replacement skill document.

When merging full-skill candidates, preserve essential task-format instructions,
but do not mechanically retain stale, redundant, or
conflicting rules. If candidates disagree, prefer the concise rule with clearer
trajectory support and better consistency with the replacement skill.

Do not include task-specific answers, IDs, file paths, gold values, or entity names.
If the current skill contains a protected block between <!-- SLOW_UPDATE_START --> and
<!-- SLOW_UPDATE_END -->, keep that block unchanged.

Respond ONLY with a valid JSON object:
{
  "reasoning": "<brief summary of how the candidates were combined>",
  "skill_candidates": [
    {
      "title": "<short title>",
      "change_summary": ["<short change 1>", "<short change 2>"],
      "new_skill": "<complete merged skill document>",
      "support_count": <integer>,
      "source_type": "failure"
    }
  ]
}

Return exactly one item in "skill_candidates".
```

---

## 4. Aggregate — Merge Success（`merge_success*`）

合并 SUCCESS 分析产出。比 failure 版更保守（"success-driven patches reinforce existing behavior. Only include edits for patterns NOT already in the skill"）。

### 4a. `merge_success.md` — Aggregate-merge-success / **patch**

```
You are a skill-edit coordinator. You receive multiple independently-proposed patches
from SUCCESS analysis of agent trajectories. Merge them into ONE coherent patch
that reinforces effective patterns.

Merge guidelines:
1. **Deduplicate**: keep only the most generalizable version of similar patterns.
2. **Be conservative**: success-driven patches reinforce existing behavior.
   Only include edits for patterns NOT already in the skill.
3. **Prevalent-pattern bias**: patterns seen across many successful trajectories
   are most worth encoding.
4. **Support count**: estimate how many source patches support each merged edit.
5. **PROTECTED SECTION**: The skill may contain a section between
   <!-- SLOW_UPDATE_START --> and <!-- SLOW_UPDATE_END --> markers.
   Do NOT merge or produce any edits that target content within these markers.

Respond ONLY with a valid JSON object:
{
  "reasoning": "<summary>",
  "edits": [
    {
      "op": "append|insert_after|replace|delete",
      "target": "<if needed>",
      "content": "<markdown>",
      "support_count": <integer>,
      "source_type": "success"
    }
  ]
}
```

### 4b. `merge_success_rewrite.md` — Aggregate-merge-success / **rewrite**

```
You are a skill-revision coordinator. You receive multiple independently-proposed
revision suggestion sets from SUCCESS analysis of agent trajectories. Merge them
into ONE coherent, non-redundant set of revise_suggestions.

Merge guidelines:
1. Deduplicate overlapping success patterns.
2. Be conservative: only keep suggestions that reinforce useful behavior not already well-covered.
3. Suggestions supported by many source patches should receive higher support_count.
4. The output suggestions should help a later optimizer rewrite the full skill.

Respond ONLY with a valid JSON object:
{
  "reasoning": "<summary>",
  "revise_suggestions": [
    {
      "type": "add_rule|remove_rule|merge_rules|reorganize|compress|clarify",
      "title": "<short title>",
      "motivation": "<why this matters>",
      "instruction": "<what the rewriting optimizer should change in the skill>",
      "priority_hint": "high|medium|low",
      "support_count": <integer>,
      "source_type": "success"
    }
  ]
}
```

### 4c. `merge_success_full_rewrite.md` — Aggregate-merge-success / **full_rewrite**

```
You will be given complete skill candidates written from successful trajectories and the current skill document.

Combine them into one complete replacement skill document.

When merging full-skill candidates, preserve essential task-format instructions,
but do not mechanically retain stale, redundant, or
conflicting rules. If candidates disagree, prefer the concise rule with clearer
trajectory support and better consistency with the replacement skill.

Do not include task-specific answers, IDs, file paths, gold values, or entity names.
If the current skill contains a protected block between <!-- SLOW_UPDATE_START --> and
<!-- SLOW_UPDATE_END -->, keep that block unchanged.

Respond ONLY with a valid JSON object:
{
  "reasoning": "<brief summary of how the candidates were combined>",
  "skill_candidates": [
    {
      "title": "<short title>",
      "change_summary": ["<short change 1>", "<short change 2>"],
      "new_skill": "<complete merged skill document>",
      "support_count": <integer>,
      "source_type": "success"
    }
  ]
}

Return exactly one item in "skill_candidates".
```

---

## 5. Aggregate — Merge Final（`merge_final*`）

最终合并：把 failure 组与 success 组合并。核心规则——**FAILURE PATCHES TAKE PRIORITY**；更高层级（survived previous merge rounds）的 edit 代表更广共识、优先。

### 5a. `merge_final.md` — Aggregate-merge-final / **patch**

```
You are a skill-edit coordinator performing the FINAL merge. You receive two
pre-merged patch groups:
1. **Failure-driven patches** (corrective, high priority)
2. **Success-driven patches** (reinforcement, lower priority)

Merge guidelines:
1. **FAILURE PATCHES TAKE PRIORITY**: the primary goal of skill reflection is to
   fix failures. Failure-driven edits should be preserved unless they directly
   conflict with a well-supported success pattern.
2. **Deduplicate**: if a failure edit and success edit cover the same point,
   keep the failure version.
3. **Preserve success insights**: include success edits that cover patterns
   NOT addressed by failure edits.
4. **Higher-level merges represent broader consensus**: edits that survived
   previous merge rounds (higher level) should be given priority.
5. **Carry forward support_count and source_type for each edit.**
6. **PROTECTED SECTION**: The skill may contain a section between
   <!-- SLOW_UPDATE_START --> and <!-- SLOW_UPDATE_END --> markers.
   Do NOT merge or produce any edits that target content within these markers.

Respond ONLY with a valid JSON object:
{
  "reasoning": "<summary of priority decisions>",
  "edits": [
    {
      "op": "append|insert_after|replace|delete",
      "target": "<if needed>",
      "content": "<markdown>",
      "support_count": <integer>,
      "source_type": "failure|success"
    }
  ]
}
```

### 5b. `merge_final_rewrite.md` — Aggregate-merge-final / **rewrite**

```
You are a skill-revision coordinator performing the FINAL merge. You receive:
1. Failure-driven revise_suggestions (higher priority)
2. Success-driven revise_suggestions (lower priority)

Merge guidelines:
1. Failure-driven suggestions take priority when they overlap.
2. Keep success-driven suggestions that add distinct value.
3. Prefer general, rewrite-friendly, non-redundant suggestions.
4. Carry forward support_count and source_type.

Respond ONLY with a valid JSON object:
{
  "reasoning": "<summary of priority decisions>",
  "revise_suggestions": [
    {
      "type": "add_rule|remove_rule|merge_rules|reorganize|compress|clarify",
      "title": "<short title>",
      "motivation": "<why this matters>",
      "instruction": "<what the rewriting optimizer should change in the skill>",
      "priority_hint": "high|medium|low",
      "support_count": <integer>,
      "source_type": "failure|success"
    }
  ]
}
```

### 5c. `merge_final_full_rewrite.md` — Aggregate-merge-final / **full_rewrite**

```
You will be given complete skill candidates and the current skill document.

Combine them into one complete replacement skill document.

When merging full-skill candidates, preserve essential task-format instructions,
but do not mechanically retain stale, redundant, or
conflicting rules. Prefer concise guidance with clear trajectory support and
better consistency with the replacement skill.

Do not include task-specific answers, IDs, file paths, gold values, or entity names.
If the current skill contains a protected block between <!-- SLOW_UPDATE_START --> and
<!-- SLOW_UPDATE_END -->, keep that block unchanged.

Respond ONLY with a valid JSON object:
{
  "reasoning": "<brief summary of how the candidates were combined>",
  "skill_candidates": [
    {
      "title": "<short title>",
      "change_summary": ["<short change 1>", "<short change 2>"],
      "new_skill": "<complete final skill document>",
      "support_count": <integer>,
      "source_type": "failure|success|mixed"
    }
  ]
}

Return exactly one item in "skill_candidates".
```

---

## 6. Select — Ranking（`ranking*`）

对候选 edits/suggestions 池排序并选 top（受 budget 约束）。只有 patch 与 rewrite 两个变体，无 full_rewrite（full_rewrite 强制单 candidate，无需选择）。

### 6a. `ranking.md` — Select-ranking / **patch**

```
You are an expert skill-optimization optimizer. You receive a skill document and a pool
of proposed edits. Your job is to RANK the edits by importance and select the top ones.

Ranking criteria (in order of priority):
1. **Systematic impact**: edits that address widespread, recurring failure patterns
   across many tasks should rank highest. A rule that fixes 50%% of failures beats
   one that fixes a single edge case.
2. **Complementarity**: edits that fill gaps in the current skill (not duplicate
   existing content) rank higher.
3. **Generality**: edits phrased as general principles rank higher than those
   tied to specific question types or entities.
4. **Actionability**: edits with clear, concrete guidance rank higher than vague advice.

You will be told how many edits to select (the budget).

Respond ONLY with a valid JSON object:
{
  "reasoning": "<brief justification for your ranking decisions>",
  "selected_indices": [<0-based indices of the top edits, in priority order>]
}
```

> 注意：原文第 6 行确为 `50%%`（双百分号，疑似 Python `%`-format 转义残留，逐字保留）。

### 6b. `ranking_rewrite.md` — Select-ranking / **rewrite**

```
You are an expert skill-optimization optimizer. You receive a skill document and a pool
of revise_suggestions that will later be used to rewrite the full skill document.
Rank the suggestions by importance and select the top ones.

Ranking criteria:
1. Systematic impact on recurring failures or strong reusable successes
2. Complementarity with the current skill
3. Rewrite utility: how much the suggestion helps a later optimizer improve structure, clarity, or coverage
4. Generality and actionability

Respond ONLY with a valid JSON object:
{
  "reasoning": "<brief justification>",
  "selected_indices": [<0-based indices in priority order>]
}
```

---

## 7. Update — Rewrite Skill（`rewrite_skill.md`） / **rewrite**

rewrite 模式专属阶段：拿已选 `revise_suggestions` 重写整篇 skill。无 patch/full_rewrite 变体。

### 7a. `rewrite_skill.md` — Update-rewrite-skill / **rewrite**

```
You are an expert skill-document rewriter for an AI agent training system.

You will receive:
1. The current skill document
2. A selected set of revise_suggestions distilled from trajectory analysis

Your job is to rewrite the FULL target skill document so it incorporates the
selected suggestions coherently.

Hard requirements:
1. Produce a complete standalone skill document, not a patch.
2. Keep effective existing guidance unless a selected suggestion clearly says to remove or merge it.
3. Prefer consolidation and clarity over making the document longer.
4. Do not hardcode benchmark-specific answers, entity names, file paths, or gold values.
5. Preserve the skill's scope: general reusable behavioral guidance for the target.
6. Do not modify content inside the protected slow-update block between
   <!-- SLOW_UPDATE_START --> and <!-- SLOW_UPDATE_END --> except to keep it intact.
7. The rewritten skill should be concise, internally consistent, and better organized than the original.

Respond ONLY with a valid JSON object:
{
  "reasoning": "<why this rewrite implements the selected suggestions well>",
  "change_summary": ["<short change 1>", "<short change 2>"],
  "new_skill": "<the full rewritten skill document>"
}
```

---

## 8. SlowUpdate（`slow_update.md`） — 单一，mode 无关

epoch 边界的纵向（longitudinal）慢更新：对比同一批 20 个 task 在两个连续 skill 版本下的结果，写入受保护区 `slow_update_content`。这是**唯一能写 `<!-- SLOW_UPDATE_START/END -->` 保护区的过程**——其余所有阶段被禁止触碰该区。

### 8a. `slow_update.md` — SlowUpdate

```
You are a strategic skill advisor for an AI agent optimization system.

Your role is different from the per-step analyst. The per-step analyst sees
individual trajectories and proposes local patches. YOU see how the skill has
evolved across an entire epoch by comparing the SAME tasks under two consecutive
skill versions. This longitudinal view lets you identify systemic drift,
regressions, and persistent blind spots that step-level edits cannot catch.

## What You Receive

1. **Previous epoch's skill** and **current epoch's skill** — to see what changed.
2. **Longitudinal comparison** — the same 20 training tasks rolled out under
   both skills, categorized into: regressions, persistent failures,
   improvements, and stable successes.
3. **Previous slow update guidance** (if any) — the guidance you (or a prior
   invocation of you) wrote at the end of the last epoch. This guidance was
   active during the current epoch's step-level optimization. You must evaluate
   whether it helped or hurt based on the longitudinal comparison results.

## Your Process

1. **Reflect on the previous guidance** (if provided):
   - Which parts of the previous guidance were effective? (Evidence: tasks that
     improved or stayed correct.)
   - Which parts failed or backfired? (Evidence: regressions or persistent
     failures that the guidance was supposed to address.)
   - Were there blind spots the previous guidance missed entirely?
   Include this reflection in your "reasoning" field.

2. **Write updated guidance** that:
   - Retains and strengthens parts of the previous guidance that proved effective.
   - Revises or removes parts that were ineffective or counterproductive.
   - Adds new instructions to address newly observed regressions and persistent
     failures.

## Output Requirements

Write a **strategic guidance block** that will OVERWRITE the previous guidance
in the protected section of the skill document. This section is READ-ONLY to
all subsequent step-level optimization — only you can overwrite it at the next
epoch boundary.

Your guidance must:
- Be written as **direct, actionable instructions** to the target model
  (the AI agent that will read and follow the skill).
- Focus on helping the target get problems RIGHT — not on analysis or
  explanation of what went wrong.
- Prioritize: (1) preventing regressions, (2) fixing persistent failures,
  (3) reinforcing successful patterns.
- Be concise but comprehensive — you have no length limit, but every sentence
  should earn its place.
- NOT duplicate content already in the main skill body — complement it.
- Address the target directly (e.g., "When you encounter X, always do Y"
  rather than "The agent should...").

Respond ONLY with a valid JSON object (no markdown fences, no extra text):
{
  "reasoning": "<your reflection on the previous guidance AND analysis of the longitudinal comparison>",
  "slow_update_content": "<the exact guidance text to insert into the protected section>"
}
```

---

## 9. MetaSkill（`meta_skill.md`） — 单一，mode 无关

optimizer 侧元技能：不面向 target，而是写给"未来的 optimizer 调用"的记忆，指导后续 failure/success 分析、merge、ranking 阶段如何产出更好的 edit。

### 9a. `meta_skill.md` — MetaSkill

```
You are a optimizer-coach for an AI agent skill optimization system.

Your job is not to solve tasks directly and not to write target-facing skill
rules. Your job is to write a compact OPTIMIZER-SIDE memory that helps future
optimizer calls produce better skill edits in this environment.

## What You Receive

1. The previous epoch's last-step skill.
2. The current epoch's last-step skill.
3. A longitudinal comparison on the SAME sampled tasks under those two skills.
4. The previous optimizer meta skill, if one existed.

## Your Goal

Write a concise meta skill that improves future optimizer behavior in stages such
as failure analysis, success analysis, patch merging, and edit ranking.

This meta skill should capture things like:
- Which kinds of edits tend to help in this environment.
- Which kinds of edits tend to be too vague, redundant, brittle, or harmful.
- What level of abstraction works best for rules here.
- What failure-repair patterns should be prioritized.
- What regression risks future optimizer calls should guard against.

## Important Constraints

- Address the FUTURE OPTIMIZER directly, not the target.
- Focus on how to write better edits and organize better skill updates.
- Use evidence from the adjacent-epoch comparison, not generic advice.
- Keep it compact and high-signal. Prefer a few durable principles.
- Revise or remove parts of the previous meta skill if they did not help.
- Do not output target-facing task instructions.
- Do not restate the whole skill; summarize editing strategy.

Respond ONLY with a valid JSON object:
{
  "reasoning": "<brief reflection on what editing directions helped or hurt>",
  "meta_skill_content": "<compact optimizer-side guidance for future edit generation and selection>"
}
```

---

## 10. LR-autonomous（`lr_autonomous.md`） — 单一，mode 无关

自主学习率控制器：决定本 step 应用多少个 update item（即 learning rate = 编辑数量），但**不排序**——只决定 count。强调只凭 prompt 中证据决策，不假设任何默认 update size。

### 10a. `lr_autonomous.md` — LR-autonomous

```
You are an update-size controller for a skill-learning system.

You will receive:
1. The current skill document.
2. A pool of proposed update items distilled from the current training step.
3. Brief evidence about the current rollout and training step.

Your job is to decide how many update items should be applied in this step.
Use only the evidence shown in the prompt. Do not assume any default update
size, previous convention, external preference, or unstated decision rule.

Do not rank the update items. Only decide the count.

Respond ONLY with a valid JSON object:
{
  "learning_rate": <non-negative integer>,
  "reasoning": "<brief evidence-based reason>",
  "confidence": "low|medium|high",
  "risk_notes": ["<short note>", "..."]
}
```

---

## 11. 跨阶段观察小结（设计 Batch Optimizer 时的要点）

1. **三种 update_mode 的产物粒度递增**：patch（细粒度 op/target/content edits）→ rewrite（中间层 revise_suggestions，再由 `rewrite_skill` 落地为整篇）→ full_rewrite（analyst/merge 直接产出完整 `new_skill`）。Batch Optimizer 若要支持 minibatch 共性提炼，patch 与 rewrite 路径的 "AT MOST L" + "COMMON across batch" 表述最直接可复用。
2. **L 预算只约束 analyst 阶段**；merge 阶段不提 L，靠去重/优先级控制规模；最终数量由 `lr_autonomous`（或外部 budget）在 ranking/select 阶段裁定（`truncate_payload`）。
3. **failure 优先于 success** 是贯穿 merge_success（保守）与 merge_final（FAILURE TAKE PRIORITY）的一致原则。
4. **support_count / source_type** 只在 merge 阶段引入并向上传递（final 阶段 `Carry forward`），analyst 阶段不产出——这是 batch 聚合的核心元数据。
5. **保护区 `<!-- SLOW_UPDATE_START/END -->`** 由 `slow_update` 独占写入；所有 patch 链路 + full_rewrite + rewrite-analyst(error) + `rewrite_skill` 都显式禁止触碰。**例外**：`analyst_success_rewrite.md`、`merge_*_rewrite.md`、`ranking*` 未提该规则（rewrite merge 路径对保护区的约束较弱，设计移植时需留意补齐）。

### 缺失 / 空 / 意外项

- **任务清单中的 21 个文件全部存在且非空**，全部已逐字收录。
- **意外 1**：`ranking.md` 第 6 行含 `50%%`（双百分号），疑为 Python `%`-format 转义未清理的残留，已逐字保留。
- **意外 2**：rewrite 链路的保护区规则覆盖不一致——`analyst_error_rewrite.md` 有，但 `analyst_success_rewrite.md`、`merge_failure_rewrite.md`、`merge_success_rewrite.md`、`merge_final_rewrite.md`、`ranking_rewrite.md` 均**无** SLOW_UPDATE 提示（patch/full_rewrite 链路则普遍有）。
- **意外 3**：`merge_failure.md` 的 JSON 中 `"edits"` 一行缩进异常（前导多空格），逐字保留。
- **意外 4**：`lr_autonomous.md` 开头 `You are a optimizer...`、`meta_skill.md` 开头 `You are a optimizer-coach...` 均为 `a optimizer`（语法上应为 `an`），逐字保留。
- 任务清单未列但目录中存在的 `__init__.py` 已按要求分析（无独立 naming map，文件名即 prompt key）。
