# agent-flow

> ZCode 多智能体调度 Skill —— 让主 Agent 通过「检索 → 执行 → 命令」三阶段工作流编排子智能体，完成复杂的多步骤开发任务。

[![Skill](https://img.shields.io/badge/ZCode-Skill-blue)](https://github.com/qiuqiu19950918-hue/agent-flow) [![License: MIT](https://img.shields.io/badge/License-MIT-yellow)](./LICENSE)

## ✨ 核心特性

- **三阶段流水线**：检索（定位代码）→ 执行（改写代码）→ 命令（构建/测试/打包）
- **并行优先**：多个独立子任务在同一条消息里并行调度
- **广播透明**：每次调度都向用户显式报告子 Agent 名称与所用模型
- **分级容错**：L1 重试 → L2 兜底转交（Explore / general-purpose）→ L3 主 Agent 接管
- **broad fan-out**：code-retriever 用 Glob/Grep 大范围扫描，作为一等能力

## 🚀 快速开始

### 安装（全局，对所有项目生效）

```bash
# 方式一：克隆本仓库
git clone https://github.com/qiuqiu19950918-hue/agent-flow.git
cd agent-flow
# 方式二：将 agent-flow/ 目录复制到 ZCode 的全局 Skill 目录
cp -r agent-flow ~/.agents/skills/agent-flow
```

> Windows 路径示例：`C:\Users\<你>\.agents\skills\agent-flow\`

### 依赖的子 Agent

本 Skill 依赖三个内置子 Agent，配置见 [`agent-flow/agents.md`](./agent-flow/agents.md)：

| 子 Agent | 颜色 | 职责 |
|---|---|---|
| `code-retriever` | 蓝色 | 检索定位代码 + broad fan-out 大范围扫描 |
| `Code-Executor` | 绿色 | 按蓝图精确改写文件 |
| `cmd-executor` | 橙色 | 安全执行构建/测试/部署命令 |

请按 `agents.md` 在 ZCode 客户端「子智能体」中复刻这三个子 Agent。

### 调用

- **手动**：`$agent-flow`
- **自动**：任务描述与 `agent-flow/SKILL.md` 的 description 意图匹配时自动加载

## 📁 目录结构

```
agent-flow/
├── SKILL.md        # Skill 入口：元数据 + 工作流规范 + 分级错误处理
├── agents.md       # 三个子 Agent 的完整配置（名称/颜色/描述/工具/系统提示词）
├── retrieval.md    # 检索子 Agent 工作流规范
├── execution.md    # 执行子 Agent 工作流规范
├── command.md      # 命令子 Agent 工作流规范
└── README.md       # Skill 级快速上手
```

## 📖 文档

- 工作流总览：[`agent-flow/SKILL.md`](./agent-flow/SKILL.md)
- 子 Agent 配置：[`agent-flow/agents.md`](./agent-flow/agents.md)

## 🤝 贡献

欢迎提 Issue 和 PR。请先阅读 [贡献指南](./CONTRIBUTING.md)。

## 📄 许可证

[MIT License](./LICENSE)
