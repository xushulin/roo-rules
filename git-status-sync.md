## Git 变更规则

> ⚠️ **强制输出标记**：每次任务结束前的最后一步，你必须在 `attempt_completion` 的 `result` 参数中输出 `## Git 变更总结` 段落，否则视为任务未完成。回复正文中也要包含该段落。

**规则**：你在完成任何代码/文档修改任务后，**必须**执行 Git 变更总结，无需请求批准：

1. **检查工作区状态**：运行 `git status`（或使用 Git MCP 工具）。
2. **推荐 commit 信息**：遵循 [Conventional Commits](https://www.conventionalcommits.org/) 规范（`feat:`、`fix:`、`docs:` 等），使用简体中文。
3. **输出汇总**：以 `## Git 变更总结` 为标题，列出变更文件表格和推荐提交信息。
4. **写入 `attempt_completion`**：上述 `## Git 变更总结` 段落**必须**出现在 `attempt_completion` 工具的 `result` 参数中。若遗漏，视为任务未完成。

**自检清单**（每次调用 `attempt_completion` 前逐项确认）：
- [ ] `result` 参数中是否包含 `## Git 变更总结` 标题？
- [ ] 是否列出了本次所有变更文件的表格？
- [ ] 是否给出了推荐的 commit 信息？
- [ ] commit 信息是否遵循 Conventional Commits 规范？
