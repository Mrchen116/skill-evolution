# 06 · skill-creator — 冷启动（造种子 skill，非本 unit 主体）

源码：`~/.claude/plugins/marketplaces/claude-plugins-official/plugins/skill-creator/`。已有笔记：`Mrchen116.github.io/_articles/skill-creator-running-mechanism.html`。落地：**冷启动**（部门管理员侧一次性造种子 skill），**不是本 unit 进化主体**——本 unit 只"引用"它，v1 不重造。

## ① 它是什么

"造 skill 的 skill"：draft → 测试用例 → 跑 with/without 对照 → 人审 viewer → improve → 迭代 → description 优化 → 打包。skill 全生命周期的**第 0 步：创建 + 验证**。

## ② 我们搬什么 / 不搬什么

| 要素 | 搬不搬 | 用在哪 |
|---|---|---|
| draft 阶段的 skill 写法纪律（description "pushy" 防欠触发；progressive disclosure 三层；解释 why 而非堆 MUST；imperative） | ✅ 搬 | 进化 agent **写/改 skill 时的质量标准**（A/B prompt 共用基底） |
| 从对话/轨迹 capture intent | ✅ 搬 | B 层新建 skill 时的意图抽取 |
| "发现多次重复写同一脚本 → bundle 进 scripts/" | ✅ 搬 | B 层可沉淀 support file（对应 Hermes write_file） |
| description 优化（`scripts/improve_description.py`：60/40 train/test、按 test 分选 best_description 防过拟合） | ⚠️ 可选 | 新建 skill 后优化触发词，M3 可选增强 |
| with/without 对照 eval + 人审 viewer | ❌ 不搬（进化期） | 这是**冷启动**人在环闭环；在线进化是 silent 全自动（spec Q1/Q6），不走人审 viewer |
| 打包 `.skill` | ❌ 不搬 | 冷启动下发用 |

## ③ 对本方案的定位

- **冷启动（unit 外）**：部门用 skill-creator 造好种子 skill，下发=拷贝到每人 `~/.claude/skills/`。本 unit 的 v1 主体是"下发后的进化"。
- **进化期复用的只是它的"写 skill 质量纪律"**：进化 agent 改/建 skill 时，prompt 里要带上 description-pushy、progressive disclosure、解释 why、imperative、class-level 命名这些 skill-creator + Hermes 共有的写作标准——否则进化出来的 skill 会越改越乱、触发不准。

## ④ 对 prompt 的影响

skill-creator 的"skill 写作纪律"是 **A/B/C 三个进化 prompt 的共享底座**（怎么写出一个好 skill 文档）。它和 Hermes review prompt 的命名/反模式约束合并成一份"skill 写作规范"片段，被三个阶段 prompt 复用。
