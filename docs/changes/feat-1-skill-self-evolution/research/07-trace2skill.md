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

> ⚠️ **更正一处：进化阶段其实有两套实现，不止 map-reduce。**
> - **顺序累积版**（`skill_evolver/skill_evolving_agent.py::run_evolution`）：把分析记录按 `batch_size`（默认 1）切批，**一批一批顺序喂**；每批一次 LLM 直接**输出整文件完整内容**（modify/create/delete），skill 目录**滚动累积**（下一批看到上一批改完的版本），每批校验+回滚；末尾跑一次 **final consolidation**。调用 ≈ 批数+1，**无 support_count、无 ranking 预算、无 gate**。"反复出现"靠**同类失败在多批被反复加固 + consolidation 合并重叠**。
> - **map-reduce 版**（`parallel_evolving_agent.py`）：下面这张图画的就是它——冻结快照 + 并行 map + conflict-free reduce + translate + apply + verify。重、抗顺序偏差、可并行，**为大批量设计**。
>
> **对我们（单 skill、~20 会话、量小）：顺序累积版的形态几乎是量身的，复杂度低一个数量级**；map-reduce 版大概率用不上。下文流程图以 map-reduce 版为主（细节最全），但落地参考优先看顺序版。

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
     输入: 【按任务的分析记录(record)】按 batch_size 切批（默认 1 个任务/批）+ 冻结快照
           （一条 record = 一个失败任务的 {instance_id, items:[失败原因条目们 + ≤3 失败教训]}）
     每批一次 LLM: 针对这批 record，对【冻结快照】提一个 补丁(Patch=若干 PatchEdit)
     输出: N 个补丁（N=批数）
     ★找共性的位置：batch=1 时单次 MAP 只看一个任务 → 跨任务共性推迟到 ② REDUCE 去重才体现；
       batch>1 时一批含多任务，analyst 可直接抓"同一失败原因出现在多个 record"。
       （对比 SkillOpt：minibatch=8，analyst 一次看 8 条直接找共性）

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

═══════════════ 验证（注意：无环内性能 gate）═══════════════
  进化阶段对 skill 改动只做格式校验(QUICK_VALIDATE：frontmatter/行数)，不过才回滚——
    ★不重跑 agent 打分、不按性能 accept/reject★（这点和 SkillOpt 不同，SkillOpt 有 selection-split gate）
  真正的 gold 依赖在【分析阶段的 agentic 根因验证】(复现+修 output_fixed.xlsx+重测 gold)，不在 edit 把关
  （论文最终在 held-out 上测分、跑 3 个种子取平均，那纯属 benchmark 评测报告，
    ★与进化机制无关、不影响 skill 怎么改★——3-seed 多跑在 run_spreadsheetbench.py，不在 skill_evolver）
```

> **上面阶段 II 为简单起见只画了"只吃失败"的情形。** 其实有**两种用法**：① 只用失败（论文叫 self-create / xlsx-35B）；② 失败 + 成功一起用（self-deepen / combined，论文里最好那版）。**成功教训具体放进哪种改动、怎么和失败一起跑，见下面 ② 的 map-reduce 放大图（已画全）。** 这正是我们 spec **D 决策（success 保留但弱化、failure 优先）** 的范本。

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
分析阶段产出【两个池】：
   〔失败记录池〕每条 = 一个失败任务：失败原因 + ≤3 失败教训
   〔成功记录池〕每条 = 一个成功任务：精简赢法 + ≤3 成功教训

选哪个入口 → 决定喂哪些池：
   · error-driven 入口：只喂〔失败记录池〕         （成功池不参与）
   · combined  入口：〔失败池〕+〔成功池〕混成一池一起喂
            │
            ▼  按任务切批（默认 1 任务/批）
  [批1]→MAP→补丁1    [批2]→MAP→补丁2    …    [批N]→MAP→补丁N   （并行；都对着同一份原 skill 冻结快照）
      │   每个补丁里的改动按来源分两种：
      │     · 失败教训 ──► "补空缺 / 防再错" 的改动
      │     · 成功教训 ──► "把被验证有效的赢法 固化/强化进 skill" 的改动   ← 成功教训落在这
      └──────────────┬───────────────────────────────┘
                     ▼  REDUCE 无冲突合并（行级独立 + create与其链接成对；★失败的改动优先★）
                 1 个最终合并补丁
                     ▼  TRANSLATE 把每条改动翻成确切内容
                     ▼  APPLY 写进 skill 目录各文件（程序化）
                     ▼  VERIFY 复查 + 全目录 consolidation checklist
                 进化后的 skill 目录
```

**和 SkillOpt 的对应**：MAP≈各 minibatch 跑 analyst（但这里"一批"是教训条目、不是原始轨迹）；REDUCE≈aggregate（但更强调 conflict-free + 目录完整性）；TRANSLATE/APPLY≈update（但 SkillOpt 是单文件、这里是多文件程序化写盘）；没有 SkillOpt 的 support_count/改动预算 那套显式截断，靠合并去重 + 行数上限控规模。

### 论文实证：成功经验"用，但高方差"（本地：[`papers/trace2skill-2603.25158v1.html`](./papers/trace2skill-2603.25158v1.html)）

论文把分析师分 +Error / +Success / +Combined 三配置做消融，结论很关键：

> "**+Combined is the most consistently strong signal, +Error the most reliable, and +Success the most volatile.**"

- **+Error**：每个设置都稳定为正（最安全默认）。
- **+Success**：方差最大——最大单项 **+17.6pp**（122B Creation），但也会**掉到基线以下 −0.9pp**（122B Deepening）。
- > "Success-derived patches can be highly valuable, **but only when the hierarchical merge filters them effectively; otherwise they are less stable** than error-driven updates."
- 成功分析是**单遍非交互**（成功轨迹无需交互诊断）；成功补丁在合并时与失败补丁平等竞争，但**只有跨多条轨迹反复出现的才存活**。

**→ 直接支撑我们 spec 的 D 决策（success 保留但弱化、failure 优先）**：success 高方差、必须靠"反复出现才采纳"强过滤。我们的 **support_count≥2 门槛**正对应论文"only when the merge filters them effectively"——**对 success 侧尤其要卡死**，甚至可让 success 侧用更高门槛 / 更小预算。

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
| **验证** | **有环内 gate**：selection split 专门 accept/reject 每处 edit（严格更高才收） | **无环内性能 gate**：edit 只过格式校验+回滚；唯一 gold 依赖在「分析时 agentic 根因验证」 | 两者离线依赖位置不同；在线都搬不动（3-seed 取平均是 benchmark 评测，与进化无关） |
| **创建 vs 改进** | 只改进单文件 | **deepen（改）+ create-from-scratch（建）两模式** | create 模式 ≈ 我们 F1/F2 |

## ④ 对我们 B 的取舍（供你拍）

**Trace2Skill 比 SkillOpt 更值得搬的**：
1. **skill 当目录 + conflict-free 合并 + create/链接原子配对 + 行数上限/进阶披露 + final consolidation checklist** —— 直接命中我们现实：CC skill 是文件夹，A/B/C 改它、Curator 维护它都需要"行级独立可并行""链接不断""细节进 references"。**SkillOpt 只管单文件，这块空白。建议这套搬给 SkillStore.apply + Curator。**
2. **分析↔进化两段解耦** + **教训条目(≤3、泛化)** 作中间产物 —— 比 SkillOpt"analyst 直接吐改动"更可控、可审计。

**Trace2Skill 搬不动的（和 SkillOpt 的 gate 同性质：离线）**：
1. **agentic 验证驱动 error 分析**（复现+打补丁+重测 gold）—— 在线**没 gold、没可重放环境**，最亮设计用不了，退化回"读轨迹找共性"，又回到 SkillOpt analyst 那档。**这是 Trace2Skill 唯一的离线硬依赖**，位置在「分析时验证根因诊断」，不是 edit 把关。
2. ⚠️ **更正**：Trace2Skill **没有** SkillOpt 那种"环内性能 gate"——它对 skill 改动只做格式校验+回滚，不重跑打分。所以这侧**根本没有可搬/不可搬的性能 gate**。（论文里"3 个随机种子(41/42/43)取平均"是 **benchmark 评测报告**，多跑在 `run_spreadsheetbench.py:636 for seed in run_seeds`、聚合在 `evaluate_with_official.py`，**和自进化机制无关**——别再把它当成"进化跑 3 遍挑最好"。）

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
| **顺序累积版进化(贴我们)** | `skill_evolver/skill_evolving_agent.py::run_evolution` |
| 已发布 skill 目录样例 | `released_skills/trace2skill-xlsx-122B-combined/` |

## ⑥ 可参考 / 不可参考速查（出方案时直接查这张）<a id="6"></a>

> 同 [01-skillopt §8](./01-skillopt.md#8) 的拆法：能学的再分「纪律/内容」（W/A 都用，进 prompt）和「工作流编排件」（W 派显式用 / 落给 apply 模块）。

**两个先记住的判断点**：
1. ⚠️ **Trace2Skill 进化有两套实现**：顺序累积版（贴我们、Workflow-lite）+ map-reduce 版（重、我们大概率不用）。见 §① 更正框。
2. ⭐ **Trace2Skill 自己就是「混血」活样本**：**error 分析=agent 模式**（ReAct 自己调查），**evolution=工作流**（受控合并/应用）。直接给我们的**方案丙（开放调查用 agent、安全落盘用工作流）背书**。

### 一类 · 能学，且是「纪律/内容」（W/A 都用，进 prompt）

| 名字 | 是什么 | 为什么值得学 | 怎么用 · 出处 |
|---|---|---|---|
| ⭐**按"脆弱度"调写法松紧** | 改 skill 不是只会"加一句话"——看一个地方多容易出错来定写多死：**容易反复出错的地方写死**（精确脚本、或必须逐条走的 checklist）；**做法因情况而变的地方写松**（参数化伪代码、原则）。 | 让 skill 既不啰嗦也不僵硬，且能"越被发现错、那处就写得越死"。**SkillOpt 完全没有这一层。** | prompt 条款｜`modification_strategies_section.txt` Strategy 2 |
| **改 skill 的手法清单** | 一组明确动作：加警告要带"错的写法 vs 对的写法"对照例、把细节从 SKILL.md 拆进 references、合并重复啰嗦、删掉 LLM 本就会的常识。 | 给分析器/agent 一个现成"动作菜单"，否则它只会一招"在末尾再加一段话"。 | prompt 条款｜Strategy 1/3/4 |
| **四条安全护栏** | ① 先加不删（别动现在还对的）；② 宁可多个小改、别整篇重写；③ 每条改动要能指回一个真实失败；④ 不动 frontmatter 的 name/description。 | 自动改 skill 最大的风险是越改越飘/越乱，这四条是最低限度的刹车。 | prompt 条款｜`system_prompt_base.txt` Constraints |
| **目录卫生规则** | 因为 skill 是文件夹：多条改动不许碰同一行（可并行不打架）；新建 references 必须同时在 SKILL.md 加链接（否则成孤儿文件/断链）；SKILL.md<500 行、reference<300 行；统一祈使句；不堆 README/CHANGELOG。 | 保证自动改完后，目录不破损、链接不断、不膨胀。 | prompt 条款（A 派也照此约束 agent）｜`merge_system_prompt.txt` 第6/7条 + `apply_constraints.txt` + `final_consolidation_checklist.txt` |
| **"教训卡片"中间层（Memory Item）** | 不让分析器直接改 skill，而是先把每条失败/成功轨迹蒸馏成 **≤3 张"教训卡片"**（每张含 标题 / 一句话描述 / 正文，要泛化、不绑死单个任务），**再拿这些卡片去改 skill**。 | 多这一道中间产物，卡片能被人/系统**先检查、计数、拦截**，比"分析器一步直接吐改动"更可控、可审计、可复用。 | prompt 指令："先列 ≤3 条泛化教训，再据此改"｜`error_analysis_system.txt` / `success_analysis_system_llm.txt` |
| **教训严禁引用"答案"** | 强制分析器站在"只有任务输入、拿不到标准答案的 agent"视角写教训，不准出现"对照 gold 发现…"。 | skill 是给**没有答案**的 agent 读的；引用了答案的教训对它没用、甚至误导。（在线本就无 gold，天然满足；精神要留。） | prompt 条款｜两个 analysis prompt 的 Perspective Constraints |
| **失败+成功一起用、成功最不稳** | 论文消融：失败+成功一起用最强、只用失败最稳、只用成功方差最大（同招能 +17.6pp、也能掉到基线下）。 | 指导我们：以失败为可靠主干，成功只当加分、且只采纳"反复出现"的，宁可少用。 | success 侧更小预算/更高门槛、failure 优先合并｜论文消融（见"成功经验用，但高方差"段） |

### 二类 · 能学，但是「工作流编排件」

| 名字 | 是什么 | 为什么值得学 | 落到哪 · 出处 |
|---|---|---|---|
| ⭐**安全写入器（目录手术引擎）** | 一套把改动安全写进 skill 文件夹的程序化机器：段落/文件级多种编辑操作、确定性写盘、强制"新建文件↔加链接"成对、**TRANSLATE**（分析器写的位置常是"大概在讲 X 那段"这种近似措辞、对不上原文，先对齐成文件里真实精确文本再改）、**VERIFY**（写完格式校验并修复）、全目录复查清单。 | 任何方案都需要一个"不会把 skill 改坏"的写入器；TRANSLATE 解决的"近似位置对不上原文"是 **SkillOpt replace/insert_after 同样会犯的毛病**。 | 落给 SkillStore.apply / Curator，**W/A 都需要**（A 派 agent 用 Edit 改，但校验+整理+回滚仍放代码侧）｜`run_apply_phase_programmatic` + `translation_system_prompt.txt` + `verification_system_prompt.txt` |
| **最简骨架（顺序累积版）** | 失败记录一条条喂，每条让 LLM 重写它涉及的整个文件，skill 滚动累积（下一条看到上一条改完的版本），最后整理一遍，格式不过就回滚。 | **Workflow 派最轻的现成骨**，比 SkillOpt"分桶→minibatch→合并→排序"多阶段省得多，最贴我们单 skill、小批量。 | W 派可直接拿来当骨｜`skill_evolving_agent.py::run_evolution` |
| **重型骨架（map-reduce 版）** | 所有记录对同一份"冻结快照"并行各提补丁，再无冲突合并。 | 为大批量、抗顺序偏差设计；我们量小，**大概率不用**，仅作对照。 | 不用（仅对照）｜`parallel_evolving_agent.py` |
| **更全的编辑操作** | insert_after / insert_before / append_to_section / replace_in_section / add_section / delete_section / create / delete_file。 | 改"文件夹型" skill 需要"按小节改、能新建/删整个文件"，而 SkillOpt 只有单文件 4 个 op（append/insert_after/replace/delete），不够用。 | 给 SkillStore.apply 的操作集｜`map_output_format.txt` |

### 三类 · 不能学（离线特权）

| 名字 | 是什么 | 为什么不能学 | 在线替代 · 出处 |
|---|---|---|---|
| ⭐**靠"重测标准答案"确认根因的失败分析** | 失败分析不是读轨迹猜原因，而是一个会动手的 agent：复现失败、写个最小修复（output_fixed.xlsx）、**拿标准答案 gold 重测**，直到改对了才确认"根因就是这个"。 | 强依赖"有标准答案 + 能反复重放任务环境"；在线是真实用户在用，既没有 gold、也不能把人家的活拿去重放——**"重测确认"这个内核直接死掉**（agent 调查的"形"能留，"重测实证"的"核"留不住）。 | 退化为"读轨迹推断根因" + 靠"下一批真实使用是否复发"事后检验｜`error_analysis_system.txt` + `error_analysis_agent.py` |
| ~~训练集验证 + 3 seed 选优（误判，已删）~~ | Trace2Skill **无环内性能 gate**：edit 只过格式校验+回滚，不重跑打分。 | — | 它**唯一**离线依赖是上面那条 agentic 根因验证；"3-seed 取平均"是 benchmark 评测、**与进化无关** |

### 四类 · 似是而非（旧二手笔记的误判，已在 §① 更正）

1. **"Trace2Skill = map-reduce 重型管线"——只对一半。** 还有**顺序累积版**，且它才贴我们。
2. **"无显式 support_count 是弱点"——重定性。** 它的"反复"处理是**隐式**的（顺序多批加固+consolidation 合并 / 并行多分析师+去重），不是弱点、是另一种机制；我们可在其上**叠加** Codex 的显式 ≥2。

### 一句话

Trace2Skill 对 B 的独有贡献 = **目录手术那套（二类，落给 apply/Curator，两套都要）** + **按脆弱度调自由度（一类，进 prompt）**。而它**本身的混血结构**（agent 调查 + 工作流落盘）就是方案丙的现成范本。
