---
name: agent-flow
description: 定义了主Agent调度检索、执行、命令三个子Agent的标准工作流、广播透明度、分级错误处理及兜底转交内置Agent（Explore/General-Purpose）的完整规范。当需要执行复杂的多步骤开发任务（如代码分析、修改、打包）时自动匹配。
---

# agent-flow · 多智能体调度工作流

> **调用命令**：`$agent-flow`（Skill 用 `$` 前缀；`/` 前缀为 Command 专用，不可用于 Skill）。
> **自动匹配**：当任务描述与本 `description` 意图匹配时，ZCode 会自动询问是否加载本 Skill，无需手动输入 `$agent-flow`。
> **配套文档**：三个子 Agent 的完整配置见 `agents.md`。

## 1. 核心思想

主 Agent（Orchestrator）不直接动手写代码 / 跑命令，而是**调度**三个专职子 Agent 完成「检索 → 执行 → 命令」流水线。主 Agent 的职责是：

1. **拆解任务** → 确定需要哪些子 Agent、按什么顺序。
2. **分发指令** → 给每个子 Agent 一段**自包含**、边界清晰的 prompt。
3. **广播透明** → 向用户实时汇报「正在调度哪个子 Agent、调用哪个模型」。
4. **分级错误处理** → 子 Agent 失败时，按级别升级（重试 → 兜底转交内置 Agent）。

## 2. 角色与内置 Agent 映射

| 子角色（逻辑层） | 内置 subagent_type（执行层） | 职责 | 可用工具 |
|---|---|---|---|
| 📡 检索（Retrieval） | `code-retriever`（首选，含 broad fan-out）/ `Explore`（兜底） | 精准定位代码（符号/定义/片段）**及** broad fan-out 大范围扫描目录与内容；Explore 仅在需联网/shell 管道时兜底 | Read, Grep, Glob（Explore 另含 Bash/WebSearch/WebFetch） |
| 🛠 执行（Execution） | `code-executor`（首选）/ `general-purpose`（兜底） | 严格按主 Agent 蓝图用 Edit/Write 精确改文件，不做额外改动 | Read, Edit, Write |
| ⚙️ 命令（Command） | `cmd-executor` | 安全执行 mvn/npm/docker/git 等 shell 命令并返回结果；可联网查阅 | Bash, WebFetch, WebSearch |

> 主 Agent 自身可访问全部工具，但遵循「能调度就不自己动手」的原则，仅在**无法拆分的小修小补**或**兜底**时直接处理。
> 各子 Agent 的颜色标记、系统提示词、完整工具列表见 **`agents.md`**。

## 3. 标准工作流（三阶段）

```
用户需求
   │
   ▼
[主 Agent] 拆解 + 制定执行蓝图（TodoWrite）
   │
   ├──► 阶段1 检索：code-retriever / Explore
   │       输入：文件列表 + 关键词   输出：定位结论（路径:行号、片段）
   │
   ├──► 阶段2 执行：code-executor / general-purpose
   │       输入：执行蓝图（基于阶段1结论）  输出：修改/新增的文件清单
   │
   └──► 阶段3 命令：cmd-executor
           输入：需执行的构建/测试/打包命令   输出：命令退出码与关键输出
   │
   ▼
[主 Agent] 汇总 → 广播给用户 → （失败则升级处理）
```

### 3.1 调度原则
- **prompt 自包含**：每个子 Agent 是全新会话，必须带上完整上下文（目标、文件、约束），不能依赖主 Agent 内存。
- **并行优先**：当阶段内存在多个**相互独立**的子任务（无共享状态、无顺序依赖）时，在同一条消息里并行发起多个 Agent 调用。多个独立检索点应发起**多个 code-retriever 并行**，各自定位一个符号/文件。
- **结论导向**：要求子 Agent 返回**结论**（结论 + 关键证据），而非整文件转储。

### 3.2 广播透明度（Transparency）
每次调度子 Agent 时，主 Agent 必须向用户**显式**报告：
1. 正在调度的**子角色 + 内置 agent 名称**（如 `code-retriever`）。
2. 该 agent **自身调用的模型名称**（由子 Agent 在回复中声明）。

> 示例："🔎 正在调度 **检索子Agent（code-retriever）** … 它报告自身模型为 `deepseek-v4-pro`。"

## 4. 分级错误处理与兜底

子 Agent 调用失败 / 结果不可用时，按**级别升级**，避免在同一级反复重试：

| 级别 | 触发条件 | 主 Agent 动作 |
|---|---|---|
| **L1 重试** | 子 Agent 报错 / 输出空 / 超时 | **重写更清晰的 prompt**（补上下文、缩范围），最多重试 1 次。检索类可换用 code-retriever 自身的 broad fan-out（Glob 扫命名 + Grep 批量搜内容） |
| **L2 兜底转交** | L1 仍失败 | 转交**另一类内置 Agent**：检索失败→`Explore`（利用其 Bash/WebSearch/WebFetch）；执行失败→`general-purpose`；命令失败→主 Agent 自行用 Bash |
| **L3 主 Agent 接管** | L2 仍失败 / 任务过小不值得调度 | 主 Agent 直接用自身工具完成，并在回复中**说明为何接管** |

> 升级时必须向用户广播："⚠️ `code-retriever` 失败（原因），升级为兜底 agent `Explore`。"

## 5. 输出契约（每个子 Agent 回复模板）

为了让主 Agent 高效汇总，子 Agent 回复应遵循轻量结构：

```
[角色] <agent 名称>
[模型] <自身调用的模型名>
[结论] <一两句话核心结果>
[证据] <file_path:line 或命令退出码 / 关键输出片段>
```

## 6. 边界与本 Skill 不做什么
- **不做**：浏览器自动化（由 `control-browser` skill 专用，且仅主 Agent 可用，禁止子 Agent 加载）。
- **不做**：一次性 2–3 行的小修补（直接主 Agent 用 Edit 更快）。
- **不做**：盲目合并/删除用户已有文件——改写前先 Read，与描述矛盾则先反馈不擅自执行。

## 7. 配套资源
本 Skill 目录下还提供子角色说明，供子 Agent 调度时参考引用：
- `agents.md` — **三个子 Agent 的完整配置**（名称 / 颜色 / 描述 / 工具 / 系统提示词）
- `retrieval.md` — 检索子 Agent 工作流规范
- `execution.md` — 执行子 Agent 工作流规范
- `command.md` — 命令子 Agent 工作流规范
- `README.md` — 快速上手说明
