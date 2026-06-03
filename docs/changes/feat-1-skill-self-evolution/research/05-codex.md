# 05 · Codex 自蒸馏 prompt（Vaibhav）— 重复 workflow → 新建 skill 的门槛

来源：Vaibhav / @reach_vb 的 Codex self-improvement prompt（用户在 spec【原始需求】已粘全文）。落地：**M3 (B 层)** 的"新建 skill"判定 + reuse-before-create。本身就是一段 **prompt 方法论**，无代码仓。

## ① 它是什么

让 agent 回看最近 30 天 session/memory/chronicle，把**反复出现的手动 workflow** 沉淀成最小够用的资产。价值全在"怎么判断该不该资产化"。

## ② 我们搬什么 / 不搬什么

| 要素 | 搬不搬 | 怎么落 |
|---|---|---|
| **资产化硬门槛**（≥2 次 / 输入稳定 / 有明确输出停止条件 / 未被现有资产覆盖） | ✅ 搬 | B 层"新建 skill"前置判据；≥2 次 = SkillOpt `support_count≥2`（[01](./01-skillopt.md)） |
| **reuse-before-create**（优先扩展已有，别造重复） | ✅ 搬 | 新建前先查现有 skill 是否覆盖；倾向 patch 而非 create |
| **重复信号来源**：重复 workflow / 反复纠偏 / 反复失败模式 | ✅ 搬 | 对应 `workflow_observations` 表 + success/failure 打标 |
| 四分法 Skill/Subagent/Automation/Skip | ⚠️ 砍 | spec Q8 定 v1 只做 skill → 只保留 **Skill / Skip** 两路 |
| 先候选清单 → 人工批准 | ❌ 不搬 | spec Q1 定全自动；批准门去掉，保留 evolution_log 审计 |
| Chronicle/Memories 多源 | ⚠️ 简化 | v1 只有 CC jsonl 一源 |

## ③ 具体怎么运作（判定逻辑，B 层"新建 skill"分支）

```
从轨迹检测候选重复 workflow（workflow_observations.signature 计数）
  对每个候选，过门槛（全部满足才资产化）：
    - 出现 ≥2 次（或明显会重复且重复成本高）
    - 输入稳定、流程可重复、有明确输出/停止条件
    - 没被现有 skill 覆盖（先 list 现有 skill 比对）
  → 满足: 倾向"扩展已有 skill"(patch)；确无可扩展才 create 新 skill（class-level 命名，[02](./02-hermes.md) 的反模式约束）
  → 不满足: Skip（不造，避免垃圾污染库）
```

workflow"签名"（signature）怎么算：v1 用启发式——任务意图 + 工具序列骨架的归一化指纹；同签名累计计数，`count≥2 && 未覆盖` 才触发 create。M3 写实并可调。

## ④ 对 prompt 的影响（接下来 prompt 综合分析用）

Codex prompt 的"门槛 + reuse-before-create + Skip 缺省"这套**保守纪律**，要融进 B 层"新建 skill"子 prompt：宁可 Skip 也不造可疑 skill；强约束 class-level 命名、禁一次性 artifact 名（与 [02](./02-hermes.md) Hermes 的 Do-NOT-capture 反模式合流）。

---

## 附 · Codex prompt 原文中译

> 用户在 spec【原始需求】粘了两版英文原文。这里译成中文供分析。**译文不改原意，只为可读。**

### 版本 A · 官方可用版（Vaibhav 自蒸馏 prompt，"Codex 的做法"本体）

```text
让 Codex 横扫我的 sessions、Memories 和 Chronicle，识别模式，复用已有资产，只创建"最小有用"的 skill、subagent 或 automation。

回看我最近 30 天的工作（历史更短就看全部），找出值得打包的、反复出现的手动工作流。

按以下优先级使用可用证据：
- 最近的 Codex sessions 和任务摘要。
- Codex Memories 和 rollout 摘要，用来发现跨 session 反复出现的模式。
- Chronicle（若启用），用来发现 Codex 之外的重复工作。Chronicle 仅用于发现线索；重要细节尽量回到对应的源系统核实。
- 已有的 skills、custom agents、automations——以便复用或扩展现有资产，而不是造重复的。

广泛地找这类工作：反复做的、耗时的、易出错的、上下文很重的，或能从"一致流程"中获益的。涵盖编码、研究、写作、规划、沟通、运维、分析、个人事务等各类工作流。

只有当一个候选满足以下条件时才动手：
- 至少发生过两次，或明显很可能再次发生且重复成本高；
- 输入稳定、流程可重复、有明确的输出或停止条件；
- 能实质性提升速度、质量、一致性或可靠性；
- 尚未被已有资产充分覆盖。

选择最小的合适形态：
- Skill：可复用的工作流或 playbook。
- Custom subagent：边界清晰、适合委派的专家角色或调查任务。
- Automation：定时或周期性的检查、报告、提醒、监控。
- Skip：太一次性、太模糊、太敏感、或证据不足，不值得打包的。

先产出一份精简的候选清单，每条含：
- 重复的工作流
- 支撑证据与日期
- 频次/置信度
- 推荐形态：skill / subagent / automation / 扩展已有 / skip
- 为什么值得（或不值得）创建

然后只创建"高置信度、确实缺失"的项。保持窄、实用、知道来源、易于验证。不要创建投机的、重叠的、过于宽泛的资产。

最后给出：
- 你创建或扩展了什么
- 你刻意跳过了什么
- 还差什么证据才能打包
```

> 用户强调实际跑时要在末尾加一句兜底（防它乱写文件）：
```text
在我批准候选清单之前，不要创建或修改任何文件。先只给我看候选项。
```

### 版本 B · 用户定制版（更适合复杂 agent 环境，用户更推荐）

```text
回看我最近 30 天的 Codex 工作（历史更短就看全部）。

目标：
找出反复出现的手动工作流、反复的请求、我反复做的纠正、以及反复出现的失败模式——这些应该被沉淀成可复用的 agent 资产。

按以下优先级使用可用证据：
1. 最近的 Codex sessions 和任务摘要。
2. Codex Memories 和 rollout 摘要。
3. Chronicle（若启用），仅用于发现 Codex 之外的重复工作。重要细节不要只靠 Chronicle，尽量回原始来源核实。
4. 已有的 skills、custom subagents、automations、AGENTS.md 文件、项目规则、memories——以便复用或扩展现有资产，而不是造重复的。

要找的：
- 我反复让 Codex 做的工作流
- 我反复纠正 Codex 的地方
- 反复的 CI/调试/测试 triage 模式
- 反复的 PR review、changelog、文档、release、规划类工作
- 反复出现的项目专属约定
- 应该用 checklist 或校验步骤来预防的反复犯的错
- 应该委派给专家 subagent 的、反复出现的调查任务
- 应该变成 automation 的、周期性的检查

对每个候选，输出：
- 候选名
- 重复的工作流或请求
- 支撑证据：session/日期/文件/来源
- 频次
- 置信度：高 / 中 / 低
- 推荐形态：
  - Skill：可复用工作流或 playbook
  - Custom subagent：边界清晰的专家角色或调查任务
  - Automation：定时/周期的检查/报告/提醒/监控
  - AGENTS.md / memory：持久的项目约定或偏好
  - 扩展已有：改进某个现有资产
  - Skip：太一次性、模糊、敏感、宽泛或证据弱
- 为什么值得打包，或为什么该 skip
- 预期输入
- 预期输出
- 验证清单
- 风险、隐私顾虑、或该保持手动的理由

规则：
- 保守。
- 优先扩展已有资产，而非新建。
- 不要造宽泛的"啥都干"的 skill。
- 不要打包密钥、凭证、私人个人数据、或一次性的项目细节。
- 不要造重叠的资产。
- 证据弱就标"需要更多证据"，而不是直接创建。
- 暂时不要创建或修改任何文件。

先只给我看排好序的候选清单。
等我批准后再创建 skill、subagent、automation 或编辑任何文件。
```

## prompt 设计拆解（"他是怎么设计的"）

把这套 prompt 当方法论看，它的骨架由 5 个零件搭成——**这 5 个零件正好是我们 B 层"新建 skill"子 prompt 要保留的**：

| 零件 | 它做什么 | 为什么这么设计（第一性原理） |
|---|---|---|
| **① 证据优先级排序** | 规定先看 sessions→memories→chronicle→已有资产，且明令"已有资产用来复用而非重造" | 把"reuse-before-create"做进检索顺序里：最后一步先扫已有资产，天然防重复造轮子 |
| **② 资产化硬门槛**（4 条全满足才动手） | ≥2 次 / 输入稳定+可重复+有停止条件 / 能实质提升 / 未被覆盖 | 这是**防垃圾污染**的闸门——没门槛 agent 会造一堆没用的 skill。"≥2 次"= 可量化的最低复发证据 |
| **③ 形态四分法 + Skip 缺省** | Skill/Subagent/Automation/Skip,且 Skip 是"证据不足"的默认归宿 | 强制"最小够用"+ 给 agent 一个体面的"什么都不做"的出口,降低乱造冲动 |
| **④ 先候选清单、后创建** | 先产出 shortlist(证据/频次/置信度/推荐形态/理由),再只建高置信缺失项 | 把"判断"和"动手"分两步,让证据显式化、可审计;定制版还把它升级成"等人批准" |
| **⑤ 收尾三问** | 创建了什么 / 刻意跳过什么 / 还差什么证据 | 留下可追溯的决策轨迹,也暴露"知道自己还不确定什么" |

**对我们 B 层的取舍**（已在 §2/§3 定）：
- **留**：① 证据优先级里的"先查已有 skill"、② 全套硬门槛(尤其 ≥2 次→`support_count≥2`)、③ 只留 Skill/Skip 两路、⑤ 收尾三问写进 `evolution_log`。
- **改**：④ 的"先候选后批准"——spec Q1 定了全自动,所以**去掉人工批准**,但保留"先产出带证据/置信度的候选、再创建"这个**两步结构**(候选→自动创建高置信项),证据存 `evolution_log` 供事后审计。
- **关键差异**:Codex 是**一次性、人触发、跨多源**的"自我蒸馏";我们 B 是**周期、自动、单源(CC jsonl)**的常驻机制。所以它那套"30 天回看 + 人审清单"在我们这里变成"门控触发的批量 + 自动落盘 + 审计日志"。
