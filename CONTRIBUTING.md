# 贡献指南

感谢你对 subagent-orchestrator 的关注！欢迎通过以下方式参与贡献。

## 🐛 报告问题

- 提交 Issue 描述问题或建议，附上复现步骤与期望行为。
- Skill 行为异常时，请贴出主 Agent 的调度日志（子 Agent 名称 + 模型名）。

## 🔧 提交修改

1. Fork 本仓库并新建分支：`git checkout -b feature/your-feature`
2. 保持改动聚焦，一个 PR 解决一件事。
3. **Skill 文件（`SKILL.md` / `agents.md` 等）改动**：确保元数据 `name` 与目录名一致；涉及子 Agent 配置时同步更新 `agents.md` 与对应 `retrieval.md`/`execution.md`/`command.md`。
4. 提交信息遵循 Conventional Commits（如 `feat: ...`、`fix: ...`、`docs: ...`）。
5. 提交 PR 并说明动机与测试方式。

## ✅ 自检清单

提交 PR 前请确认：
- [ ] `SKILL.md` 的 YAML frontmatter 格式正确（`name` / `description`）
- [ ] 引用的子 Agent 名称与 `agents.md` 一致（区分大小写）
- [ ] 如改了子 Agent 工具列表，`agents.md` 与 `command.md`/`execution.md`/`retrieval.md` 已同步
- [ ] 文档链接有效

## 📜 行为准则

请保持友善、尊重，聚焦技术讨论。
