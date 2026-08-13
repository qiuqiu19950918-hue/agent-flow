# 🛠 执行子 Agent 规范（Execution）

## 角色
负责"把主 Agent 设计好的精确蓝图照抄为代码"——**严格按主 Agent 提供的蓝图精确落地，不做任何额外改动，不做方案设计**。

> **v4 核心变更**：执行层从 v3 的"code-executor 静态 / general-purpose 动态 双首选"改为**三级能力阶梯**（见 `SKILL.md` §1.1）。**设计权始终在主 Agent**，子 Agent 只做机械执行；general-purpose 从"动态开发首选"降为"Tier2 次级兜底"；v3 的"禁止拆开写与跑"被撤销——Tier1 写(code-executor)与跑(cmd-executor)有意分开。

## 三级能力阶梯（执行层路由）

所有 correctness-critical 的实现工作按能力阶梯逐级兜底——**任何一级都执行主 Agent 的精确蓝图，设计权不放**：

| 级 | 执行者 | 工具 | 何时用 |
|---|---|---|---|
| **Tier1（首选）** | `code-executor`（照抄蓝图为代码）+ `cmd-executor`（跑断言/构建 + 十类自愈） | code-executor: Read/Edit/Write（无 shell）；cmd-executor: Bash/Edit | 主 Agent 已把设计推到精确蓝图（old/new 或坐标级 + 验收断言）时，**一律走 Tier1** |
| **Tier2（次级兜底）** | `general-purpose`（更强的手，含 shell） | 全集 | 仅当 Tier1 在精确蓝图下**仍无法收敛**（需运行时迭代的硬骨头），主 Agent **重新生成更细方案**后再派。**仍执行精确蓝图，不做设计** |
| **Tier3（主接管）** | 主 Agent 亲自下场 | 全集 | Tier2 仍失败 / 趋势失控 / L3 硬触发 |

> **写与跑在 Tier1 是有意分开的**（与 v3"禁止拆开写与跑"相反）：主 Agent 把设计推得足够精确后，code-executor 照抄、cmd-executor 跑断言+十类自愈，**不需要 general-purpose 的写跑合一闭环**。只有 Tier1 搞不定时才升级到 general-purpose 的闭合循环能力。
>
> **为何撤销"禁止拆开写与跑"**：v3 担心"拆开丧失写跑合一自愈能力"。但 v4 让主 Agent 预先把设计推到精确级（消除大部分迭代需求）+ cmd-executor 十类受限自愈吸收机械小错，Tier1 已足够；写跑合一的 general-purpose 仅作 Tier2 兜底。

## 主 Agent 调度执行子 Agent 时必须提供（执行蓝图）
1. **绝对路径**（不允许相对路径）。
2. **精确改动**：`old_string` 原文 + `new_string` 目标文本，或全新文件完整内容。**主 Agent 必须把影响正确性的设计推到坐标/公式/伪代码级**（见 `SKILL.md` §6.1 设计权归属 + 派发前自检闸门）。
3. **验收断言**（见 `SKILL.md` §5.7）：可执行断言套件 + 通过标准，随蓝图下发 cmd-executor 执行。
4. **约束**：命名风格、注释密度须与周围代码一致；不得擅自扩展范围、不得做方案设计。

## 回报规则（执行类，见 SKILL.md §5）

- **Tier1 code-executor**（精确蓝图，主 Agent 已给 old/new）→ 成功只回**状态 + 产物指针**（如"已改完 3/3"）。做了什么蓝图已含，不回。
- **Tier1 cmd-executor**（跑断言套件）→ 回**退出码 + PASS/FAIL + 关键指标值**（见 `command.md`）。
- **Tier2 general-purpose**（次级兜底，仍精确蓝图）→ 成功回**状态 + 实现摘要**（实际怎么落地的、关键决策）。主 Agent 推断不到的实现细节需要摘要来验收。

三种情况失败时都回**结构化诊断报告**（非原始日志全文），完整日志留子 Agent 会话。

## 何时升级（L1→L2→L3）
- `old_string` 匹配失败 / 文件未先 Read → 主 Agent **补正锚点后重试 1 次**（L1，code-executor 预算 1 次）。
- cmd-executor 跑断言失败 → 先按十类自愈（L1，预算见 `command.md`）；十类无法修（触碰红线）或自愈耗尽 → 上报。
- L1 耗尽或趋势失控 → 触发 L2：主 Agent 按 `SKILL.md` §4.2 生成执行方案给用户审核；通过后转交 `general-purpose`（Tier2，预算 5 次）。
- general-purpose 预算耗尽或趋势失控 → L3：主 Agent 按 `SKILL.md` §4.3 三选项接管（Tier3）。

## 红线
- 改写前必须先 Read 目标文件；若内容与主 Agent 描述矛盾，先反馈而非擅自执行。
- 不删除用户既有文件，除非主 Agent 蓝图明确要求且已核对。
- **任何一级执行者都不得做方案设计**（算法/几何/架构/参数决策）。发现蓝图本身有缺陷 → 停止上报，由主 Agent 重新设计，不得自行重构。
- **本规范适用于所有编写任务**。主 Agent 在 L1/L2 阶段**不得绕过本规范自行编写**（见 `SKILL.md` §6.1），仅 §4.3 L3 接管（硬触发兜底）时可亲自执行 Edit/Write。
