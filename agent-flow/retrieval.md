# 📡 检索子 Agent 规范（Retrieval）

## 角色
负责"定位代码"，不做修改、不跑命令。

## 首选与兜底
- **首选**：`code-retriever` —— 项目代码检索专家，支持两类起点：
  - **起点①·pmem anchor（可选，需项目装有 project-memory skill）**：主Agent 已从 pmem 图谱提取精确 anchor（file:symbol），code-retriever 从 anchor 出发，沿调用链/依赖链做广度搜索（参照原生 Explore 机制：读 excerpts 不读全文、只定位不审查、返回切片不 dump）。pmem 的实体/关系/规则作为"硬包含"信息直接进蓝图，不靠检索。
  - **起点②·关键词（无 pmem 时）**：按文件列表 + 关键词定位；精准无结果则切 broad fan-out（Glob 扫命名 + Grep 宽匹配）。
  - **结果不足**：code-retriever 判断结果明显不足时，回报标注"结果不足建议"，主Agent 决定是否转交 Explore。

  （工具：Read, Grep, Glob）
- **兜底**：`Explore` —— 只读搜索 Agent，仅在 code-retriever 结果不足或 L1 失败后转交。它相比 code-retriever **多出 `Bash` / `WebSearch` / `WebFetch`**（可跑只读 shell 命令、可联网），故额外适用于：用 shell 管道做复杂文件筛选（如 `find ... | wc -l`）、或 code-retriever 搜不全时的补充盲扫。（工具：Read, Bash, WebFetch, WebSearch, TodoWrite）

> **能力对照**：纯代码 / 目录 / 配置的检索与扫描，`code-retriever` 的 Read/Grep/Glob 已完全胜任。只有需要 "**shell 管道**" 或 "**code-retriever 结果不足的补充**" 时，`Explore` 才具有不可替代价值。
>
> **⚠️ 联网检索不在本文件范畴**：查阅线上文档/API/技术资料等**联网检索**，首选 `cmd-executor`（见 `command.md` 联网检索升级链），L1 失败转交 `general-purpose`，不走 code-retriever / Explore。本文件的"首选/兜底"仅针对**本地代码检索**。
>
> **⚠️ pmem 为可选集成**：若项目未安装 project-memory skill，检索直接走"起点②·关键词"，不影响 agent-flow 正常运行。pmem 起点仅在项目装有 pmem 且主Agent 蓝图含 [检索起点] 字段时启用。

## 主 Agent 调度它时必须提供
1. **检索目的**（如："找到处理登录的 controller 以便后续修改入参校验"）。
2. **起点**（二选一）：
   - 若项目装有 pmem：**pmem anchor 列表**（file:symbol）+ [项目上下文]（实体/关系/规则，硬包含）。
   - 若无 pmem：**文件列表 / 目录范围 + 关键词 / 符号名**。
3. **广度提示**（可选，如"very thorough，沿调用链扩搜"）。

## 输出契约
```
[角色] code-retriever / Explore
[模型] <自身模型名>
[结论] 找到 / 未找到；目标符号定义在何处
[证据] path/to/file.java:42  的关键片段
```

> **检索类不套用"成功回摘要"**（见 `SKILL.md` §5.3）：检索结果是信息型产物，是主 Agent 后续工作的原料，**必须以"精确切片"（path:line + 关键片段）形式留在主 Agent 上下文**，不能压缩为"找到了"这种摘要（无价值），也不能回整文件（上下文爆炸）。token 控制**靠派发时给清范围/关键词/要几行**，让子 Agent 切得准。
>
> **pmem 偏差标注（pmem 起点任务专属）**：若使用 pmem anchor 起点，code-retriever Read anchor 时发现代码已不在该处（anchor 失效），必须在回报标注"pmem偏差:<具体不符>"，主Agent 收到后触发 pmem 归档更新。

## 何时升级（L1→L2）
- 返回空结果、找不到符号、或工具报错 → 主 Agent **重写更具体的 prompt 重试 1 次**（可换用 code-retriever 自身的 broad fan-out：Glob 扫命名 + Grep 批量搜内容）。
- 仍失败 → 转交 `Explore`，利用其额外的 `Bash`/`WebSearch`/`WebFetch`（联网或 shell 管道）突破本地文件检索的局限。
