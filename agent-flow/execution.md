# 🛠 执行子 Agent 规范（Execution）

## 角色
负责"改代码 / 写代码"，严格按主 Agent 提供的蓝图精确落地，**不做任何额外改动**。

## 首选与兜底
- **静态编辑首选**：`code-executor` —— 代码实施专家，使用 Edit/Write 精确修改或新增文件。（工具：Read, Edit, Write）
- **动态开发首选**：`general-purpose` —— 通用多步任务 Agent，工具全集（含 Bash），负责**需要运行时反馈的代码**（脚本、可执行代码、需 build 验证的代码）。它天然闭合「写→跑→看错→改→再跑」循环，是脚本开发的主力。
- **兜底**：code-executor 失败时转交 `general-purpose`；general-purpose 失败时由主 Agent 接管（L3）。

## 执行者选择规则（按是否需要运行时反馈）

| 任务特征 | 执行者 | 理由 |
|---|---|---|
| **静态编辑**（改完不需跑验证）：业务代码修改、文档编写、配置改动 | `code-executor` | 无循环，精确蓝图最省 |
| **动态开发**（写完必须跑、看错、迭代）：py/js 脚本、迁移脚本、需 build 验证的代码 | `general-purpose` | 闭合写跑循环，单 Agent 折叠迭代，省 H 请求 |
| **纯命令**（产物是别人写的或无产物）：构建/测试/git/部署 | `cmd-executor` | 无需 Edit（除受限自愈） |

**判断技巧**：分水岭是"这个产物要不要执行验证"。要 → general-purpose；不要 → code-executor。

> **禁止拆开写与跑（硬规则）**：动态开发任务的「写脚本」和「跑脚本验证」**必须由同一个 general-purpose 闭合**，不得拆成"general-purpose 写 → 主 Agent 居中 → cmd-executor 跑"。拆开的代价：① 多 1 次主 Agent 调度往返；② 失去"跑出来报错就自己修"的自愈能力，失败要回流主 Agent。即便脚本看似无迭代需求，也应让 general-purpose 写完即自跑验证——跑成功则结果直接回流，跑失败（如依赖未装/字段错）则自行修复重跑（自愈预算 ≤5 次），均无需主 Agent 介入。
>
> **cmd-executor 的职责不变**：只跑**现成的、别人写的或已存在的**命令/脚本（构建、测试、打包、git、跑现成 .py），以及按自愈授权契约对已写好脚本做受限小修。cmd-executor 不承担"动态开发产物的首跑验证"——那是 general-purpose 的职责。

**典型反例**：主 Agent 让 general-purpose"只写不跑，运行交给后续命令子 Agent"——这把 general-purpose 退化成了 code-executor，丧失了动态开发的核心价值（闭合循环）。正确做法：蓝图直接写"编写脚本并自行运行验证，跑通后返回结果汇总，如遇环境问题自行修复重跑"。

## 主 Agent 调度它时必须提供（执行蓝图）
1. **绝对路径**（不允许相对路径）。
2. **精确改动**：`old_string` 原文 + `new_string` 目标文本，或全新文件内容。
3. **约束**：命名风格、注释密度须与周围代码一致；不得擅自扩展范围。

## 输出契约
```
[角色] code-executor / general-purpose
[模型] <自身模型名>
[结论] 已修改 / 新增的文件清单 + 每处改动一句话摘要
[证据] path/to/file.ext —— <改动摘要>
```

## 何时升级（L1→L2→L3）
- `old_string` 匹配失败 / 文件未先 Read → 主 Agent **补正锚点后重试 1 次**（L1，code-executor 预算 1 次，详见 `SKILL.md` §4.1）。
- 仍失败 → 触发 L2：主 Agent 按 `SKILL.md` §4.2 生成执行方案给用户审核；通过后转交 `general-purpose`（预算 5 次）。
- general-purpose 预算耗尽或趋势失控 → L3：主 Agent 按 `SKILL.md` §4.3 三选项接管。

## 红线
- 改写前必须先 Read 目标文件；若内容与主 Agent 描述矛盾，先反馈而非擅自执行。
- 不删除用户既有文件，除非主 Agent 蓝图明确要求且已核对。
