# agent-flow Skill

多智能体调度工作流：主 Agent 调度"检索 → 执行 → 命令"子 Agent，含广播透明度、分级错误处理、自愈预算、L2 前置审核闸门、L3 主 Agent 接管三选项与兜底转交内置 Agent 机制。

## 调用
- 手动：`$agent-flow`
- 自动：任务描述与 SKILL.md 的 description 意图匹配时自动加载

## 四个子 Agent

| 角色 | 子 Agent | 职责 |
|---|---|---|
| 📡 检索 | `code-retriever` / `Explore`（兜底） | 定位代码、broad fan-out 大范围扫描 |
| 🛠 执行（静态） | `code-executor` | 业务代码/文档/配置的精确编辑（改完不需跑验证）|
| 🔧 执行（动态） | `general-purpose` | 脚本/可执行代码开发（写→跑→改→再跑闭合循环），执行类兜底 |
| ⚙️ 命令 | `cmd-executor` | 安全执行 shell 命令；可对已写好脚本做受限自愈 |

## 核心机制

- **执行者选择**：按"产物是否需要运行时反馈"路由——静态编辑 → code-executor；动态开发 → general-purpose；纯命令 → cmd-executor。
- **自愈预算**（防无限循环）：L1 当前角色自愈（cmd ≤3 次）；L2 general-purpose ≤5 次；总迭代 ≤8 轮；趋势失控立即上交。
- **cmd-executor 自愈授权契约**：十类可改（路径/标识符/字面量/依赖/表达式/边界/API/错误处理/输出/格式）+ 四维红线（函数边界/控制流/数据流/对外契约），结构化上报（三要素）+ 分层验收。
- **L2 前置审核闸门**：转交 general-purpose 前先生成执行方案给用户审核，排除"方案设计缺陷"风险。
- **L3 接管三选项**：从末版继续修 / 推倒重写 / 重新设计方案。
- **主 Agent 自修额度（防呆）**：单文件 ≤2 处 **且** 总文件数 ≤2 个方可主 Agent 自行 Edit；超出必须派 code-executor / general-purpose。配 4 项机制（执行前显式判定 / 硬阈值红线 / 调度强制下限 / 交付报告调度审计），防止便利性偏差虚耗高性能额度。详见 SKILL.md §6.1。
- **输出契约 v2（回报=实际发生−蓝图已含）**：子 Agent 只回主 Agent 推断不到的部分。详细内容留子 Agent 会话不回灌；按任务类型分流（执行/命令类→状态+指针，检索类→精确切片，语义蓝图→实现摘要）；失败回结构化诊断非原始日志。详见 SKILL.md §5。
- **输出契约 v3（派发强制锁死）**：v2 契约写在文档里但子 Agent 实测不遵循（仍复述蓝图）。v3 要求主 Agent 派发执行/命令类任务时蓝图必含 `[返回格式]` 字段，把约束从"被动遵循"升级为"主动锁死"。详见 SKILL.md §5.5。
- **pmem 集成（可选）**：若项目装有 project-memory skill，检索走"pmem 起点锚定"——主 Agent 用 pmem 1-hop 框选范围（结构关系硬包含进蓝图），anchor 作 code-retriever 检索起点，用原生 Explore 广度搜详细代码+衍生；结果不足转 Explore 盲扫；anchor 失效标注触发 pmem 更新。无 pmem 时退化为关键词检索，不影响运行。详见 SKILL.md §3.1。

## 版本变更说明

| 版本 | 改了啥 | 为啥这么改 |
|---|---|---|
| **pmem 集成 + cmd 联网检索（本轮）**| retrieval.md 改为 pmem 起点锚定+Explore 原生范围+无 pmem 退化+结果不足转 Explore；code-retriever 描述/提示词加两类起点；cmd-executor 提示词加联网检索段；SKILL.md §3.1 加 pmem 快路径原则；command.md 补本地/联网检索分工 | 让检索有精确起点（pmem anchor）而非盲扫，结构关系硬包含防漏检；pmem 为可选，无 pmem 退化到关键词不影响运行。cmd 承担联网检索与本地检索分工明确 |
| 输出契约 v3 | §5.4 模板升级；新增 §5.5 主 Agent 派发强制规则（`[返回格式]` 字段必填 + 两种任务模板）；原 §5.5 验收规则顺延为 §5.6 并补"违反返回格式时的处置" | 实测 v2 上线后子 Agent（deepseek-v4-flash）仍按 v1 风格详尽复述蓝图，单次回复冗余 ~75%。根因：契约在文档里约束力弱，低性能子 Agent 不主动遵循。v3 把约束点从"子 Agent 自觉"前移到"主 Agent 派发时强制锁死"，让 v2 的理论精简真正落地 |
| 输出契约 v2 | §5 重写为"回报=实际发生−蓝图已含"统一原则 + 按任务类型/蓝图强度分流细则；retrieval/command/execution 各补回报注 | 前版"结论导向"太模糊，子 Agent 自律不可靠导致主 Agent 上下文膨胀。新原则精确界定"回什么"：蓝图已含的不回、推断不到的才回。详细内容留子 Agent 会话不回灌（harness 整条全量回灌且主 Agent 无法主动丢弃，控制窗口期在子 Agent 写 final message 之前） |
| 写跑合一（上一版）| execution.md 禁止拆开写与跑；general-purpose 强化"写完即跑不可拆开" | 防止主 Agent 把动态开发拆成"GP写→cmd跑"的乒乓，丧失闭合循环自愈能力 |
| 联网检索路由（上一版）| cmd-executor 承担联网检索首选 + 联网检索升级链 | 厘清本地检索（code-retriever）与联网检索（cmd-executor）边界 |
| 浏览器分级授权（上一版）| §6.0 解除一刀切禁令，按验证类型分级 | 实测子 Agent 能用 MCP 浏览器工具，政策性禁令可解除；主观视觉验证仍归主 Agent（子 Agent 看不了图） |

## 文件
| 文件 | 说明 |
|---|---|
| `SKILL.md` | Skill 入口：元数据 + 工作流规范 + §4 分级错误处理（含预算/闸门/接管三选项）|
| `retrieval.md` | 检索子 Agent 规范（code-retriever / Explore 兜底） |
| `execution.md` | 执行子 Agent 规范（含执行者选择规则、code-executor / general-purpose）|
| `command.md` | 命令子 Agent 规范（cmd-executor + 自愈授权契约） |
| `agents.md` | 四个子 Agent 完整配置（名称/颜色/描述/工具/系统提示词） |

## 安装路径
全局：`~/.agents/skills/agent-flow/SKILL.md`
