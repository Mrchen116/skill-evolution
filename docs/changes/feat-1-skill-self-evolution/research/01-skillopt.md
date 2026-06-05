# 01 · SkillOpt — 从一批轨迹提炼有界编辑的引擎

源码：`self-evolution/SkillOpt/`（microsoft/SkillOpt, arXiv 2605.23904）。落地：**M3 (B 层)** 的提炼引擎 + SkillStore 的保护区。

## 符号速查表（本文出现的字母都在这）

| 符号 | 中文 | 是个啥 |
|---|---|---|
| **skill** | 技能文档 | 一份 `SKILL.md`，就是被优化的对象 |
| **当前skill (S_t)** | 第 t 步的技能文档 | 这一步开始时手上的 skill |
| **候选skill (S_t+1)** | 改完的下一版 | 应用了改动后的 skill，待验收 |
| **改动 (edit)** | 一处文本改动 | 对 skill 的一处增/删/改（append/insert/replace/delete） |
| **改动预算 (L)** | 一次最多改几处 | 相当于深度学习的"学习率"，默认 4 |
| **分组大小 (M)** | 每组放几条轨迹 | minibatch 大小，默认 8 |
| **小批/分组 (minibatch)** | 把一桶轨迹切成每组 M 条 | 一组一起分析，找"共性" |
| **一批任务 (batch)** | 一次拿多少题 | 默认 40 题/批 |
| **训练轮 (epoch)** | 把训练集整体跑一遍 | 默认 4 轮 |
| **对错分 (hard)** | 1=做对 / 0=做错 | 任务正确性（需要标准答案才算得出） |
| **部分分 (soft)** | 0~1 的分 | 部分正确程度 |
| **取前 L (top-L)** | 排序后取最重要的 L 条改动 | |
| **支持数 (support_count)** | 一条改动被几条轨迹支持 | 越多说明越是"共性问题" |
| **来源 (source_type)** | 失败轨迹 / 成功轨迹 | 这条改动是从哪个桶来的 |
| **留出集 (held-out / selection split)** | 专门打分、不参与"出主意"的题 | 用来验收候选 skill 好不好 |
| **验收闸门 (gate)** | 接受/拒绝候选的关卡 | 分数严格更高才接受 |
| **轨迹 (trajectory)** | 一次干活的完整过程 | 用户↔agent 的对话 + 工具调用 + 结果 |
| **拒绝缓冲 (rejected buffer)** | 被否过的改动清单 | 下次提醒别再提 |

## ① 它是什么

把一个 `SKILL.md` 当作"冻结 agent 的可训练参数"，用深度学习优化器的纪律去训它：epoch / minibatch / learning rate / validation gate，但**不动模型权重**，零部署期推理开销。产物是紧凑（300–2000 token）、可审计的 Markdown。

ReflACT 6 阶段流水线（`engine/trainer.py`）：
```
① Rollout（跑）    带 skill 在数据集上干活，得 轨迹 + 对错分(hard)/部分分(soft)
② Reflect（反思）  把轨迹切成 分组(minibatch) 分析 → 产出 改动包(patch)  ← 本方案核心借鉴
③ Aggregate（合并）跨各分组 把多个 改动包 合并成一个（失败的优先）
④ Select（选）     排序后取 前 L 条改动(top-L)，L = 改动预算(一次最多改几处)
⑤ Update（应用）   把选中的 改动 写进 skill 文档 → 得到 候选skill
⑥ Evaluate（验收）在 留出集 打分：接受为新最优 / 接受 / 拒绝
训练轮末(epoch 末)：慢更新(Slow Update，纵向对比、写保护区) + 元技能(Meta Skill，给优化器自己的指导)
```

### 原版离线批量训练全貌（这是它本来怎么跑的）

把它当神经网络训练读：**数据集分两份——训练集(train) / 留出集(held-out)**；外层跑 4 个 训练轮(epoch)，每轮把训练集切成若干 一批任务(batch，默认 40 题)，每批跑一个完整 step（6 阶段）。最终产物 `best_skill.md` = 全程在 留出集 上分数最高的那一版 skill。

```
数据集（每题都带标准答案）
 ├── 训练集(train)        → 用来"跑出轨迹证据"，指导怎么改 skill
 └── 留出集(held-out)     → 专门给改完的候选 skill 打分、决定接不接受（不参与"出主意"）

for 训练轮 epoch in 1..4:                       # 外层：把训练集整体跑 4 遍
  for 一批任务 batch in 切分(训练集, 40题/批):    # 每一批 = 一个 step（走一遍下面 6 阶段）

    当前skill (S_t) = 这一步开始时手上的 skill

    ① ROLLOUT（跑）：用 当前skill 带着这批 40 题去环境里干活
        → 每题得到 一条轨迹 + 对错分(hard, 1对/0错) / 部分分(soft)

    ② REFLECT（反思，关键）：
        按 对错分(hard) 把轨迹分两桶：
            失败桶 failures (做错的)         成功桶 successes (做对的)
        各自切成 小批(minibatch，每组 M=8 条轨迹)
            失败桶 → analyst_error.md：找"共性失败"→ 提议"填空缺"的改动
            成功桶 → analyst_success.md：找"共性成功"→ 提议"强化有效做法"的改动
        每个小批产出 1 个 改动包(patch，里面 ≤L 条改动)
        （同时把上一步"被拒的改动(rejected buffer)"塞进去当反面教材）

    ③ AGGREGATE（合并）：把各小批各自的 改动包 交给 LLM 合并成一个——
        重复/重叠的改动归并成一条，顺带数出"这条被几个小批提过"(= 支持数)，失败的改动优先
        （改动多就分层合：先合失败、再合成功、最后总合；改动少就一把合完）

    ④ SELECT / RANKING（选）：合并包里那串改动可能仍 >L 条 → 按"系统性影响"排序，只取 前 L 条(top-L)，其余丢
        L = 改动预算(一次最多改几处，默认 4，会随训练衰减)；③只去重不截断，④才砍到 ≤L

    ⑤ UPDATE（应用）：把选中的 ≤L 条改动写进 当前skill → 得到 候选skill (S_t+1)
        （不许动 <!-- SLOW_UPDATE --> 保护区）

    ⑥ EVALUATE / GATE（验收闸门，核心）：
        在 留出集 上给 候选skill 打分，和"当前分/历史最高分"比：
          分数严格 > 当前  → 接受（当前skill ← 候选skill）；若破历史最高 → 记为 best
          分数没更高        → 拒绝：撤销，当前skill 不变
                              被拒的改动 → 进 拒绝缓冲(rejected buffer)
                              （下一个 step 的 ② 会把它塞进去，提醒"别再犯"）

  # —— 训练轮边界（每跑完一遍训练集做一次"慢更新"）——
  慢更新 SLOW UPDATE：拿同样 20 道题，用"上一轮的 skill"和"这一轮的 skill"各重跑一遍，
       对比【回归/持续失败/改善/稳定成功】四类 → 提炼长期"避坑铁律"，
       覆写进 skill 的 <!-- SLOW_UPDATE --> 保护区（step 级的 analyst 只能读、不能改）
  元技能 META SKILL：更新"只给优化器自己看"的手册（哪种改法有效/太抽象/有害）

产物：best_skill.md（全程 留出集 分数最高那版，300–2000 token）
```

**三个"它能这么做、是因为离线"的前提**（也正是我们在线没有、必须替换的）：
1. **有标准答案(ground-truth)**：题目自带答案，才能算 对错分(hard)/部分分(soft)、才能把轨迹分成 失败/成功 两桶。
2. **能重放打分**：同一批题可反复跑，所以 ⑥ 能在 留出集 上"分数严格更高才接受"、慢更新 能"同 20 题用新旧两版各重跑对比"。
3. **离线无用户**：随便试错、拒了重来，没有真实用户被波及。

→ 我们在线把这三条分别替换成：① 用 LLM 启发式给轨迹打 失败/成功 标（[§3.5](#3)）；② **入口证据门槛（跨会话反复 ≥2 才改）+ 可逆** 替代 留出集闸门——**对齐 Hermes，不做"测效果/复发回滚"那套**（[§4](#4)）；③ 有界改动 + 每改必快照可回滚，把试错伤害压到最小。下面 ② 起逐条说。

### 每阶段的输入 / 输出（精确到数据结构）

> 数据结构出处：`skillopt/types.py`（`RolloutResult` 轨迹结果 / `Edit` 一条改动 / `Patch` 改动包 / `GateResult` 验收结果）。

| 阶段 | 输入 (IN) | 输出 (OUT) | 代码/prompt |
|---|---|---|---|
| **① Rollout（跑）** | 当前skill(S_t, md) + 一批任务(每题含题面 + **标准答案**) + 冻结的目标模型 | `轨迹结果列表`：每条 = {轨迹(工具调用/观察), **对错分(hard, 0/1)**, 部分分(soft), 失败原因, …} | `envs/<env>/rollout.py` |
| **② Reflect（反思）** | ①的 轨迹结果列表 + 当前skill + 改动预算(L) + 分组大小(M) + [上轮 拒绝缓冲] + [元技能] | `改动包列表`（每个小批一个）：改动包 = {理由, 改动:[{增删改操作, 锚点, 内容}]}，整体标 来源(失败/成功) | `gradient/reflect.py` |
| **③ Aggregate（合并）** | ②的 改动包列表 + 当前skill | **1 个合并后的改动包** = {改动:[每条带 支持数(support_count)、来源(source_type)、合并层级]}，**失败的优先** | `gradient/aggregate.py` + `merge_*.md` |
| **④ Select/Ranking（选）** | ③的 改动池（=合并包里那串改动，**可能多于 L 条**）+ 当前skill + 改动预算(L) | 选中的 **前 L 条改动**（编号列表），其余丢弃 | `optimizer/clip.py` + `ranking.md` |
| **⑤ Update（应用）** | 当前skill + 选中的改动 | **候选skill (S_t+1)**（按增删改操作改完） | `optimizer/skill.py` |
| **⑥ Gate（验收）** | 候选skill + **留出集**(带答案) + 当前分/历史最高分 | `验收结果{接受为新最优 / 接受 / 拒绝}`；拒绝 → 改动进 **拒绝缓冲**（喂回②） | `evaluation/gate.py` |
| **轮末·慢更新** | 上一轮 skill + 这一轮 skill + **同 20 题新旧两版对比**(回归/持续失败/改善/稳定成功) + 上次指导 | 一段长期指导文本 → 覆写保护区 | `slow_update.md` |
| **轮末·元技能** | 历次改动 + 接受/拒绝结果 | 给优化器自己看的"怎么改才有效"手册 | `meta_skill.md` |

### ② REFLECT 拆开看（你最关心的一段）

```
② REFLECT（反思）—— 输入: 轨迹结果列表(带 对错分 + 轨迹) + 当前skill + 改动预算(L) + 分组大小(M) + [拒绝缓冲]
════════════════════════════════════════════════════════════════════════════════
 2a 分桶 SPLIT（纯代码,无 LLM）
    输入: 轨迹结果列表
    输出: 失败桶 (做错的, 对错分<阈值)          成功桶 (做对的, 对错分=1)

 2b 切小批 MINIBATCH（纯代码,无 LLM）
    输入: 失败桶 / 成功桶
    输出: 失败小批 = [[M条轨迹],[M条],…]        成功小批 = [[M条],…]   （M=每组8条）

 2c 分析 ANALYST（每个小批一次 LLM 调用,全部并行）
    输入(每次): 当前skill 全文
              + 这【1 个小批】的 M 条轨迹(格式化、各截断到~12k 字符)
              + 改动预算(L, 本次最多产几条改动)
              + [拒绝缓冲, 反面教材] + [元技能]
    prompt   : 失败小批 → analyst_error.md   /   成功小批 → analyst_success.md
    输出(每次): 1 个 改动包 = { 理由, 改动:[{增删改操作, 锚点, 内容}] }，标 来源(失败/成功)
════════════════════════════════════════════════════════════════════════════════
 ② 总输出: 改动包列表（= 小批个数那么多个），失败/成功 改动包各自带标
           → 交给 ③ 合并(失败优先)、给每条改动打上 支持数(support_count)
```

**注意三个关键点**（直接影响我们怎么改）：
1. **分桶靠的是 对错分(hard)**（任务对错的标准答案）——我们在线没有，所以 REFLECT **前面要插一步 LLM 标注器**，产出 失败/成功 标，替代 对错分（[步0](./prompts/b-stage-prompts.md)）。
2. **分析器每次只看 1 个小批、看不到全局**——所以"一条改动被几条轨迹支持(支持数)"要等 ③ 合并 跨小批时才算得出。我们单 skill 量小，可让 小批 = 整桶 → 分析器一次看全 → **直接产支持数、省掉 ③**。
3. **分析器的输入除了 当前skill + 轨迹，离线还偷偷喂了 gold**——`fmt_minibatch_trajectories` 把每条失败轨迹的 **Hidden Reference（标准答案）+ 目标 system/user prompt + 表格预览**一起塞给 analyst（`reflect.py:156-218`）。也就是说**它读着标准答案找共性**。我们在线**既没有 gold、也没有外部部门知识源**——所以两件事要做：① 承认在线 analyst **结构上弱于** SkillOpt（无 gold 兜底）；② 往输入里**加 MCP 工具**（决策 C 内联）补上部门知识这一路。

### 对照：我们的 B 在这张 I/O 表上改了哪几格

| 阶段 | SkillOpt 原版 | 我们 B（单 skill·在线） |
|---|---|---|
| 打标 | 无（直接用 ① 的 对错分 hard） | **新增 LLM 标注器**：轨迹切片 → 失败/成功标（**只为分桶**；不抽失败指纹，见 §4 决策 E） |
| ① Rollout（跑） | 拿数据集任务现跑出轨迹 | **不跑**——直接拿该 skill 的 X 段【已结束会话】真实轨迹切片当输入 |
| ② Reflect（反思） | 分析器输入 = 当前skill+轨迹 | 分析器输入 **+ MCP 工具**；开放式找缺口；可直接产 支持数 |
| ③ Aggregate（合并） | 跨小批合并 | 量小可跳过（小批=整桶时） |
| ④ Ranking（选） | 取前 L 条 | 取前 L 条 **+ 支持数≥2 门槛** |
| ⑥ Gate（验收） | 留出集打分、严格更高才收 | **删**——换**入口 ≥2 门槛 + 可逆**（Hermes 式，不做效果反馈/复发回滚） |

## ② 我们搬什么 / 不搬什么

| 阶段 | 搬不搬 | 在线怎么处理 |
|---|---|---|
| ② Reflect（success/failure 分桶 + analyst） | ✅ 照搬 | B 层核心 |
| ③ Aggregate（失败优先层级合并） | ✅ 照搬 | B 层 |
| ④ Select（ranking top-L、edit budget） | ✅ 照搬 | + support_count≥2 门槛（接 Codex） |
| `Edit` 结构（op/target/content/support_count/source_type） | ✅ 照搬 | SkillStore.apply 的入参 |
| rejected-edit buffer | ❌ **不搬** | 它在 SkillOpt 里靠 gate 喂；在线无 gate 即无来源，且 Hermes 式在线方案不做效果反馈 |
| `<!-- SLOW_UPDATE -->` 保护区 | ✅ 照搬 | 由 Curator(C) 维护 |
| ⑥ Validation gate（留出集打分） | ❌ **不搬** | 在线无可重放留出集 → 改**入口 ≥2 门槛 + 可逆**（Hermes 式，见 §4） |
| ① Rollout 的 ground-truth 打分 | ❌ 不搬 | 用启发式 success/failure 打标（见 §3） |
| Slow Update 的"同 20 题双版本重跑" | ⚠️ 改造 | 无法重跑；Curator 用历史 evolution_log 做纵向判断（见 §5） |

## ③ 具体怎么运作（代码级）<a id="3"></a>

### 3.1 success/failure 分桶（`gradient/reflect.py::run_minibatch_reflect`）

```python
failures  = [r for r in results if not r.get("hard") or float(r["hard"]) < 1e-9]
successes = [r for r in results if r.get("hard")]   # failure_only=True 时跳过 success
failures  = shuffle(failures);  successes = shuffle(successes)
fail_batches = split(failures, M);  succ_batches = split(successes, M)   # M=minibatch_size=8
# fail_batches → run_error_analyst_minibatch (analyst_error.md)
# succ_batches → run_success_analyst_minibatch (analyst_success.md)
# 全部 minibatch 并行（analyst_workers=16），每个 minibatch 产一个 patch
```
**判定后的不同操作见 [README 跨源流程二](./README.md)。** 关键点：分析的是**一组**轨迹的**共性**，不是单条——"identify the most prevalent, systematic failure patterns across them … not individual edge cases"。这就是"通用程序错误 vs 偶发"的区分机制。

### 3.2 analyst 产出的结构（`prompts/analyst_error.md` / `analyst_success.md`）

每个 minibatch analyst 返回 JSON：
```json
{ "batch_size": 8,
  "failure_summary": [{"failure_type":"...","count":3,"description":"..."}],
  "patch": { "reasoning":"...", "edits": [
    {"op":"append","content":"..."},
    {"op":"insert_after","target":"<精确锚点>","content":"..."},
    {"op":"replace","target":"<旧文本>","content":"<新文本>"},
    {"op":"delete","target":"<要删文本>"} ]}}
```
`Edit` 类型（`skillopt/types.py`）：`op ∈ {append, insert_after, replace, delete}`，`target`(锚点)，`content`，`support_count`(几条轨迹支持)，`source_type ∈ {failure, success}`，`merge_level`。
约束（两个 analyst prompt 都写死）：≤L 条、必须 generalizable 不 hardcode、只补空缺不重复已有、**不得 target/改/删 `<!-- SLOW_UPDATE_START..END -->` 内的内容**。

### 3.3 aggregate（`gradient/aggregate.py`）

**一句话**：把各小批各自的 改动包 交给 LLM 合并成一个——重复/重叠的改动归并成一条，顺带数出"被几个小批提过"(= 支持数)，失败的改动优先。

**"合"和"分层"分工**（关键）：
- **合并动作 = LLM**：每次喂一组 改动包，配 `merge_failure.md`/`merge_success.md`/`merge_final.md` 提示词合成一个；合不出就 fallback 成简单拼接。
- **分层调度 = 代码**（`_hierarchical_merge`，纯 Python 循环）：LLM 不决定层级、看不到全局，层号只作为上下文写进 user message。

**什么时候分层 —— 判据只有一条：待合并的 改动包数 vs 每批上限(`merge_batch_size`=8)**：
- 改动包 **≤1** → 不合，直接返回（连 LLM 都不调）。
- **2–8** → 一层：凑 1 批 → 1 次 LLM 合并。
- **>8** → 多层：每 8 个切一批、各合 1 次（并行）→ 合出 N 个中间结果 → `while len>1` 再切再合，直到剩 1 个。

**整体顺序**（`merge_patches`）：失败桶各小批先合 → 成功桶各小批再合 → 最后"失败 merged + 成功 merged"做一次 final 合（**失败优先**）。

> 改动包数 = 该桶小批数 = ⌈桶内轨迹数 ÷ 分组大小 M⌉。
> **对我们**：单 skill、X≈20 → 失败桶顶多 2–3 个小批 → 一层合完；若令 分组大小 M = 整桶 → 只 1 个改动包 → **压根不进合并、支持数让分析器直接产**（这就是 §② 说的"量小可跳过 ③"）。

### 3.4 select / clip（`optimizer/clip.py::rank_and_select` + `prompts/ranking.md`）

> **"改动池"是什么？** ③ 合并 ≠ 砍到只剩几条——它只**去重**，产出的那 1 个合并包里仍可能有**好多条互不重复的改动**（各带支持数）。"改动池" = **这个合并包里那串改动（数量可能 > 预算 L）**。
> RANKING **不再合并**，而是**从这串里按重要性排序、只留前 L 条、其余丢掉**（执行"改动预算/学习率"）。分工：③ 管"不重复"，④ 管"只留最值的 L 条"。

把这个 改动池 交给 ranking LLM，按优先级排序选 前 L 条(top-L)：
1. **systematic impact**（修复面广的 > 修单个边缘的："fixes 50% of failures beats one edge case"）
2. **complementarity**（填空缺 > 重复已有）
3. **generality**（通用原则 > 绑定具体实体）
4. **actionability**（具体可执行 > 含糊）

L = learning rate / edit budget。

> ⚠️ **更正一处常见误读**：SkillOpt 的 ranking **不带 `support_count≥2` 硬门槛**。`clip.py` 的真实逻辑是——改动数 **≤ 预算就原样返回、连 ranking LLM 都不调**；超了才让 LLM 按上面 4 准则选 top-L。support_count 只在 ③ 合并时由 LLM 估，**不参与 ④ 的截断**。我们 B 里那条 "support_count≥2 才采纳" 是从 **Codex**（[05](./05-codex.md)）搬来、是**我们加的**，不是 SkillOpt 自带。

### 3.5 在线打标替代（我们新增，SkillOpt 原本靠数据集 label）

我们没有 `hard`，用启发式从 jsonl 轨迹打标：
- **failure 侧**：用户纠偏/重做/否定语、工具调用报错后重试、任务被放弃、显式负反馈。
- **success 侧**：任务顺利完成、无返工、无纠偏。
- 打标**只为分桶**（失败/成功，给 analyst 分两路分析用）。**不再抽"失败指纹"做跨批比对**——我们不做效果反馈/复发回滚（见 §4 修正）。
- 打标启发式的权重/词表在 M3 写实并可调。

## ④ validation gate 为什么不搬 & 在线替代 <a id="4"></a>

`evaluation/gate.py`：`evaluate_gate` 在**留出验证集**上比候选 vs current/best 分数，`hard` 模式要求**严格更高**才 `accept`/`accept_new_best`，否则 `reject`。被 reject 的 edits 进 buffer，下次作为 `rejection_context` 注入 analyst（"这些试过被否，别再提"，`trainer.py:1411-1420`）。

**为什么不搬**：在线没有"同一批题可重放打分"的留出集（spec Q3 / design 约束6）。

**在线替代（对齐 Hermes——最成熟的在线方案，它刻意不做效果反馈）**：
> 一手确认：Hermes 的 usage 只记 `use/view/patch_count`（零成败信号），Curator 只按**时间**衰减，A 改完不回头看——**它从不衡量"某次改动有没有效"**。它把质量赌在 ①入口证据 + ②可逆，而不是赌在"测效果"。我们照此：

1. **入口证据门槛**：只在跨会话**反复 ≥2** 才改——质量挡在入口，不轻易改，就不容易改坏。
2. **可逆**：每改前 `SkillStore.snapshot`；archive 不硬删、可恢复；provenance 护住 seed/user 内容不被动。
3. **持续进化自然收敛**：若某次改没解决真实需求，用户后续会**再触发同类证据**，下一轮继续朝真实需求改——靠**证据持续流入收敛**，不靠"测它有没有用"。
4. **C（Curator）周期维护**兜底：archive / merge / dedupe + 时间衰减。

❌ **明确不做**（这些是把离线 gate 山寨到在线——四个参考里**没有一个在线方案**这么做，Hermes 也不做）：效果打分、**失败指纹跨批比对、复发回滚、outcome-driven `rejected_edits`**。

> ⚠️ 这更正了本文早期把 SkillOpt 的 `rejected_edits` buffer / "下批自纠"硬搬到在线的写法——那是离线 gate 的产物，在线无 gate 即无来源，且成熟在线方案本就不做。

## ⑤ slow update & meta skill（保护区与优化器自指导）<a id="5"></a>

- **保护区**（`<!-- SLOW_UPDATE_START/END -->`）：放跨轮长期"避坑铁律"，step 级 analyst 只读不可改，只有 slow update 在 epoch 末覆写（`prompts/slow_update.md`）。SkillOpt 原版靠"同 20 题双版本重跑"做纵向对比；**我们改造**：无法重跑，改由 **Curator(C)** 周期跑，用 `evolution_log` + `fitness_signals` 的历史做纵向判断，把"反复被验证有效/反复踩的坑"提炼进保护区。
- **meta skill**（`prompts/meta_skill.md`）：只给 optimizer 看的"哪种改法有效/太抽象/有害"手册。本方案 v1 可选——B 层 analyst 调用时把它当 system 附加上下文（`format_meta_skill_context`）。优先级低于其它，列为 M3 可选增强。

## ⑥ 关键参数默认值（`configs/_base_/default.yaml`）

| 参数 | 中文 | 默认 | 我们的取值 |
|---|---|---|---|
| `gradient.minibatch_size` | 分组大小 M（每组几条轨迹） | 8 | B 层沿用 8（轨迹少时退化为 1 组） |
| `optimizer.learning_rate` | 改动预算 L（一次最多改几处） | 4 | B=4，A=1–2 |
| `optimizer.min_learning_rate` | 改动预算下限 | 2 | 同 |
| `optimizer.lr_scheduler` | 预算衰减方式 | cosine | v1 可先固定=4，简化 |
| `optimizer.skill_update_mode` | 改 skill 的方式 | patch（局部） | 用 patch，不用整篇重写 |
| `gradient.analyst_workers` | 分析器并发数 | 16 | 单用户可降到 2–4 |
| `gradient.max_analyst_rounds` | 分析最多几轮 | 3 | 同 |
| `optimizer.use_slow_update` | 是否开慢更新 | true | 由 C 层接管 |
| `optimizer.slow_update_samples` | 慢更新对比用几道题 | 20 | 在线无重跑，改用历史日志 |
| `gate_metric` | 验收闸门用哪种分 | hard（对错分，严格更高才收） | **不用**（见 §4） |

## ⑦ 论文实证结论（代码看不出来、但直接影响我们取舍）<a id="7"></a>

> 以下是读 `01-skillopt.md` 的源码挖不出来、只有论文（arXiv 2605.23904v2 §4 消融 / §4 迁移 / §5 学到的 skill）才有的结论。每条后面标注**对我们 B 的含义**。

### 7.1 改动极少而精：靠的是几条改动，不是一堆

- 六个 benchmark 上，真正被接受、写进 `best_skill.md` 的改动**总共只有 1–4 条（中位数 2.5）**。
- LiveMath **+29.3 分**来自**单独一条**改动；OfficeQA **+39.0 分**也来自**单独一条**。
- 优化器每轮**提议**的改动远多于此，绝大多数被验收闸门拒掉、进了拒绝缓冲，**根本没到目标模型**。

→ **对我们**：印证了 B 用**小预算**（L=4、support_count≥2、reuse-first）是对的——在线每次跑这个 skill，期望产出也就**一两条真正值得写进去的改动**，宁缺毋滥。不要因为"跑了一批轨迹"就觉得非得改很多。

### 7.2 消融的"承重墙"：闸门 + 拒绝缓冲 + 慢更新，而不是 batch 大小

论文消融的核心结论（§4 末原话转述）：增益对 **rollout 批大小 / 反思 minibatch 大小 / 学习率衰减曲线 这些"超参"相对不敏感**；但对以下四样**高度敏感**——
1. 有界文本学习（bounded edit budget）
2. **验收闸门（validation gate）**
3. **拒绝缓冲负反馈（rejected-edit feedback）**
4. epoch 级慢更新 / 元技能（slow/meta update）

> 原文：这些正是"让 skill 编辑表现得像一个受控训练回路"的设计。

→ **对我们（重要的诚实定调）**：我们 §4 **删掉了验收闸门**（在线没留出集）。论文说闸门+拒绝缓冲是离线的**承重墙**——但这是**离线训练回路**的承重墙。**在线我们不重建这堵墙**（成熟在线方案 Hermes 本就不做效果反馈），而是改换地基：
- **质量挡在入口**：只在跨会话 ≥2 才改（少改 → 少错），而不是"先改了再测效果回滚"。
- **可逆兜底**：每改快照 + archive 不硬删 + provenance 护 seed/user。
- **慢更新/保护区** → 交给 Curator(C) 周期维护，按时间衰减 + consolidation，**不是可选项**。
- minibatch/batch 大小放心简化（量小整桶、跳过 aggregate），论文背书"对这些不敏感"。

> 这里更正了本文早期"必须把拒绝缓冲/下批自纠做实"的写法——那是离线 gate 的山寨，已在 §4 改掉。

### 7.3 闸门"严格更高才收" = 让被拒改动变成"信息"而非"隐藏状态"

- 闸门是**严格大于**（strictly greater）：打平也拒，**已部署的 skill 永不会无声漂移**。
- 这个保守判据的副作用：被拒的改动变成**可用的负反馈**（喂回 analyst），而不是悄悄堆积的脏状态。
- 每步还落一份 `edit_apply_report.json`，记录**每条改动 accept/skip**，事后**每一处改动的来源都可追溯**。

→ **对我们**：①"永不无声漂移"在线对应**每改必快照 + 可回滚**；② `edit_apply_report.json` 的"逐改动可追溯"正是我们 **provenance（来源追溯）** 设计的同款思想——每条 edit 记 {来源轨迹、support_count、knowledge_ref、时间}，落 `evolution_log`。

### 7.4 学到的 skill 是"可迁移工件"，不是"刷题 prompt"

- **跨模型**：大模型上训出的 skill 迁到 mini / nano **四条迁移全为正**（+3.0 ~ +9.4），一条甚至反超目标模型自训版。
- **跨 benchmark**（math→math，最严格）：三种模型规模**全为正**（+1.3 ~ +3.7），但幅度更小。
- 结论：skill 编码的是**可复用的过程性知识**，不是"记住的题目格式"。

→ **对我们**：直接给**部门级下发 → 个人级进化**这条路线背书——部门统一训/写一版 skill 下发，到每个人手上仍然有用（跨"使用场景"迁移），再各自在线微调。这正是 spec 里"部门下发一份、之后个人级演化"的设计前提。

### 7.5 优化器强度是"训练期杠杆"，部署期零成本

- 优化器只在**离线训练回路**里跑，**部署用 skill 时根本不调用**。
- 所以**换更强的优化器能提升下发 skill 的质量，却不抬高用 skill 时的推理成本**。

→ **对我们**：B 在**会话之间**（已结束 session）后台跑，可以**放心用强模型**当 analyst——这笔成本只发生在 B-time，成员**日常用 skill 时一分钱不多花**。这是 B 相对 A（实时、轻、得用快模型）的结构性优势。

### 7.6 SpreadsheetBench 实例：恰好就是我们文章里的 xlsx 例子

论文 §5 给的 SpreadsheetBench 演化实例（**和我们对比文章里的 xlsx 例子撞上了**，可直接引用）：

- 初始 skill：只泛泛说"用 Python 表格库、保留无关内容"。
- 接受的改动把它变成一套**"工作簿取证"策略**：**先检查真实工作簿而非预览**、跨多 sheet 定位表头与目标区、**lookup/聚合前先规范化键和单元格类型**、结构化编辑时保留格式、对"公式型"提问要**算出静态值写回**（哪怕提问里提到 INDEX/MATCH/XLOOKUP）、**填满完整目标区（含空白结果单元格）**、保存后**重开工作簿检查边界行和残留空白**。
- 这一版把 held-out 成绩从 **40.4 → 78.9**。

→ **对我们**：①"聚合前先规范化、先看真实数据区"正是我们文章 §5 那个"SUM 范围误含表头行"失败模式的同类问题，**论文实证了这类过程性修正确实涨分**；②说明 B 的产出应当是**这种过程性、可泛化的操作纪律**，而不是绑死某张表的硬编码——这就是 analyst prompt 里反复强调 generalizable 的实证依据。

## ⑧ 可参考 / 不可参考速查（出方案时直接查这张）<a id="8"></a>

> 关键不只是"能学/不能学"，而是把能学的再分成**「纪律/内容」**（与范式无关，进 prompt，Workflow 派和 Agent 派**都用**）和**「工作流编排件」**（只有 **Workflow 派**显式用；**Agent 派**里这些被 agent 自己的一个回合吸收掉）。这样出 W/A 两套方案时，A 派直接抄一类、W 派额外抄二类，融合不必再想。

### 一类 · 能学，且是「纪律/内容」（W/A 都用，主要进 prompt）

| 名字 | 是什么 | 为什么值得学 | 怎么用 · 出处 |
|---|---|---|---|
| **失败/成功意图分离** | 两类轨迹用两套不同目标的分析器：失败轨迹→**补空缺、防再错**（"Only patch gaps"）；成功轨迹→**强化已有 section**（"prefer reinforcing existing sections over adding new"），而不是开新章。 | 失败和成功该学的东西不一样，混成一套会两头不讨好；分开才能"失败补漏、成功固化"。 | analyst 指令(W) / agent 主 prompt(A)｜`prompts/analyst_error.md` / `analyst_success.md` |
| ⭐**找跨轨迹共性、不修单次边缘** | 分析器一次看一组轨迹，只提**多条轨迹都犯的、系统性的**问题（"most prevalent, systematic failure patterns across them … not individual edge cases"），不为某一次的偶发情况改 skill。 | 这是 B 的灵魂——**区分"真缺陷"和"一次性噪声"**，正是 B 存在的理由。 | **两套必写的灵魂条款**｜`analyst_error.md` |
| **有界改动、少而精** | 每次最多改 L 条（默认 4）；论文实证真正涨分常常**就 1 条**（[§7.1](#7)）。 | 防止"跑了一批轨迹就觉得非得大改"；自动改 skill 越克制越安全。 | prompt 给预算 + "宁缺毋滥"｜`clip.py` + 论文 §5 |
| **失败优先于成功** | 合并改动时，失败驱动的改动**优先级高于**成功驱动的（"FAILURE PATCHES TAKE PRIORITY"），冲突时保失败的。 | 进化的首要目的是堵住失败；成功固化是锦上添花，不该挤掉补漏。 | 取舍/合并优先级条款｜`merge_final.md` |
| **可泛化、不硬编码、不重复** | 改动必须是通用规则，不能写死具体值（路径/单元格/期望值）；只补 skill 现在缺的，不重复已有内容。 | 硬编码的改动换个任务就失效；重复内容只会让 skill 膨胀。 | 条款｜两个 analyst prompt 写死 |
| **保护区思想** | skill 里有一段 `<!-- SLOW_UPDATE -->` 保护区，放跨轮沉淀的长期铁律；快速分析器**只读不许改**，只有慢更新能覆写。 | 把"高频小改"和"低频沉淀"物理隔开，防止日常小改把长期铁律冲掉。 | prompt"别动保护区" + 保护区交 C 维护｜analyst prompt 末尾 |
| ~~rejected-edit 负反馈回灌（已剔除）~~ | SkillOpt 里被 gate 否掉的改动会回灌 analyst。 | **在线不用**：无 gate 即无"被否改动"来源；且成熟在线方案（Hermes）不做效果反馈。列此仅为记录"为何不搬"。 | 不搬｜`reflect.py` `step_buffer_context` |
| **meta_skill 思想** | 一份**只给优化器自己看**的经验（"这个环境里哪种改法有效/太虚/有害"），**绝不混进下发给用户的 skill**。 | 把"怎么改 skill 的经验"和"skill 内容本身"分开存，避免污染用户实际读的 skill。 | 可选：B 的自我经验单独存、不进 skill 正文｜`prompts/meta_skill.md` |

### 二类 · 能学，但是「工作流编排件」（只有 W 派显式用；A 派被 agent 回合吸收）

| 名字 | 是什么 | 为什么值得学（及对我们） | 落到哪 · 出处 |
|---|---|---|---|
| **阶段切分** | 把"轨迹→改 skill"拆成固定流水线：打标→分桶→minibatch→合并→排序，每步一次窄任务 LLM 调用、代码当指挥。 | 它是 **Workflow 派的骨架**；但 **A 派不需要**——agent 在自己一个回合里就把这些一起做了。 | 只 W 派显式用｜`reflect.py` + `aggregate.py` + `clip.py` |
| **层级合并** | 多个 minibatch 各自产的改动包，分层合成一个：失败组先合、成功组再合、最后失败优先做总合；待合的 ≤1 个就不合。 | 为"大批量(40题)产很多改动包"设计；**我们单 skill 量小，基本退化成"一把合"或直接跳过**。 | 量小可跳过｜`aggregate.py::_hierarchical_merge` |
| **edit 结构 + 锚点** | 每条改动 = {操作, 锚点文本, 内容}，操作有 append/insert_after/replace/delete。 | 是改动的标准数据结构，落给写入器用；但**只管单文件**，文件夹型 skill 的目录操作得补 Trace2Skill。 | 给 SkillStore.apply（单文件部分）｜`analyst_error.md` schema |
| **support_count 计数动作** | 合并时让 LLM 估"这条改动被几个改动包支持"，作为"反复程度"的量。 | 它是"反复=共性"的量化抓手；**注意：SkillOpt 自己排序截断时并不用它卡 ≥2**（见四类·似是而非），那个 ≥2 是我们接 Codex 加的。 | 我们据此做 ≥2 门槛｜`merge_failure.md` 第6条 |

### 三类 · 不能学（离线特权，在线根本没有）

| 名字 | 是什么 | 为什么不能学 | 在线替代 · 出处 |
|---|---|---|---|
| ⭐**验收 gate** | 每改一版 skill，就在专门的留出集上重跑打分，**分数严格更高才接受**，否则撤销、改动进拒绝缓冲。 | 要"同一批题可反复重放打分"的留出集，在线只有真实用户的一次性使用，**没有可重放留出集**。 | **不重建这堵墙**；改 **入口 ≥2 门槛 + 可逆**（Hermes 式，不做效果反馈/复发回滚，见 [§4](#4)）｜`gate.py`：`cand_score > current_score` |
| **rollout + 对错打分** | 拿数据集任务现跑出轨迹，因题目自带标准答案，能算出每条轨迹的对/错分(hard)、部分分(soft)，据此分成失败/成功桶。 | 在线没有标准答案，算不出对错分。 | 用 LLM 启发式打标（且比有 gold 弱）｜`gate.py` / 数据集自带答案 |
| ⭐**分析器能偷看 gold** | 失败分析时，连**标准答案(Hidden Reference)**、目标 prompt、表格预览一起喂给分析器——它是**对着答案找共性**。 | 在线分析器**没有 gold 可看**，所以它**结构上就弱于** SkillOpt 的分析器——这点要诚实承认，不能假装平替。 | 无替代，只能承认更弱 + 靠 MCP 补部门知识｜`reflect.py::fmt_minibatch_trajectories` |
| **慢更新的"双版重跑"** | epoch 末拿同样 20 道题，用上一版和这一版 skill 各重跑一遍，对比回归/改善，把长期铁律写进保护区。 | 同样要重放，在线做不到。 | 保护区改由 **C** 用历史 `evolution_log` 做纵向判断｜`prompts/slow_update.md` |

### 四类 · 似是而非（旧二手笔记的误判，已在正文更正）

1. **"ranking 带 support_count≥2 门槛"——错。** `clip.py`：≤预算原样返回、超了才按 4 准则选 top-L，**无 ≥2 硬筛**。"≥2"是 **Codex** 来的、我们*加*的。（已更正 [§3.4](#3)）
2. **"在线打标 ≈ SkillOpt 的 hard 标"——不对等。** hard 来自 gold；在线靠启发式，弱很多。（已更正 §② REFLECT 第 3 点）

### 一句话

SkillOpt 对 B 的贡献**大头在一类（纪律/内容），不在二类（编排）**——走 **Agent 派**时它降级为"prompt 素材库"，走 **Workflow 派**才额外吃它的编排。**gate 那堵承重墙端不动，是 B 的头号风险点。**
