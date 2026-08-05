# ⚙️ 命令子 Agent 规范（Command）

## 角色
负责"跑命令"——构建、测试、打包、安装依赖、git 操作等 shell 命令的**安全执行**。

## 首选
- **命令执行唯一**：`cmd-executor` —— 系统命令执行专家，负责安全地执行 Bash 命令（mvn package、npm install、docker build 等）并返回结果；可对已写好脚本做受限自愈。（工具：Bash、Edit、WebFetch、WebSearch）
- **联网检索首选**：`cmd-executor` —— 拥有 `WebSearch` / `WebFetch` 工具，是 agent-flow 中**联网检索的首选执行者**（查阅线上文档/API、搜索技术资料、抓取网页内容）。检索失败时按下方「联网检索升级链」升级。

> 命令失败时一般**不换 agent**，而是由主 Agent 决策：修复 → 重跑，或主 Agent 自行用 Bash 接管（L3）。

## 主 Agent 调度它时必须提供
1. **工作目录**（绝对路径，避免 `cd` 触发权限提示）。
2. **完整命令**（如 `mvn -q -DskipTests package`）。
3. **期望**（成功标志 / 关注的输出关键字 / 超时预算）。

## 输出契约
```
[角色] cmd-executor
[模型] <自身模型名>
[结论] 成功 / 失败；退出码
[证据] <关键输出片段（编译错误行 / BUILD SUCCESS / Tests run 等）>
```

> **命令类成功只回"状态 + 关键行"，不回"做了什么"**（见 `SKILL.md` §5.1/5.3）：主 Agent 已从命令蓝图知道跑了什么命令，"做了什么"无需回灌。子 Agent 只补主 Agent 推断不到的运行时事实（退出码、关键输出关键字、产物路径、commit hash）。**完整命令日志只在子 Agent 会话内展示，不回灌**；失败时回结构化诊断报告，非原始 stderr 全文。

## 何时升级（L1→L2→L3）
- 命令非零退出 → cmd-executor 先按下方「自愈授权契约」尝试自愈（L1，预算 3 次，详见 `SKILL.md` §4.1）：智能重试 + 受限 Edit 修脚本。
- L1 耗尽或趋势失控 → 主 Agent 判断根因：**代码问题**（脚本逻辑错，转 L2 `general-purpose`）还是**环境问题**（缺依赖，主 Agent 修复后重跑）。
- 触发 L2 时，主 Agent 按 `SKILL.md` §4.2 生成执行方案给用户审核；通过后放行 `general-purpose`（预算 5 次）。
- general-purpose 耗尽或审核未通过 → L3：主 Agent 按 `SKILL.md` §4.3 三选项接管。

## 自愈授权契约（cmd-executor 对已写好脚本的受限修改）

cmd-executor 已获 `Edit` 工具，可对 code-executor / general-purpose 写好的脚本做**受限自愈**，避免小错回流主 Agent。

### 可改清单（十类实现层偏差，改了不改变处理流程结构）
| 类别 | 内容 | 例子 |
|---|---|---|
| 路径与位置 | 文件路径、输出目录、`__dirname` 误用 | `__dirname` → `.aiknowledge/` 根 |
| 标识符修正 | 变量名 / 字段名 / 常量名写错 | `.name` → `.tableName`、`repair_part_id` → `repair_part_ids` |
| 字面量修正 | 字符串 / 数值 / 布尔常量写错 | 阈值 `0.8` → `0.6`、分隔符 `";"` → `","` |
| 依赖补全 | 缺失的 import / require | 补 `import yaml from 'js-yaml'` |
| 表达式修正 | 运算符 / 比较符 / 正则 / 类型转换写反 | `===` → `!==`、`String(x)` → `Number(x)` |
| 边界修正 | 循环起止 / 数组索引 / 切片范围 | `i <= len` → `i < len` |
| API 调用补全 | 调用参数缺失 / 顺序错 | `fs.readFile(path)` → `fs.readFile(path, 'utf8')` |
| 错误处理补强 | 给裸奔调用包 try-catch、补空值兜底 | 给 `JSON.parse` 包 try-catch |
| 输出 / 日志调整 | console.log 内容、输出格式 | `console.log(x)` → `console.log('migrate:', x)` |
| 注释与格式 | 注释 / 缩进 / 换行 / 风格 | 无功能影响 |

> **风险分级**（供上报与验收用）：路径 / 标识符 / 字面量 / 依赖 / 注释格式 = **低风险**（自明性高）；错误处理补强 / 输出日志 = **中风险**；表达式 / 边界 / API 调用 = **高风险**（可能改对也可能改错语义）。

### 红线（禁止，触即上报，不得自愈）
**逻辑架构重构 = 改了会动到以下任一维度**：
1. **函数边界**：新增 / 删除 / 拆分 / 合并函数；改函数签名（增删参数、改返回值结构）。
2. **控制流结构**：同步↔异步转换、回调↔Promise↔async/await 互转、改变循环 / 分支整体结构。
3. **数据流**：改变数据流向（如"先读后写"→"边读边写"）、改变中间数据结构、改变处理顺序。
4. **对外契约**：改变脚本对外的输入 / 输出契约（CLI 参数、文件格式、模块导出）、改公共接口名。

> **自检口诀**：改动需要先"理解这段代码为什么这么设计"才能动手 → 那是重构，不是自愈 → 必须上报。

### 预算与止损
- **处数**：单次自愈循环内累计 ≤ **8 处**改动（覆盖中等脚本调通期常见小错 4~6 处 + 余量）。
- **连续失败**：同一脚本连续跑失败 ≤ **3 次**（含自愈）。
- **新报错停止**：自愈中若出现原本没有的新报错，立即停止上报（说明越改越乱）。
- **高风险占比止损**：自愈改动中高风险类（表达式 / 边界 / API 调用）占比 > **50%** → 视为趋势失控，提前上报（说明问题不是"小修"而是逻辑有缺陷）。
- 任一触发 → 按上文「何时升级」上交。

### 自愈蓝图模板（主 Agent 下发 cmd-executor 时附加）
```
[工作目录] <绝对路径>
[脚本] <path>（已由 code-executor / general-purpose 写好）
[运行] node <script>（或 python <script>）
[期望]
  成功标志：<如 BUILD SUCCESS / Tests run: N, Failures: 0 / 输出落 .aiknowledge/ 根>
  失败信号：<如 ERROR / BUILD FAILURE / 路径仍是 temp/>
[自愈授权]
  可改范围：见 command.md 自愈授权契约十类
  预算：累计 ≤8 处，连续失败 ≤3 次，高风险占比 ≤50%
  红线（触即上报）：函数边界 / 控制流 / 数据流 / 对外契约
  收敛要求：出现新报错立即停止上报
[上报格式]
  触发条件 + 已改动处清单（每处三要素）+ 当前阻塞点
```

### 结构化上报格式（强制三要素）
自愈失败上报时，每处改动必须含三要素，让主 Agent 不必再 Read 文件即可验收：
```
[自愈改动清单]
1. migrate.mjs:142  [风险:低-路径]
   - old: const outDir = path.join(__dirname, '.aiknowledge')
   + new: const outDir = path.join(process.cwd(), '.aiknowledge')
2. migrate.mjs:88   [风险:低-标识符]
   - old: entity.repair_part_id
   + new: entity.repair_part_ids
3. migrate.mjs:203  [风险:高-表达式]
   - old: if (val === target) flag = false;
   + new: if (val !== target) flag = false;
```
- `file:line`：定位。
- `[风险:级别-类别]`：cmd-executor 自评，供主 Agent 决定验收深度。
- `old / new`：改动内容，让主 Agent 直接判断对错。

### 分层验收（主 Agent 验收规则，控成本）
自愈权限放大后，主 Agent 验收从"浅扫"升级，但**按风险分层**，不和"总处数"成正比：

| 风险等级 | 验收方式 | 成本 |
|---|---|---|
| **低**（路径 / 标识符 / 字面量 / 依赖 / 注释格式）| 扫一眼即可信（自明性高，old/new 一看对错）| 极低 |
| **中**（错误处理补强 / 输出日志）| 抽样核 1~2 处确认没改坏 | 低 |
| **高**（表达式 / 边界 / API 调用）| **逐条核**（必要时针对性 Read 该处周围区域，非全文件）| 中 |

> 多数情况下主 Agent 可在**单 turn 内并行核对全部变更**（8 处 × 小片段，上下文很小）。仅当高风险处多且片段不足以判语义时，才针对性 Read，仍尽量批量塞进同一 turn。
>
> **质量对价提醒**：分层验收的前提是 cmd-executor 如实标注风险等级。若主 Agent 发现"标注为低风险实为高风险"的瞒报，应将该次自愈结果整体作废、转 L2。

## 安全约束
- 交互式 flag（如 `git rebase -i`、`git add -i`）不可用。
- 仅在用户明确要求时才 commit / push；在默认分支上需先建分支。

## 联网检索升级链（WebSearch / WebFetch 失败处理）

> **分工**：本地代码/文件检索归 code-retriever（见 retrieval.md，支持 pmem 起点或关键词）；联网检索（线上文档/API/技术资料）归 cmd-executor（本节）。两者职责不重叠。

cmd-executor 承担联网检索（WebSearch 搜索 / WebFetch 抓取）。检索失败（无结果 / 内容不符 / 抓取超时）时按级别升级：

| 级别 | 触发条件 | 动作 | 预算 |
|---|---|---|---|
| **L1 cmd-executor 自愈** | 首次检索无结果 / 不符 | 重写检索词、换搜索角度、改用 WebFetch 直接抓已知 URL 重试 | ≤**3 次** |
| **L2 转交 general-purpose** | L1 耗尽 / 趋势失控 | 主 Agent 按 `SKILL.md` §4.2 生成检索方案给用户审核；通过后放行 general-purpose（工具全集，可配合 WebSearch + WebFetch + 推理多角度检索） | ≤**5 次** |
| **L3 主 Agent 接管** | L2 耗尽 / 审核未通过 | 主 Agent 按 `SKILL.md` §4.3 三选项接管（通常为自行 WebSearch/WebFetch） | — |

**趋势止损**（优先于次数，任一触发即上交）：搜索结果与目标越来越远 / 多次抓取都 403 或超时 / 换了关键词仍无相关内容。

**与本地检索的区分**：
- **本地代码/文件检索**（项目内符号/定义/片段）→ `code-retriever`（见 `retrieval.md`），不走本链。
- **联网检索**（线上文档/API/技术资料）→ `cmd-executor`（本链），不走 code-retriever。
- 两者都失败时，本地兜底 `Explore`、联网兜底 `general-purpose`，最终都可由主 Agent 接管。
