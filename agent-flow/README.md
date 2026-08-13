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
| 🔧 执行（Tier2 兜底） | `general-purpose` | 仅当 Tier1(code-executor+cmd-executor) 无法收敛时承接更难的实现；仍执行主的精确蓝图，不做设计 |
| ⚙️ 命令 | `cmd-executor` | 安全执行 shell 命令；可对已写好脚本做受限自愈 |

## 核心机制

- **执行者选择（v4 三级阶梯）**：Tier1 code-executor(照抄)+cmd-executor(跑/自愈) 首选；Tier1 无法收敛 → Tier2 general-purpose(更强的手，仍精确蓝图)；Tier3 主接管。设计权始终在主 Agent。
- **设计权归属（v4 最高原则）**：主 Agent 拥有所有影响正确性的设计（架构/算法/几何/参数/验收方案）；子 Agent 只做机械执行。派发前自检闸门："影响正确性的设计我亲手推完了吗？"详见 SKILL.md §6.1。
- **验收形式化（v4）**：主 Agent 把验收设计为可执行断言套件，随蓝图下发 cmd-executor 执行；无法形式化的残余（视觉等）主自留。详见 SKILL.md §5.7。
- **自愈预算**（防无限循环）：L1 当前角色自愈（cmd ≤3 次）；L2 general-purpose ≤5 次；总迭代 ≤8 轮；趋势失控立即上交。
- **cmd-executor 自愈授权契约**：十类可改（路径/标识符/字面量/依赖/表达式/边界/API/错误处理/输出/格式）+ 四维红线（函数边界/控制流/数据流/对外契约），结构化上报（三要素）+ 分层验收。
- **L2 前置审核闸门**：转交 general-purpose 前先生成执行方案给用户审核，排除"方案设计缺陷"风险。
- **L3 接管三选项**：从末版继续修 / 推倒重写 / 重新设计方案。
- **主 Agent 编写/执行权限（防呆）**：L1/L2 阶段**绝对禁止**主 Agent 编写（Edit/Write）/执行（Bash），一律派子 Agent，**无判据、无规模阈值、无主观豁免**；仅 §4.3 L3 硬触发（L1+L2 预算耗尽 + 趋势失控的客观条件）时解锁。配 4 项机制（执行前显式判定 / 调度强制 / 调度强制下限 / 交付报告调度审计）。详见 SKILL.md §6.1。
- **输出契约 v2（回报=实际发生−蓝图已含）**：子 Agent 只回主 Agent 推断不到的部分。详细内容留子 Agent 会话不回灌；按任务类型分流（执行/命令类→状态+指针，检索类→精确切片，语义蓝图→实现摘要）；失败回结构化诊断非原始日志。详见 SKILL.md §5。
- **输出契约 v3（派发强制锁死）**：v2 契约写在文档里但子 Agent 实测不遵循（仍复述蓝图）。v3 要求主 Agent 派发执行/命令类任务时蓝图必含 `[返回格式]` 字段，把约束从"被动遵循"升级为"主动锁死"。详见 SKILL.md §5.5。
- **pmem 集成（可选）**：若项目装有 project-memory skill，检索走"pmem 起点锚定"——主 Agent 用 pmem 1-hop 框选范围（结构关系硬包含进蓝图），anchor 作 code-retriever 检索起点，用原生 Explore 广度搜详细代码+衍生；结果不足转 Explore 盲扫；anchor 失效标注触发 pmem 更新。无 pmem 时退化为关键词检索，不影响运行。详见 SKILL.md §3.1。

## 版本变更说明

| 版本 | 改了啥 | 为啥这么改 |
|---|---|---|
| **设计权归属 + 三级阶梯（v4，本轮）** | SKILL.md 新增 §1.1 三级能力阶梯 + §6.1 设计权归属与派发前自检闸门 + §5.7 验收断言形式化；§2/§3 路由翻转（general-purpose 首选→Tier2 兜底）；execution.md 整体重写为三级阶梯（撤销"禁止拆开写与跑"）；agents.md general-purpose 定位降级、code-executor/cmd-executor 角色同步 | 实测（4 组对比实验）：主 Agent 把设计权下放给子 Agent（语义蓝图）时，产物质量全靠子 Agent 能力——弱子 Agent(flash)认知过载空响应烧 220 万 token；主 Agent 亲手做完全部设计给精确蓝图后，弱子 Agent 也"全一次过零自愈"。子 Agent 命运由主蓝图精度决定。故把"主做全部设计"从最佳实践提升为硬规则，general-purpose 从首选降为兜底以堵死放权 |
| **权限反转（上一轮）** | SKILL.md §6.1 彻底重写：额度制（单文件≤2处且文件总数≤2个）→ **无判据硬规则**（L1/L2 阶段绝对禁止主 Agent 编写/执行，一律派子 Agent）；删除所有规模判据与主观豁免（含调试性临时改动、锚点现场确认）；仅保留 §4.3 L3 硬触发（预算耗尽+趋势失控）为唯一解锁。SKILL §1/§1.5、execution.md 红线段、command.md 角色/联网段同步措辞强化"常规禁主 Agent 动手" | 实测任何判据（规模/context污染风险）都会被主 Agent 找借口绕过；且大文件改 1 行也会灌 13K token 进主 context 永久清不掉。唯一可靠规则是"无判据硬禁止 + 硬触发兜底"。执行验证任务（grep/wc/测试）也禁主 Agent 自己跑，含决策依赖型执行改为 cmd-executor 回传关键行 |
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
| `execution.md` | 执行子 Agent 规范（v4：三级能力阶梯、Tier1 code-executor+cmd-executor / Tier2 general-purpose 兜底）|
| `command.md` | 命令子 Agent 规范（cmd-executor + 自愈授权契约） |
| `agents.md` | 四个子 Agent 完整配置（名称/颜色/描述/工具/系统提示词） |

## 安装路径
全局：`~/.agents/skills/agent-flow/SKILL.md`
