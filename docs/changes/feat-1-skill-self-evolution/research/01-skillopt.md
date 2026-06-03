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

→ 我们在线把这三条分别替换成：① 用 LLM 启发式给轨迹打 失败/成功 标（[§3.5](#3)）；② "下一批真实轨迹自纠 + 拒绝缓冲"替代 留出集闸门（[§4](#4)）；③ 有界改动 + 每改必快照可回滚，把试错伤害压到最小。下面 ② 起逐条说。

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
3. **分析器的输入只有 当前skill + 轨迹**，没有外部知识源——我们要往这个输入里**加 MCP 工具**（决策 C 内联），让它当场查部门知识。

### 对照：我们的 B 在这张 I/O 表上改了哪几格

| 阶段 | SkillOpt 原版 | 我们 B（单 skill·在线） |
|---|---|---|
| 打标 | 无（直接用 ① 的 对错分 hard） | **新增 LLM 标注器**：轨迹切片 → 失败/成功标 + 失败指纹 |
| ① Rollout（跑） | 拿数据集任务现跑出轨迹 | **不跑**——直接拿该 skill 的 X 段【已结束会话】真实轨迹切片当输入 |
| ② Reflect（反思） | 分析器输入 = 当前skill+轨迹 | 分析器输入 **+ MCP 工具**；开放式找缺口；可直接产 支持数 |
| ③ Aggregate（合并） | 跨小批合并 | 量小可跳过（小批=整桶时） |
| ④ Ranking（选） | 取前 L 条 | 取前 L 条 **+ 支持数≥2 门槛** |
| ⑥ Gate（验收） | 留出集打分、严格更高才收 | **删**——换"下批轨迹自纠 + 拒绝缓冲" |

## ② 我们搬什么 / 不搬什么

| 阶段 | 搬不搬 | 在线怎么处理 |
|---|---|---|
| ② Reflect（success/failure 分桶 + analyst） | ✅ 照搬 | B 层核心 |
| ③ Aggregate（失败优先层级合并） | ✅ 照搬 | B 层 |
| ④ Select（ranking top-L、edit budget） | ✅ 照搬 | + support_count≥2 门槛（接 Codex） |
| `Edit` 结构（op/target/content/support_count/source_type） | ✅ 照搬 | SkillStore.apply 的入参 |
| rejected-edit buffer | ✅ 照搬 | 自纠负反馈 |
| `<!-- SLOW_UPDATE -->` 保护区 | ✅ 照搬 | 由 Curator(C) 维护 |
| ⑥ Validation gate（留出集打分） | ❌ **不搬** | 在线无可重放留出集 → 改"下一批自纠"（见 §4） |
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

### 3.5 在线打标替代（我们新增，SkillOpt 原本靠数据集 label）

我们没有 `hard`，用启发式从 jsonl 轨迹打标：
- **failure 侧**：用户纠偏/重做/否定语、工具调用报错后重试、任务被放弃、显式负反馈。
- **success 侧**：任务顺利完成、无返工、无纠偏。
- 信号本身也存入 `fitness_signals`（失败指纹），供 §4 自纠"是否复发"判定。
- 打标启发式的权重/词表在 M3 写实并可调。

## ④ validation gate 为什么不搬 & 在线替代 <a id="4"></a>

`evaluation/gate.py`：`evaluate_gate` 在**留出验证集**上比候选 vs current/best 分数，`hard` 模式要求**严格更高**才 `accept`/`accept_new_best`，否则 `reject`。被 reject 的 edits 进 buffer，下次作为 `rejection_context` 注入 analyst（"这些试过被否，别再提"，`trainer.py:1411-1420`）。

**为什么不搬**：在线没有"同一批题可重放打分"的留出集（spec Q3 / design 约束6）。
**在线替代**：
1. 每次 apply 前 `SkillStore.snapshot`。
2. 直接生效（spec Q1 全自动）。
3. **下一批 B 跑时回看**：上次为某失败指纹打的补丁，这批该指纹是否仍复发 / 是否新增回归（用 `fitness_signals` 比对）。
4. 复发或回归 → `rollback` 到快照 + 把该 edit 写 `rejected_edits`；不复发 → 保留。
5. `rejected_edits` 作为负反馈注入下一轮 analyst（**这一段是从 SkillOpt buffer 直接搬的**）。

→ "持续真实轨迹流"当验证集，离线一次性 gate 被在线连续自纠取代。

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
