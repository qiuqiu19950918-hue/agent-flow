# agent-flow Skill

多智能体调度工作流：主 Agent 调度"检索 → 执行 → 命令"三个子 Agent，含广播透明度、分级错误处理与兜底转交内置 Agent 机制。

## 调用
- 手动：`$agent-flow`
- 自动：任务描述与 SKILL.md 的 description 意图匹配时自动加载

## 文件
| 文件 | 说明 |
|---|---|
| `SKILL.md` | Skill 入口：元数据 + 工作流规范 + 分级错误处理 |
| `retrieval.md` | 检索子 Agent 规范（code-retriever / Explore 兜底） |
| `execution.md` | 执行子 Agent 规范（code-executor / general-purpose 兜底） |
| `command.md` | 命令子 Agent 规范（cmd-executor） |

## 安装路径
全局：`~/.zcode/skills/agent-flow/SKILL.md`
