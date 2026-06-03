# 07 · Trace2Skill — 把"轨迹局部教训"蒸馏进可迁移的 skill 目录

源码：`self-evolution/Trace2Skill/`（Qwen Applications, Alibaba, arXiv 2603.25158）。和 [01·SkillOpt](./01-skillopt.md) 同源思路（从轨迹炼 skill），但 skill 是**目录**、error 分析是 **agentic 验证驱动**、合并是 **conflict-free**。逐字 prompt 见 [`prompts/trace2skill-prompts.md`](./prompts/trace2skill-prompts.md)。**本文供与 SkillOpt 对比,判断哪个更适合我们的 B。**

## 术语速查

| 词 | 中文 | 是个啥 |
|---|---|---|
| **skill 目录 (skill folder)** | 一个技能 = 一个文件夹 | `SKILL.md` + `references/*.md`（**和 Claude Code 的 skill 形态一致**） |
| **教训条目 (Memory Item)** | 一条可复用经验 | 分析阶段的产出单元，格式 Title/Description/Content；分 失败教训/成功教训，每类 ≤3 条 |
| **失败原因条目 (Failure Cause Item)** | 这次为什么错 | 因果性根因（非症状），且"修好它就够让输出对" |
| **精简解法路径 (Lean Solution Path)** | 成功轨迹的最短赢法 | 剥掉死路/返工，只留制胜步骤 |
| **补丁 (Patch)** | 对 skill 目录的一组改动 | 由若干文件级 改动(PatchEdit) 组成 |
| **改动 (PatchEdit)** | 一条文件级操作 | `{file, op, target_section/target_text, content}`，op 见下 |
| **map-reduce** | 映射-归约 | 各批记录并行提补丁(map) → 分层合并成一个(reduce) |
| **conflict-free** | 无冲突 | 合出的多条改动**行级互不重叠**，可并行 apply |
| **deepen / create** | 深化 / 从零建 | 在既有 skill 上改进 / 从弱草稿建新 skill（两种模式） |
| **冻结快照 (frozen snapshot)** | 原 skill 的只读副本 | map 阶段所有批都对着同一份原 skill 提补丁，互不干扰 |

`op`（文件级编辑操作，比 SkillOpt 单文件丰富）：`insert_after / insert_before / append_to_section / replace_in_section / add_section / delete_section / create(新建 references/*.md) / delete_file`。

## ① 它是什么 + 全景流程图

一句话：**对一个 skill 目录做 map-reduce 式进化**——并行分析一池轨迹、各自提"轨迹局部"补丁、再分层无冲突合并、翻译成具体内容、应用到目录、校验。

它把流程**切成两大相分离的阶段**（比 SkillOpt 把 analyst 和 edit 揉一起更解耦）：

```
═══════════════ 阶段 0：跑 + 评测 + 分类（产生轨迹和成败标签）═══════════════
  run_spreadsheetbench（带 skill 跑 SpreadsheetBench）
     → 每题: 日志 log + 产物 output.xlsx
  evaluate_with_official（对 gold 打分）→ 每题 PASS / FAIL
  analyze_results（把结果和日志对上）→ 失败题集 / 成功题集

═══════════════ 阶段 I：分析轨迹 → 产"教训条目"（每题一份）═══════════════
  失败题 ─► run_error_analysis（★agentic 验证驱动★）
     输入: 该题 日志 + input/output.xlsx + 脚本 + gold.xlsx + evaluate_output 工具
     干啥: 一个 ReAct agent 反复 [复现→定位→写 output_fixed.xlsx→重测 vs gold]
           直到修对，才"实证"根因（能验证就不许猜）
     输出: analysis_report.md  →(解析)→ 失败原因条目 + ≤3 条失败教训(Memory Items)

  成功题 ─► run_success_analysis_llm（单次 LLM）
     输入: 该题成功 chat log
     输出: 精简解法路径 + ≤3 条成功教训

  → parse_*  汇成 parsed_error_records.json / parsed_success_records.json
            （一池"每题的教训条目"）

═══════════════ 阶段 II：教训条目 → skill 目录（map-reduce-apply）═══════════════
  read_skill_state：把原 skill 目录读成【冻结快照】（map 全程对着它，不互相覆盖）

  ① MAP（并行提补丁）run_map_phase
     输入: 教训条目按 batch_size 切批（默认 1 题/批）+ 冻结快照
     每批一次 LLM: 针对这批教训，对【冻结快照】提一个 补丁(Patch=若干 PatchEdit)
     输出: N 个补丁（N=批数）

  ② REDUCE（无冲突分层合并）run_reduce_phase
     输入: N 个补丁 + 冻结快照
     每次 LLM 合一组(merge_system_prompt): 去重/解冲突/保留独特/
           ★行级独立(并行 apply 不许碰同一行)★ / ★create 与其 SKILL.md 链接原子配对★
     分层合（>1 就继续合）→ 1 个最终合并补丁

  ③ TRANSLATE（指令→具体内容）run_translation_phase
     输入: 最终补丁的每条"指令式改动" + 对应文件【当前内容】
     每条一次 LLM: 把"在某节追加一段讲 X"翻译成确切要写入的 markdown
     输出: 带具体 content 的改动

  ④ APPLY（确定性写盘）run_apply_phase_programmatic
     按 op 改文件：插入/替换/新建 references/*.md/删文件；
     _enforce_create_pairing 保证 create↔链接 成对；_sanitize 清理；行级独立保证可并行

  ⑤ VERIFY（LLM 复查）run_verification_phase
     检查应用后的 skill 目录是否自洽、有没有写坏

  ⑥ final consolidation checklist（全目录复查）
     去重去矛盾 / SKILL.md 不超行数上限 / 所有引用链接指向真实文件 /
     合并重叠告警 / 统一祈使句 / progressive disclosure（要点留 SKILL.md、细节进 references）

═══════════════ 验证（仍是离线）═══════════════
  训练集验证 + 跑 3 个随机种子 → 选训练集表现最好的那版 → 才上 held-out 评测
```

> **combined 模式**（`run_parallel_combined_skill_evolution`）= 同时吃失败教训 + 成功教训一起进化；**error-driven**（`run_parallel_skill_evolution`）= 只吃失败教训。

## ② 每阶段输入 / 输出（精确）

### 阶段 I · 分析（产教训条目）

| 子阶段 | 输入 | 怎么干 | 输出 |
|---|---|---|---|
| **error 分析（agentic，★最大亮点★）** | 失败题的 日志 + input/output.xlsx + 脚本 + **gold.xlsx** + `evaluate_output` 工具 | **ReAct agent**：复现失败→定位→**实现最小修复、写 `output_fixed.xlsx`→重测 vs gold**，修对了才确认根因（"能验证就不许猜"） | 失败原因条目 + **≤3 条失败教训**（泛化、**严禁提及 gold/标准答案**，只站 agent 视角写） |
| **success 分析（单次 LLM）** | 成功题的 chat log | 蒸馏 **精简解法路径**（剥掉死路/返工）+ 提炼 ≤3 条成功教训 | 精简解法路径 + 成功教训 |

> error 分析为何强：它不是读轨迹"猜"根因，而是**真去环境里复现+打补丁+重测标准答案**来实证。代价：**强依赖"可复现环境 + gold + 自动评测工具"**——纯离线。

### 阶段 II · 进化（教训条目 → skill 目录，map-reduce-apply）

| 子阶段 | 输入 | 输出 | 代码 |
|---|---|---|---|
| **read_skill_state** | 原 skill 目录 | 冻结快照（文件名→内容） | `parallel_evolving_agent.py:1271` |
| **① MAP** | 一批教训条目 + 冻结快照 | 1 个 补丁(Patch)（指令式改动） | `run_map_phase` |
| **② REDUCE** | N 个补丁 + 冻结快照 | 1 个最终合并补丁（无冲突、create/链接配对） | `run_reduce_phase` + `merge_system_prompt.txt` |
| **③ TRANSLATE** | 每条指令式改动 + 该文件当前内容 | 带具体 content 的改动 | `run_translation_phase` + `translation_system_prompt.txt` |
| **④ APPLY** | 翻译后的改动 + skill 目录 | 改完的 skill 目录（程序化写盘） | `run_apply_phase_programmatic` |
| **⑤ VERIFY** | 改完的目录 | 是否自洽（LLM 复查） | `run_verification_phase` + `verification_system_prompt.txt` |
| **⑥ consolidation** | 整个目录 | 复查清单过一遍 | `final_consolidation_checklist.txt` |

### map-reduce 放大（核心，对照 SkillOpt 的 reflect+aggregate）

```
教训条目池（每题 ≤3 条失败教训）
   │  按 batch_size(默认1) 切批
   ▼
[批1]→MAP─►补丁1   [批2]→MAP─►补丁2   …   [批N]→MAP─►补丁N      （并行；都对着同一份冻结快照）
   └──────────────┬──────────────────────────────┘
                  ▼  REDUCE：每次合一组，分层合到剩 1 个（merge_system_prompt）
              最终合并补丁（无冲突：行级独立 + create/链接配对）
                  ▼  TRANSLATE：把每条"指令"翻成确切 markdown 内容
                  ▼  APPLY：程序化写进 skill 目录各文件
                  ▼  VERIFY + consolidation checklist
              进化后的 skill 目录
```

**和 SkillOpt 的对应**：MAP≈各 minibatch 跑 analyst（但这里"一批"是教训条目、不是原始轨迹）；REDUCE≈aggregate（但更强调 conflict-free + 目录完整性）；TRANSLATE/APPLY≈update（但 SkillOpt 是单文件、这里是多文件程序化写盘）；没有 SkillOpt 的 support_count/改动预算 那套显式截断，靠合并去重 + 行数上限控规模。

## ③ 关键差异 vs SkillOpt（对比表）

| 维度 | SkillOpt [01] | Trace2Skill [07] | 对我们 B 的意义 |
|---|---|---|---|
| **skill 形态** | 单 `SKILL.md` | **目录**（SKILL.md + references/） | **CC 的 skill 就是目录** → Trace2Skill 更贴 |
| **分析↔进化** | 揉一起（analyst 直接产改动） | **两段解耦**：分析产"教训条目"→ 进化再消化 | 解耦更清晰，但多一段、多一遍 LLM |
| **error 分析** | 单次 LLM 读轨迹找共性 | **agentic：复现+打补丁+重测 gold 验证根因** | 更准，但**需可复现环境+gold→纯离线**，在线搬不动 |
| **success 分析** | 单次 LLM | 单次 LLM（精简解法路径） | 类似 |
| **合并** | 层级合并（失败优先） | **map-reduce + conflict-free(行级独立) + 目录完整性(create/链接配对、断链检查)** | 多文件 skill **必须** conflict-free → Trace2Skill 强 |
| **编辑操作** | append/insert_after/replace/delete（单文件） | + add_section/replace_in_section/**create/delete_file**（跨文件、按 section） | 目录级操作更全 |
| **控规模/防漂移** | 改动预算 L + 保护区 | conciseness + **行数上限 + 进阶披露 + final checklist** | 两套纪律可互补 |
| **共性度量** | 显式 `support_count`（merge 时打） | 无显式计数，靠并行多分析师+去重体现 | **SkillOpt 的 support_count 更便于做"≥2 门槛"** |
| **验证** | held-out 闸门（每 step 严格更高才收） | 训练集验证 + 3 seed 选最优（run 级） | **都离线**；在线都搬不动 |
| **创建 vs 改进** | 只改进单文件 | **deepen（改）+ create-from-scratch（建）两模式** | create 模式 ≈ 我们 F1/F2 |

## ④ 对我们 B 的取舍（供你拍）

**Trace2Skill 比 SkillOpt 更值得搬的**：
1. **skill 当目录 + conflict-free 合并 + create/链接原子配对 + 行数上限/进阶披露 + final consolidation checklist** —— 直接命中我们现实：CC skill 是文件夹，A/B/C 改它、Curator 维护它都需要"行级独立可并行""链接不断""细节进 references"。**SkillOpt 只管单文件，这块空白。建议这套搬给 SkillStore.apply + Curator。**
2. **分析↔进化两段解耦** + **教训条目(≤3、泛化)** 作中间产物 —— 比 SkillOpt"analyst 直接吐改动"更可控、可审计。

**Trace2Skill 搬不动的（和 SkillOpt 的 gate 同性质：离线）**：
1. **agentic 验证驱动 error 分析**（复现+打补丁+重测 gold）—— 在线**没 gold、没可重放环境**，最亮设计用不了，退化回"读轨迹找共性"，又回到 SkillOpt analyst 那档。
2. **训练集验证 + seed 选优** —— 同 SkillOpt held-out，在线无；仍用"下批自纠 + 拒绝缓冲"替代。

**一句话结论**：
- "**从轨迹炼改动**"内核两者像；SkillOpt 的 **support_count + 改动预算** 对我们"≥2 门槛 + 有界编辑"更顺手。
- "**多文件 skill 目录怎么合不打架、怎么保持健康**"——**Trace2Skill 明显更强更贴 CC**，这块从它搬（喂 SkillStore/Curator）。
- "**验证**"——两者都离线，都不搬，维持在线自纠。

→ 倾向：**B 内核以 SkillOpt 为底，skill 目录的合并/健康维护从 Trace2Skill 搬**。最终哪个更适合你来定。

## ⑤ prompt 与文件索引

逐字 prompt → [`prompts/trace2skill-prompts.md`](./prompts/trace2skill-prompts.md)。关键源码：

| 内容 | 文件 |
|---|---|
| agentic error 分析 prompt | `analysis/error_analysis_system.txt` |
| 单次 success 分析 prompt | `analysis/success_analysis_system_llm.txt` |
| error 分析 agent 实现 | `analysis/error_analysis_agent.py` |
| map-reduce 进化核心 | `skill_evolver/parallel_evolving_agent.py`（`run` 在 :3058） |
| 顺序版进化（含 patch 提议 system prompt、JSON schema） | `skill_evolver/skill_evolving_agent.py` |
| 无冲突合并 prompt | `skill_evolver/prompts/parallel_evolving_agent/merge_system_prompt.txt` |
| 翻译/应用/验证 prompt | `skill_evolver/prompts/parallel_evolving_agent/{translation,apply,verification}_system_prompt*.txt` |
| 全目录复查 checklist | `skill_evolver/prompts/skill_evolving_agent/final_consolidation_checklist.txt` |
| combined(成功+失败)入口 | `skill_evolver/run_parallel_combined_skill_evolution.py` |
| 已发布 skill 目录样例 | `released_skills/trace2skill-xlsx-122B-combined/` |
