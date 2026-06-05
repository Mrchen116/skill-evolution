# research/ — 参考源挖掘与移植依据

> design.md 是"怎么做"的骨架；本目录是"凭什么这么做"的实现级依据。每个机制的具体运作、参数默认值、success/failure 等分流逻辑，都从各参考源挖出来放这里，design.md 用 `[research/xx]` 引用。

## 文档索引

| 文件 | 参考源 | 落地于 | 一句话 |
|---|---|---|---|
| [01-skillopt.md](./01-skillopt.md) | microsoft/SkillOpt | M3 (B) | 从一批轨迹提炼有界编辑的引擎：success/failure 分桶、analyst、support_count、ranking、保护区（注：gate/rejected buffer 是其离线机制，在线不搬，见 §4） |
| [02-hermes.md](./02-hermes.md) | hermes-agent | M2 (A) / M4 (C) | 运行期两层进化：per-turn review 触发 + skill_manage 工具 + Curator + provenance |
| [03-auto-dream.md](./03-auto-dream.md) | Claude Code | M3 (B) 调度 | 后台反思的门控（时间/量/锁/节流）+ 输入装配 + 整理纪律 |
| [04-claude-mem.md](./04-claude-mem.md) | thedotmack/claude-mem | M1 chassis | 在 CC 外旁挂的底盘：瘦 hook + 常驻 worker + 增量 tail + SQLite + MCP |
| [05-codex.md](./05-codex.md) | Vaibhav 自蒸馏 prompt | M3 (B) 新建判定 | 重复 workflow → 新建 skill 的门槛与 reuse-before-create |
| [06-skill-creator.md](./06-skill-creator.md) | claude 官方插件 | 冷启动（非本 unit 主体） | draft→eval→improve + description 优化 |
| [07-trace2skill.md](./07-trace2skill.md) | Qwen-Applications/Trace2Skill | M3 (B) 候选 | 与 SkillOpt 同源；skill=**目录** + conflict-free 合并 + agentic 验证驱动 error 分析；含 vs SkillOpt 对比表 |

每个文档统一结构：**①它是什么 → ②我们搬什么/不搬什么 → ③具体怎么运作（代码级）→ ④落到哪个 milestone → ⑤关键参数默认值**。

---

## 跨源核心流程一：在线进化闭环（替代 SkillOpt 离线训练循环）

SkillOpt 的离线循环是 `rollout → reflect → aggregate → select → update → gate(留出集打分) → accept/reject`。我们在线没有可重放打分的留出集——但**不重建 gate 那堵墙**。**对齐 Hermes（最成熟的在线方案，它刻意不衡量"改动有没有效"）**：质量靠**入口证据门槛 + 可逆**，不靠"测效果/复发回滚"。

```
真实使用产生轨迹(jsonl)
  → [A: 单session 即时] 或 [B: 门控触发的批量]
  → reflect(分析轨迹) → select(有界, top-L, 跨会话反复≥2 才采纳)
  → update(apply + 先快照) → 直接生效
  → 质量靠: ① 入口只在 ≥2 反复才改  ② 每改可逆(快照/archive/provenance)
            ③ 持续进化自然收敛(真实需求反复产证据,下一轮继续朝它改)
            ④ C 周期维护(archive/merge/dedupe)
  ❌ 不做: 效果打分 / 失败指纹跨批比对 / 复发回滚 / outcome-driven rejected_edits
```

**对照表**（SkillOpt 离线 → 我们在线）：

| SkillOpt 阶段 | 离线做法 | 我们在线的替代 | 出处 |
|---|---|---|---|
| rollout 打分 | 在数据集上跑、有 ground-truth `hard/soft` | 真实使用轨迹 + 启发式 success/failure 打标 | [01](./01-skillopt.md#3) |
| reflect | minibatch analyst（成功/失败分桶） | **照搬**（B 层） | [01](./01-skillopt.md#3) |
| select | ranking top-L（edit budget） | **照搬** + support_count≥2 门槛 | [01](./01-skillopt.md#3) [05](./05-codex.md) |
| gate | held-out 严格更高才接受 | **丢弃且不重建**；改**入口 ≥2 门槛 + 可逆**（Hermes 式，不做效果反馈） | [01](./01-skillopt.md#4) |
| slow update | epoch 末纵向对比、写保护区 | 由 Curator(C) 周期维护保护区 | [01](./01-skillopt.md#5) [02](./02-hermes.md#4) |

---

## 跨源核心流程二：success / failure 判定后，分别触发什么操作（你点名的问题）

这是 SkillOpt 的核心分流（代码：`gradient/reflect.py::run_minibatch_reflect`）。判定不是终点，**两类轨迹走完全不同的两条分析支路**：

```python
# reflect.py 原始判定（hard 是 0/1 的任务正确性）
failures  = [r for r in results if not r.get("hard") or float(r["hard"]) < 1e-9]   # 做错的
successes = [r for r in results if r.get("hard")]                                  # 做对的
# 各自 shuffle → 切 minibatch(M) → 各跑各的 analyst
```

| 维度 | **failure 轨迹** | **success 轨迹** |
|---|---|---|
| 用哪个 prompt | `analyst_error.md` | `analyst_success.md` |
| 分析目标 | 找**共性失败模式**（非单次边缘） | 找**共性成功模式**（可泛化的赢法） |
| 产出 edit 的**意图** | **填补 skill 空缺 / 防止重犯**（"Only patch gaps") | **强化/编码有效做法**（"prefer reinforcing existing sections over adding new") |
| edit 倾向的 op | 多 `append`/`insert_after` 新增规则、`replace` 修正错误指引 | 多 reinforce 现有 section，少开新顶级 section |
| `source_type` 标记 | `"failure"` | `"success"` |
| 合并优先级 | **高**（`aggregate.py`: "Failure-driven patches take priority over success-driven ones"） | 低 |
| 共同约束 | 都 ≤L 条、都不得动 `<!-- SLOW_UPDATE -->` 保护区、都要 generalizable 不 hardcode | 同左 |

**→ 我们在线的对应**（无 ground-truth `hard`，用启发式打标，见 [01 §3](./01-skillopt.md#3)）：

- **failure 侧信号**：同 session 内用户纠偏/重做/否定语（"不对/重来/应该用X"）、工具报错后重试、任务被放弃。
- **success 侧信号**：顺利完成、无返工、无纠偏。
- **A 层（单 session 实时）**：主要吃 failure 侧信号即时补（用户刚纠偏的，赶紧记住）；success 侧轻。
- **B 层（批量）**：两侧都分桶、各跑 analyst，failure 补丁优先合并。
- 这个分流直接服务 spec 的两条成功信号：failure 侧 → "重复纠偏被吸收"；success 侧 → 固化好用法、少返工。

---

## 跨源核心流程三：三套机制的触发与分工（避免互踩）

| | A 实时 (M2) | B 批量 (M3) | C 维护 (M4) |
|---|---|---|---|
| 源 | Hermes per-turn | auto-dream 调度 + SkillOpt 引擎 | Hermes Curator + SkillOpt slow_update |
| 触发 | PostToolUse 计数≥10 且 Stop | 门控：**某 skill 自上次 B 后被用 >X 次**（per-skill 计数，只取已结束 session） | idle≥2h 且 距上次≥7天 |
| 看的范围 | 当前单 session | **这一个 skill** 的一批已结束 session | 整个 skill 库 |
| 分析脑 | 自由 review（轻） | minibatch 分桶 analyst（重） | 合并/归档判定 + 保护区 slow_update |
| 主要动作 | 即时 patch/create | 挖共性 patch + MCP 注入（**只改不建、不做效果反馈/自纠**） | archive/merge/dedupe |
| 详见 | [02 §1-3](./02-hermes.md) | [01](./01-skillopt.md) + [03](./03-auto-dream.md) | [02 §4](./02-hermes.md) + [01 §5](./01-skillopt.md) |
