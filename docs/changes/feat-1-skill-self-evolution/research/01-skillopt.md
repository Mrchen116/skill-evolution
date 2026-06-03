# 01 · SkillOpt — 从一批轨迹提炼有界编辑的引擎

源码：`self-evolution/SkillOpt/`（microsoft/SkillOpt, arXiv 2605.23904）。落地：**M3 (B 层)** 的提炼引擎 + SkillStore 的保护区。

## ① 它是什么

把一个 `SKILL.md` 当作"冻结 agent 的可训练参数"，用深度学习优化器的纪律去训它：epoch / minibatch / learning rate / validation gate，但**不动模型权重**，零部署期推理开销。产物是紧凑（300–2000 token）、可审计的 Markdown。

ReflACT 6 阶段流水线（`engine/trainer.py`）：
```
① Rollout   带 skill 在数据集上跑，得轨迹 + 打分(hard/soft)
② Reflect   minibatch 分析轨迹 → 产 patch（本方案核心借鉴）
③ Aggregate 跨 minibatch 层级合并 patch（失败优先）
④ Select    ranking 选 top-L（= learning rate / edit budget）
⑤ Update    把 edits apply 到 skill 文档
⑥ Evaluate  在留出集打分，accept_new_best / accept / reject
epoch 末：Slow Update（纵向对比，写保护区） + Meta Skill（给优化器自己的指导）
```

### 原版离线批量训练全貌（这是它本来怎么跑的）

把它当神经网络训练读：**数据集分 train / 选择(held-out) 两份**；外层 epoch（默认 4），每个 epoch 把 train 切成若干 batch（默认 40 题/batch），每个 batch 跑一个完整 step（6 阶段）。`best_skill.md` 是全程在 held-out 上分数最高的那版。

```
数据集（带 ground-truth 答案）
 ├── train split        （用来跑 rollout、产生轨迹证据）
 └── selection split    （held-out，专门用来给候选 skill 打分、决定接不接受）

for epoch in 1..4:                                          # 外层：跑 4 遍 train
  for batch in split(train, batch_size=40):                # 每个 batch = 一个 step
    ┌─────────────────────────── 一个 STEP（6 阶段）───────────────────────────┐
    │ S_t = 当前 skill                                                          │
    │                                                                           │
    │ ① ROLLOUT：用 S_t 带着 batch 的 40 题去环境里干活                          │
    │      → 每题得一条轨迹 + hard(0/1 对错) / soft(部分分)                       │
    │                                                                           │
    │ ② REFLECT（minibatch 反思，关键）：                                        │
    │      按 hard 分桶 ──► failures (hard<ε)        successes (hard==1)         │
    │                         │                          │                      │
    │                    切 minibatch(M=8)          切 minibatch(M=8)           │
    │                         │                          │                      │
    │                  analyst_error.md           analyst_success.md            │
    │                  「找共性失败模式             「找共性成功模式               │
    │                   →提议填空缺的 edit」         →提议强化有效做法的 edit」     │
    │                         └──────────┬───────────────┘                      │
    │                    每个 minibatch 产 1 个 patch{edits[≤L]}                  │
    │                    （+ 注入：上一步被拒的 rejected_edits 当负反馈）          │
    │                                                                           │
    │ ③ AGGREGATE：跨 minibatch 层级合并所有 patch（失败补丁优先于成功）          │
    │      → 这一步给每条 edit 打 support_count（几条轨迹支持）+ source_type      │
    │                                                                           │
    │ ④ SELECT/RANKING：把合并后的 edit 池按"系统性影响"排序，选 top-L            │
    │      L = learning rate / edit budget（默认 4，cosine 衰减）                 │
    │                                                                           │
    │ ⑤ UPDATE：把选中的 ≤L 条 edit apply 到 S_t → 候选 S_t+1                     │
    │      （不得动 <!-- SLOW_UPDATE --> 保护区）                                 │
    │                                                                           │
    │ ⑥ EVALUATE / GATE（核心阀门）：                                            │
    │      在 selection split(held-out) 上给 S_t+1 打分                           │
    │      ┌── 分数严格 > 当前  → accept（S_t ← S_t+1）；若破历史最高→记 best     │
    │      └── 分数没更高        → REJECT：撤销，S_t 不变                          │
    │                                  被拒的 edits → rejected buffer            │
    │                                  （下个 step 的 ② 注入，提醒别再犯）        │
    └───────────────────────────────────────────────────────────────────────────┘

  # —— epoch 边界（每跑完一遍 train 做一次"慢更新"）——
  SLOW UPDATE：拿同样 20 道题，用"上个 epoch 的 skill"和"这个 epoch 的 skill"各重跑一遍，
               对比 回归/持续失败/改善/稳定成功 四类 → 提炼长期避坑铁律，
               覆写进 skill 的 <!-- SLOW_UPDATE --> 保护区（step 级 analyst 只读不可改）
  META SKILL：更新"给优化器自己看"的手册（哪种改法有效/太抽象/有害）

产物：best_skill.md（全程 held-out 分数最高那版，300–2000 token）
```

**三个"它能这么做、是因为离线"的前提**（也正是我们在线没有、必须替换的）：
1. **有 ground-truth**：题目自带标准答案，才能算 hard/soft、才能分 success/failure。
2. **能重放打分**：同一批题可以反复跑，所以 ⑥ 能在 held-out 上"严格更高才接受"、slow update 能"同 20 题双版本重跑对比"。
3. **离线无用户**：随便试错、拒了重来，没有真实用户被波及。

→ 我们在线把这三条分别替换成：① 启发式 success/failure 打标（[§3.5](#3)）；② "下一批真实轨迹自纠 + rejected buffer"替代 held-out gate（[§4](#4)）；③ 有界编辑 + 每改必快照可回滚，把试错伤害压到最小。下面 ② 起逐条说。

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

跨 minibatch 把多个 patch 层级合并成一个：调 optimizer LLM（`merge_failure.md`/`merge_success.md`/`merge_final.md`），**失败补丁优先于成功补丁**。合并失败则 fallback 为简单拼接，并给每条标 `merge_level`。

### 3.4 select / clip（`optimizer/clip.py::rank_and_select` + `prompts/ranking.md`）

把合并后的 edit 池交给 ranking LLM，按优先级排序选 top-L：
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

| 参数 | 默认 | 我们的取值 |
|---|---|---|
| `gradient.minibatch_size` (M) | 8 | B 层沿用 8（轨迹少时退化为 1 组） |
| `optimizer.learning_rate` (L, edit budget) | 4 | B=4，A=1–2 |
| `optimizer.min_learning_rate` | 2 | 同 |
| `optimizer.lr_scheduler` | cosine | v1 可先 constant=4，简化 |
| `optimizer.skill_update_mode` | patch | 用 patch（局部），不用 full_rewrite |
| `gradient.analyst_workers` | 16 | 单用户可降到 2–4 |
| `gradient.max_analyst_rounds` | 3 | 同 |
| `optimizer.use_slow_update` | true | 由 C 层接管 |
| `optimizer.slow_update_samples` | 20 | 在线无重跑，改用历史日志 |
| gate_metric | hard（严格更高） | **不用**（见 §4） |
