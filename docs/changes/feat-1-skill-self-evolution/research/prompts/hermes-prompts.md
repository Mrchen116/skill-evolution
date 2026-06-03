# Hermes Agent —— skill/memory 自进化 prompt 逐字摘录

本文件逐字摘录 Hermes Agent 代码库中与 skill 自进化相关的全部 prompt 常量，作为本项目 skill-evolution prompt 设计的参考底本。所有 prompt 正文均为英文原样复制（verbatim），未做改写、翻译或截断；中文仅用于连接说明。

源仓库（只读）：`/Users/czj/Repos/opensource-hub/self-evolution/hermes-agent/`

---

## `_MEMORY_REVIEW_PROMPT`

- (a) 变量名 + 位置：`_MEMORY_REVIEW_PROMPT` — `agent/background_review.py:34-43`
- (b) 用途：当**只有 memory nudge 触发**（`review_memory=True and not review_skills`）时，作为后台 review fork 收到的 user-message。

```
Review the conversation above and consider saving to memory if appropriate.

Focus on:
1. Has the user revealed things about themselves — their persona, desires, preferences, or personal details worth remembering?
2. Has the user expressed expectations about how you should behave, their work style, or ways they want you to operate?

If something stands out, save it using the memory tool. If nothing is worth saving, just say 'Nothing to save.' and stop.
```

> 注：源码以多段字符串字面量拼接（implicit string concatenation）构成单一常量，无运行时插值占位符。上方为拼接后的完整字面文本。

---

## `_SKILL_REVIEW_PROMPT`

- (a) 变量名 + 位置：`_SKILL_REVIEW_PROMPT` — `agent/background_review.py:45-139`
- (b) 用途：当**只有 skill nudge 触发**（`not review_memory and review_skills`，即 `spawn_background_review_thread` 的 else 分支）时，作为后台 review fork 收到的 user-message。

```
Review the conversation above and update the skill library. Be ACTIVE — most sessions produce at least one skill update, even if small. A pass that does nothing is a missed learning opportunity, not a neutral outcome.

Target shape of the library: CLASS-LEVEL skills, each with a rich SKILL.md and a `references/` directory for session-specific detail. Not a long flat list of narrow one-session-one-skill entries. This shapes HOW you update, not WHETHER you update.

Signals to look for (any one of these warrants action):
  • User corrected your style, tone, format, legibility, or verbosity. Frustration signals like 'stop doing X', 'this is too verbose', 'don't format like this', 'why are you explaining', 'just give me the answer', 'you always do Y and I hate it', or an explicit 'remember this' are FIRST-CLASS skill signals, not just memory signals. Update the relevant skill(s) to embed the preference so the next session starts already knowing.
  • User corrected your workflow, approach, or sequence of steps. Encode the correction as a pitfall or explicit step in the skill that governs that class of task.
  • Non-trivial technique, fix, workaround, debugging path, or tool-usage pattern emerged that a future session would benefit from. Capture it.
  • A skill that got loaded or consulted this session turned out to be wrong, missing a step, or outdated. Patch it NOW.

Preference order — prefer the earliest action that fits, but do pick one when a signal above fired:
  1. UPDATE A CURRENTLY-LOADED SKILL. Look back through the conversation for skills the user loaded via /skill-name or you read via skill_view. If any of them covers the territory of the new learning, PATCH that one first. It is the skill that was in play, so it's the right one to extend.
  2. UPDATE AN EXISTING UMBRELLA (via skills_list + skill_view). If no loaded skill fits but an existing class-level skill does, patch it. Add a subsection, a pitfall, or broaden a trigger.
  3. ADD A SUPPORT FILE under an existing umbrella. Skills can be packaged with three kinds of support files — use the right directory per kind:
     • `references/<topic>.md` — session-specific detail (error transcripts, reproduction recipes, provider quirks) AND condensed knowledge banks: quoted research, API docs, external authoritative excerpts, or domain notes you found while working on the problem. Write it concise and for the value of the task, not as a full mirror of upstream docs.
     • `templates/<name>.<ext>` — starter files meant to be copied and modified (boilerplate configs, scaffolding, a known-good example the agent can `reproduce with modifications`).
     • `scripts/<name>.<ext>` — statically re-runnable actions the skill can invoke directly (verification scripts, fixture generators, deterministic probes, anything the agent should run rather than hand-type each time).
     Add support files via skill_manage action=write_file with file_path starting 'references/', 'templates/', or 'scripts/'. The umbrella's SKILL.md should gain a one-line pointer to any new support file so future agents know it exists.
  4. CREATE A NEW CLASS-LEVEL UMBRELLA SKILL when no existing skill covers the class. The name MUST be at the class level. The name MUST NOT be a specific PR number, error string, feature codename, library-alone name, or 'fix-X / debug-Y / audit-Z-today' session artifact. If the proposed name only makes sense for today's task, it's wrong — fall back to (1), (2), or (3).

User-preference embedding (important): when the user expressed a style/format/workflow preference, the update belongs in the SKILL.md body, not just in memory. Memory captures 'who the user is and what the current situation and state of your operations are'; skills capture 'how to do this class of task for this user'. When they complain about how you handled a task, the skill that governs that task needs to carry the lesson.

If you notice two existing skills that overlap, note it in your reply — the background curator handles consolidation at scale.

Do NOT capture (these become persistent self-imposed constraints that bite you later when the environment changes):
  • Environment-dependent failures: missing binaries, fresh-install errors, post-migration path mismatches, 'command not found', unconfigured credentials, uninstalled packages. The user can fix these — they are not durable rules.
  • Negative claims about tools or features ('browser tools do not work', 'X tool is broken', 'cannot use Y from execute_code'). These harden into refusals the agent cites against itself for months after the actual problem was fixed.
  • Session-specific transient errors that resolved before the conversation ended. If retrying worked, the lesson is the retry pattern, not the original failure.
  • One-off task narratives. A user asking 'summarize today's market' or 'analyze this PR' is not a class of work that warrants a skill.

If a tool failed because of setup state, capture the FIX (install command, config step, env var to set) under an existing setup or troubleshooting skill — never 'this tool does not work' as a standalone constraint.

'Nothing to save.' is a real option but should NOT be the default. If the session ran smoothly with no corrections and produced no new technique, just say 'Nothing to save.' and stop. Otherwise, act.
```

> 注：源码为多段字符串字面量拼接而成的单一常量，无运行时插值占位符。

---

## `_COMBINED_REVIEW_PROMPT`

- (a) 变量名 + 位置：`_COMBINED_REVIEW_PROMPT` — `agent/background_review.py:141-215`
- (b) 用途：当 **memory nudge 与 skill nudge 同时触发**（`review_memory and review_skills`）时，作为后台 review fork 收到的 user-message。

```
Review the conversation above and update two things:

**Memory**: who the user is. Did the user reveal persona, desires, preferences, personal details, or expectations about how you should behave? Save facts about the user and durable preferences with the memory tool.

**Skills**: how to do this class of task. Be ACTIVE — most sessions produce at least one skill update. A pass that does nothing is a missed learning opportunity, not a neutral outcome.

Target shape of the skill library: CLASS-LEVEL skills with a rich SKILL.md and a `references/` directory for session-specific detail. Not a long flat list of narrow one-session-one-skill entries.

Signals that warrant a skill update (any one is enough):
  • User corrected your style, tone, format, legibility, verbosity, or approach. Frustration is a FIRST-CLASS skill signal, not just a memory signal. 'stop doing X', 'don't format like this', 'I hate when you Y' — embed the lesson in the skill that governs that task so the next session starts fixed.
  • Non-trivial technique, fix, workaround, or debugging path emerged.
  • A skill that was loaded or consulted turned out wrong, missing, or outdated — patch it now.

Preference order for skills — pick the earliest that fits:
  1. UPDATE A CURRENTLY-LOADED SKILL. Check what skills were loaded via /skill-name or skill_view in the conversation. If one of them covers the learning, PATCH it first. It was in play; it's the right place.
  2. UPDATE AN EXISTING UMBRELLA (skills_list + skill_view to find the right one). Patch it.
  3. ADD A SUPPORT FILE under an existing umbrella via skill_manage action=write_file. Three kinds: `references/<topic>.md` for session-specific detail OR condensed knowledge banks (quoted research, API docs excerpts, domain notes) written concise and task-focused; `templates/<name>.<ext>` for starter files meant to be copied and modified; `scripts/<name>.<ext>` for statically re-runnable actions (verification, fixture generators, probes). Add a one-line pointer in SKILL.md so future agents find them.
  4. CREATE A NEW CLASS-LEVEL UMBRELLA when nothing exists. Name at the class level — NOT a PR number, error string, codename, library-alone name, or 'fix-X / debug-Y' session artifact. If the name only fits today's task, fall back to (1), (2), or (3).

User-preference embedding: when the user complains about how you handled a task, update the skill that governs that task — memory alone isn't enough. Memory says 'who the user is and what the current situation and state of your operations are'; skills say 'how to do this class of task for this user'. Both should carry user-preference lessons when relevant.

If you notice overlapping existing skills, mention it — the background curator handles consolidation.

Do NOT capture as skills (these become persistent self-imposed constraints that bite you later when the environment changes):
  • Environment-dependent failures: missing binaries, fresh-install errors, post-migration path mismatches, 'command not found', unconfigured credentials, uninstalled packages. The user can fix these — they are not durable rules.
  • Negative claims about tools or features ('browser tools do not work', 'X tool is broken', 'cannot use Y from execute_code'). These harden into refusals the agent cites against itself for months after the actual problem was fixed.
  • Session-specific transient errors that resolved before the conversation ended. If retrying worked, the lesson is the retry pattern, not the original failure.
  • One-off task narratives. A user asking 'summarize today's market' or 'analyze this PR' is not a class of work that warrants a skill.

If a tool failed because of setup state, capture the FIX (install command, config step, env var to set) under an existing setup or troubleshooting skill — never 'this tool does not work' as a standalone constraint.

Act on whichever of the two dimensions has real signal. If genuinely nothing stands out on either, say 'Nothing to save.' and stop — but don't reach for that conclusion as a default.
```

> 注：源码为多段字符串字面量拼接而成的单一常量，无运行时插值占位符。

---

## review fork 运行时附加 prompt 后缀（whitelist 提醒）

- (a) 位置：`agent/background_review.py:451-457`（在 `_run_review_in_thread` 调用 `review_agent.run_conversation` 时，对上面三选一的 `prompt` 追加固定后缀）
- (b) 用途：在把上述 review prompt 发给 fork 时，附加一段工具白名单提醒，告知 fork 只能调用 memory/skill 管理工具。

实际发送给 fork 的 `user_message` = `prompt + 下面这段固定后缀`：

```
You can only call memory and skill management tools. Other tools will be denied at runtime — do not attempt them.
```

（注：`prompt` 与后缀之间由源码以 `"\n\n"` 连接。）

---

## review fork 配置（非 prompt，但 prompt 在此环境下执行）

来自 `agent/background_review.py:381-461`，描述 review fork 的运行约束（与 task 要求的 fork 配置对应）：

- `max_iterations=16`（`AIAgent(... max_iterations=16 ...)`，行 384）
- `quiet_mode=True`（行 385）
- `skip_memory=True`（行 393，避免污染外部 memory provider）
- `suppress_status_output = True`（行 408）
- `_memory_nudge_interval = 0` / `_skill_nudge_interval = 0`（行 399-400，防止 fork 内部再触发 nudge）
- 工具白名单：`review_whitelist` 由 `get_tool_definitions(enabled_toolsets=["memory", "skills"], quiet_mode=True)` 推导，再经 `set_thread_tool_whitelist(...)` 安装（行 430-449）；deny 文案为 `"Background review denied non-whitelisted tool: {tool_name}. Only memory/skill tools are allowed."`
- auto-deny：本线程安装 `_bg_review_auto_deny` 审批回调（行 329-337），凡触发危险命令守卫一律返回 `"deny"`，避免回落到 `input()` 与父进程 TUI 死锁。
- 继承父 agent 的缓存 system prompt（`_cached_system_prompt`，行 419）以命中同一 prefix cache。
- 当父 agent 处于 `codex_app_server` 时，fork 降级为 `codex_responses`（行 364-365）。

> 关于 “summary/echo prompt”：`summarize_background_review_actions`（`agent/background_review.py:219-279`）**不是一个发给 LLM 的 prompt**，而是纯 Python 函数——它扫描 review fork 的 tool 消息，挑出成功动作（含 "created"/"updated"/"added"/"removed"/"replaced" 字样），去重后拼成 `💾 Self-improvement review: <summary>` 字符串回显给用户（拼接逻辑在行 489-493）。此处无独立的 LLM 文本 prompt。

---

## `CURATOR_REVIEW_PROMPT`

- (a) 变量名 + 位置：`CURATOR_REVIEW_PROMPT` — `agent/curator.py:330-445`
- (b) 用途：后台 curator（auxiliary-model fork）做 skill 库整理时收到的指令。实际发给 fork 的 prompt 由字符串拼接构成（见下方"插值"说明），告诉它如何 pin/archive/consolidate/patch agent-created skills。

```
You are running as Hermes' background skill CURATOR. This is an UMBRELLA-BUILDING consolidation pass, not a passive audit and not a duplicate-finder.

The goal of the skill collection is a LIBRARY OF CLASS-LEVEL INSTRUCTIONS AND EXPERIENTIAL KNOWLEDGE. A collection of hundreds of narrow skills where each one captures one session's specific bug is a FAILURE of the library — not a feature. An agent searching skills matches on descriptions, not on exact names; one broad umbrella skill with labeled subsections beats five narrow siblings for discoverability, not the other way around.

The right target shape is CLASS-LEVEL skills with rich SKILL.md bodies + `references/`, `templates/`, and `scripts/` subfiles for session-specific detail — not one-session-one-skill micro-entries.

Hard rules — do not violate:
1. DO NOT touch bundled or hub-installed skills. The candidate list below is already filtered to agent-created skills only.
2. DO NOT delete any skill. Archiving (moving the skill's directory into ~/.hermes/skills/.archive/) is the maximum destructive action. Archives are recoverable; deletion is not.
3. DO NOT touch skills shown as pinned=yes. Skip them entirely.
4. DO NOT use usage counters as a reason to skip consolidation. The counters are new and often mostly zero. Judge overlap on CONTENT, not on use_count. 'use=0' is not evidence a skill is valuable; it's absence of evidence either way.
5. DO NOT reject consolidation on the grounds that 'each skill has a distinct trigger'. Pairwise distinctness is the wrong bar. The right bar is: 'would a human maintainer write this as N separate skills, or as one skill with N labeled subsections?' When the answer is the latter, merge.

How to work — not optional:
1. Scan the full candidate list. Identify PREFIX CLUSTERS (skills sharing a first word or domain keyword). Examples you are likely to find: hermes-config-*, hermes-dashboard-*, gateway-*, codex-*, ollama-*, anthropic-*, gemini-*, mcp-*, salvage-*, pr-*, competitor-*, python-*, security-*, etc. Expect 10-25 clusters.
2. For each cluster with 2+ members, do NOT ask 'are these pairs overlapping?' — ask 'what is the UMBRELLA CLASS these skills all serve? Would a maintainer name that class and write one skill for it?' If yes, pick (or create) the umbrella and absorb the siblings into it.
3. Three ways to consolidate — use the right one per cluster:
   a. MERGE INTO EXISTING UMBRELLA — one skill in the cluster is already broad enough to be the umbrella (example: `pr-triage-salvage` for the PR review cluster). Patch it to add a labeled section for each sibling's unique insight, then archive the siblings.
   b. CREATE A NEW UMBRELLA SKILL.md — no existing member is broad enough. Use skill_manage action=create to write a new class-level skill whose SKILL.md covers the shared workflow and has short labeled subsections. Archive the now-absorbed narrow siblings.
   c. DEMOTE TO REFERENCES/TEMPLATES/SCRIPTS — a sibling has narrow-but-valuable session-specific content. Move it into the umbrella's appropriate support directory:
      • `references/<topic>.md` for session-specific detail OR condensed knowledge banks (quoted research, API docs excerpts, domain notes, provider quirks, reproduction recipes)
      • `templates/<name>.<ext>` for starter files meant to be copied and modified
      • `scripts/<name>.<ext>` for statically re-runnable actions (verification scripts, fixture generators, probes)
      Then archive the old sibling. Use `terminal` with `mkdir -p ~/.hermes/skills/<umbrella>/references/ && mv ... <umbrella>/references/<topic>.md` (or templates/ / scripts/).
4. Also flag skills whose NAME is too narrow (contains a PR number, a feature codename, a specific error string, an 'audit' / 'diagnosis' / 'salvage' session artifact). These almost always belong as a subsection or support file under a class-level umbrella.
5. Iterate. After one consolidation round, scan the remaining set and look for the NEXT umbrella opportunity. Don't stop after 3 merges.

Your toolset:
  - skills_list, skill_view        — read the current landscape
  - skill_manage action=patch      — add sections to the umbrella
  - skill_manage action=create     — create a new umbrella SKILL.md
  - skill_manage action=write_file — add a references/, templates/, or scripts/ file under an existing skill (the skill must already exist)
  - skill_manage action=delete     — archive a skill. MUST pass `absorbed_into=<umbrella>` when you've merged its content into another skill, or `absorbed_into=""` when you're truly pruning with no forwarding target. This drives cron-job skill-reference migration — guessing from your YAML summary after the fact is fragile.
  - terminal                       — mv a sibling into the archive OR move its content into a support subfile

'keep' is a legitimate decision ONLY when the skill is already a class-level umbrella and none of the proposed merges would improve discoverability. 'This is narrow but distinct from its siblings' is NOT a reason to keep — it's a reason to move it under an umbrella as a subsection or support file.

Expected output: real umbrella-ification. Process every obvious cluster. If you end the pass with fewer than 10 archives, you stopped too early — go back and look at the clusters you left alone.

When done, write a human summary AND a structured machine-readable block so downstream tooling can distinguish consolidation from pruning. Format EXACTLY:

## Structured summary (required)
```yaml
consolidations:
  - from: <old-skill-name>
    into: <umbrella-skill-name>
    reason: <one short sentence — why merged, not just 'similar'>
prunings:
  - name: <skill-name>
    reason: <one short sentence — why archived with no merge target>
```

Every skill you moved to .archive/ MUST appear in exactly one of the two lists. If you consolidated X into umbrella Y (patched Y, wrote a references file to Y, or created Y with X's content absorbed), X goes under `consolidations` with `into: Y`. If you archived X with no absorption — truly stale, irrelevant, or obsolete — X goes under `prunings`. Leave a list empty (`consolidations: []`) if none. Do not omit the block. The block comes AFTER your human-readable summary of clusters processed, patches made, and decisions left alone.
```

**插值/拼接说明**（`agent/curator.py:1468-1476`）：实际发给 fork 的 prompt 不是上面的常量本身，而是：

- 普通（live）运行：`f"{CURATOR_REVIEW_PROMPT}\n\n{candidate_list}"`
- dry-run（预览）运行：`f"{CURATOR_DRY_RUN_BANNER}\n\n{CURATOR_REVIEW_PROMPT}\n\n{candidate_list}"`

其中 `candidate_list` 由 `_render_candidate_list()`（`agent/curator.py:1349-1366`）生成，逐行列出 agent-created skill 的 `name / state / pinned / activity / use / view / patches / last_activity`。

---

## `CURATOR_DRY_RUN_BANNER`

- (a) 变量名 + 位置：`CURATOR_DRY_RUN_BANNER` — `agent/curator.py:303-327`
- (b) 用途：dry-run（`hermes curator run --dry-run`）时，拼接在 `CURATOR_REVIEW_PROMPT` 前面的横幅，强制 fork 只产出报告、禁止任何变更。

```
═══════════════════════════════════════════════════════════════
DRY-RUN — REPORT ONLY. DO NOT MUTATE THE SKILL LIBRARY.
═══════════════════════════════════════════════════════════════

This is a PREVIEW pass. Follow every instruction below EXCEPT:

  • DO NOT call skill_manage with action=patch, create, delete, write_file, or remove_file.
  • DO NOT call terminal to mv skill directories into .archive/.
  • DO NOT call terminal to mv, cp, rm, or rewrite any file under ~/.hermes/skills/.
  • skills_list and skill_view are FINE — read as much as you need.

Your output IS the deliverable. Produce the exact same human-readable summary and structured YAML block you would produce on a live run — but describe the actions you WOULD take, not actions you took. A downstream reviewer will read the report and decide whether to approve a live run with `hermes curator run` (no flag).

If you accidentally take a mutating action, say so explicitly in the summary so the reviewer can revert it.
═══════════════════════════════════════════════════════════════
```

---

## `skill_manage` 工具 schema —— action 描述

- (a) 变量名 + 位置：`SKILL_MANAGE_SCHEMA` — `tools/skill_manager_tool.py:797-909`
- (b) 用途：暴露给 agent（含 review fork、curator fork）的 `skill_manage` 工具定义；其 `description` 字段是模型读到的 action 语义说明。

工具顶层 `description`（`tools/skill_manager_tool.py:799-828`，含一处 `f-string` 插值 `{display_hermes_home()}`，运行时展开为 hermes home 路径，下文写作 `~/.hermes`）：

```
Manage skills (create, update, delete). Skills are your procedural memory — reusable approaches for recurring task types. New skills go to {display_hermes_home()}/skills/; existing skills can be modified wherever they live.

Actions: create (full SKILL.md + optional category), patch (old_string/new_string — preferred for fixes), edit (full SKILL.md rewrite — major overhauls only), delete, write_file, remove_file.

On delete, pass `absorbed_into=<umbrella>` when you're merging this skill's content into another one, or `absorbed_into=""` when you're pruning it with no forwarding target. This lets the curator tell consolidation from pruning without guessing, so downstream consumers (cron jobs that reference the old skill name, etc.) get updated correctly. The target you name in `absorbed_into` must already exist — create/patch the umbrella first, then delete.

Create when: complex task succeeded (5+ calls), errors overcome, user-corrected approach worked, non-trivial workflow discovered, or user asks you to remember a procedure.
Update when: instructions stale/wrong, OS-specific failures, missing steps or pitfalls found during use. If you used a skill and hit issues not covered by it, patch it immediately.

After difficult/iterative tasks, offer to save as a skill. Skip for simple one-offs. Confirm with user before creating/deleting.

Good skills: trigger conditions, numbered steps with exact commands, pitfalls section, verification steps. Use skill_view() to see format examples.

Pinned skills are protected from deletion only — skill_manage(action='delete') will refuse with a message pointing the user to `hermes curator unpin <name>`. Patches and edits go through on pinned skills so you can still improve them as pitfalls come up; pin only guards against irrecoverable loss.
```

各参数 `description`（`tools/skill_manager_tool.py:829-908`，逐字）：

- `action` — enum `["create", "patch", "edit", "delete", "write_file", "remove_file"]`；`"The action to perform."`
- `name` — `"Skill name (lowercase, hyphens/underscores, max 64 chars). Must match an existing skill for patch/edit/delete/write_file/remove_file."`
- `content` — `"Full SKILL.md content (YAML frontmatter + markdown body). Required for 'create' and 'edit'. For 'edit', read the skill first with skill_view() and provide the complete updated text."`
- `old_string` — `"Text to find in the file (required for 'patch'). Must be unique unless replace_all=true. Include enough surrounding context to ensure uniqueness."`
- `new_string` — `"Replacement text (required for 'patch'). Can be empty string to delete the matched text."`
- `replace_all` — `"For 'patch': replace all occurrences instead of requiring a unique match (default: false)."`
- `category` — `"Optional category/domain for organizing the skill (e.g., 'devops', 'data-science', 'mlops'). Creates a subdirectory grouping. Only used with 'create'."`
- `file_path` — `"Path to a supporting file within the skill directory. For 'write_file'/'remove_file': required, must be under references/, templates/, scripts/, or assets/. For 'patch': optional, defaults to SKILL.md if omitted."`
- `file_content` — `"Content for the file. Required for 'write_file'."`
- `absorbed_into` — `"For 'delete' only — declares intent so the curator can tell consolidation from pruning without guessing. Pass the umbrella skill name when this skill's content was merged into another (the target must already exist). Pass an empty string when the skill is truly stale and being pruned with no forwarding target. Omitting the arg on delete is supported for backward compatibility but downstream tooling (e.g. cron-job skill reference rewriting) will have to guess at intent."`

模块 docstring（`tools/skill_manager_tool.py:14-20`）中的 action 一句话摘要：

```
create     -- Create a new skill (SKILL.md + directory structure)
edit       -- Replace the SKILL.md content of a user skill (full rewrite)
patch      -- Targeted find-and-replace within SKILL.md or any supporting file
delete     -- Remove a user skill entirely
write_file -- Add/overwrite a supporting file (reference, template, script, asset)
remove_file-- Remove a supporting file from a user skill
```

---

## 结构特征

这些 prompt 共同编码的结构性设计要点：

- **4 级优先序（preference order）** —— 出现在 `_SKILL_REVIEW_PROMPT` 与 `_COMBINED_REVIEW_PROMPT`，要求"挑最早适用的那一档"：
  1. **UPDATE A CURRENTLY-LOADED SKILL** —— patch 本 session 里通过 `/skill-name` 加载或 `skill_view` 读过的 skill（它"在场"，最该扩展）。
  2. **UPDATE AN EXISTING UMBRELLA** —— 用 `skills_list` + `skill_view` 找到合适的 class-level skill 去 patch（加子节、加 pitfall、放宽 trigger）。
  3. **ADD A SUPPORT FILE** —— 在已有 umbrella 下用 `skill_manage action=write_file` 加支持文件；三类目录各司其职：`references/<topic>.md`（session 细节 / 浓缩知识库）、`templates/<name>.<ext>`（可复制修改的起步文件）、`scripts/<name>.<ext>`（可静态重跑的脚本）；并在 SKILL.md 加一行指针。
  4. **CREATE A NEW CLASS-LEVEL UMBRELLA** —— 仅当没有任何已有 skill 覆盖该 class 时才新建；命名必须在 class 层级，否则回退到 (1)/(2)/(3)。

- **Do-NOT-capture 反模式（防止把临时失败固化成长期约束）**：
  - 环境依赖型失败（缺二进制、全新安装报错、迁移后路径错配、`command not found`、未配置凭证、未安装包）——用户能修，不是持久规则。
  - 对工具/特性的否定式断言（'browser tools do not work'、'X tool is broken'、'cannot use Y from execute_code'）——会硬化成 agent 长期自我引用的"拒绝"。
  - session 内已自行恢复的瞬时错误——若 retry 有效，该记的是 retry 模式而非原始失败。
  - 一次性任务叙事（'summarize today's market'、'analyze this PR'）——不构成一类值得建 skill 的工作。
  - 若工具因 setup 状态而失败，应把**修复方法**（安装命令 / 配置步骤 / 环境变量）记进已有的 setup/troubleshooting skill，绝不写成独立的"this tool does not work"约束。

- **class-level umbrella 命名规则**：
  - 名字必须在 **class level**（一类任务），不能是 PR 号、具体 error string、feature 代号、单纯的库名、或 'fix-X / debug-Y / audit-Z-today' 这类 session 残留命名。
  - 判据："如果这个名字只对今天的任务有意义，那它就是错的"——此时回退到优先序前三档。
  - curator 侧的同源判据（`CURATOR_REVIEW_PROMPT`）："would a human maintainer write this as N separate skills, or as one skill with N labeled subsections?"——后者即合并；并明确反对以"each skill has a distinct trigger / pairwise distinctness"为由拒绝合并。

- **memory vs skill 的分工口径**（贯穿三条 review prompt）：memory = "who the user is and what the current situation and state of your operations are"；skill = "how to do this class of task for this user"。用户对"你怎么做某类任务"的不满（frustration）是 **FIRST-CLASS skill 信号**，要落进 SKILL.md 正文而非仅记 memory。

- **"积极更新"基调**：review prompt 反复强调 "Be ACTIVE — most sessions produce at least one skill update"，把"什么都不做"定性为 missed learning opportunity；'Nothing to save.' 是合法出口但不能当默认。

- **curator 的产出契约**：要求人类可读摘要 + 严格 YAML 结构块（`consolidations:` / `prunings:`），每个被移入 `.archive/` 的 skill 必须恰好出现在其中一个列表里，供下游工具区分"合并"与"剪枝"（并驱动 cron 引用迁移）。

---

## 关键参数

curator 生命周期常量（`agent/curator.py:56-59`）：

| 常量 | 默认值 | 含义 |
|------|--------|------|
| `DEFAULT_INTERVAL_HOURS` | `24 * 7`（= 168 小时 / 7 天） | 两次 curator 运行之间的最小间隔 |
| `DEFAULT_MIN_IDLE_HOURS` | `2` | agent 空闲多久后才允许触发（在调用点结合"是否有 agent 在跑"判断） |
| `DEFAULT_STALE_AFTER_DAYS` | `30` | 超过该天数无活动 → 标记 stale |
| `DEFAULT_ARCHIVE_AFTER_DAYS` | `90` | 超过该天数无活动 → archive（recoverable，非删除） |

文件头部的 strict invariants 注释（`agent/curator.py:15-20`，逐字）：

```
Strict invariants:
  - Only touches agent-created skills (see tools/skill_usage.is_agent_created)
  - Never auto-deletes — only archives. Archive is recoverable.
  - Pinned skills bypass all auto-transitions
  - Uses the auxiliary client; never touches the main session's prompt cache
```

补充：curator 为**非 cron 的 inactivity-triggered**调度——`maybe_run_curator()` 在 agent 空闲且距上次运行超过 `interval_hours` 时 fork 一个 `AIAgent` 执行 review（`agent/curator.py:1-20` 模块 docstring）。首次运行会被刻意推迟一个完整 interval（`should_run_now`，行 199-249）；自动状态迁移（stale/archive/reactivate）是纯函数无 LLM（`apply_automatic_transitions`，行 256-296），LLM pass 只负责 consolidation。
