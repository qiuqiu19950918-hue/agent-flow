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
