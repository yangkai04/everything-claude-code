---
description: 使用专项代理对 Pull Request 进行全面审查
---

对一个 Pull Request 执行多视角综合审查。

## 用法

`/review-pr [PR编号或URL] [--focus=comments|tests|errors|types|code|simplify]`

若未指定 PR，则审查当前分支的 PR。若未指定 focus，则运行完整审查流程。

## 步骤

1. 确定 PR：
   - 使用 `gh pr view` 获取 PR 详情、变更文件和 diff
2. 查找项目指南：
   - 查找 `CLAUDE.md`、lint 配置、TypeScript 配置、仓库约定
3. 运行专项审查代理：
   - `code-reviewer`
   - `comment-analyzer`
   - `pr-test-analyzer`
   - `silent-failure-hunter`
   - `type-design-analyzer`
   - `code-simplifier`
4. 汇总结果：
   - 去重重叠发现
   - 按严重程度（severity）排序
5. 按严重程度分组输出发现

## 置信度规则（Confidence Rule）

只报告置信度 >= 80 的问题：

- 严重（Critical）：缺陷（bug）、安全漏洞、数据丢失
- 重要（Important）：缺少测试、质量问题、风格违规
- 建议（Advisory）：仅在明确请求时输出建议
