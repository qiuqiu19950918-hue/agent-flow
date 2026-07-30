# 📡 检索子 Agent 规范（Retrieval）

## 角色
负责"定位代码"，不做修改、不跑命令。

## 首选与兜底
- **首选**：`code-retriever` —— 项目代码检索专家，**同时擅长两类检索**：
  - **精准定位**：按文件列表 + 关键词，快速定位函数定义、符号、配置项、代码片段。
  - **broad fan-out（大范围扫描）**：用 `Glob` 按模式批量扫描目录结构与多种命名约定，用 `Grep` 跨文件批量搜内容。本工作流的大范围检索任务同样**优先用 code-retriever**，不必引入 Explore。

  （工具：Read, Grep, Glob）
- **兜底**：`Explore` —— 只读搜索 Agent，仅在 `code-retriever` L1 失败后转交。它相比 code-retriever **多出 `Bash` / `WebSearch` / `WebFetch`**（可联网检索、跑只读 shell 命令），故额外适用于：需要查阅线上文档/API、或用 shell 管道做复杂文件筛选（如 `find ... | wc -l`）。（工具：Read, Bash, WebFetch, WebSearch, TodoWrite）

> **能力对照**：纯代码 / 目录 / 配置的检索与扫描，`code-retriever` 的 Read/Grep/Glob 已完全胜任。只有需要"**联网**"或"**shell 管道**"时，`Explore` 才具有不可替代价值——这也正是它作为兜底而非首选的原因。

## 主 Agent 调度它时必须提供
1. **文件列表 / 目录范围**（缩小搜索面，避免全仓扫描）。
2. **关键词 / 符号名**（函数名、类名、配置项等）。
3. **检索目的**（如："找到处理登录的 controller 以便后续修改入参校验"）。

## 输出契约
```
[角色] code-retriever / Explore
[模型] <自身模型名>
[结论] 找到 / 未找到；目标符号定义在何处
[证据] path/to/file.java:42  的关键片段
```

## 何时升级（L1→L2）
- 返回空结果、找不到符号、或工具报错 → 主 Agent **重写更具体的 prompt 重试 1 次**（可换用 code-retriever 自身的 broad fan-out：Glob 扫命名 + Grep 批量搜内容）。
- 仍失败 → 转交 `Explore`，利用其额外的 `Bash`/`WebSearch`/`WebFetch`（联网或 shell 管道）突破本地文件检索的局限。
