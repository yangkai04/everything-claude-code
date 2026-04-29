# 代码审查标准（Code Review Standards）

## 目的

代码审查（Code Review）在代码合并前确保质量、安全性和可维护性。本规则定义了何时以及如何进行代码审查。

## 何时审查

**必须触发审查的情形：**

- 编写或修改代码后
- 提交到共享分支前
- 修改安全敏感代码时（认证、支付、用户数据）
- 进行架构变更时
- 合并 Pull Request 前

**审查前置要求：**

在请求审查前，请确认：

- 所有自动化检查（CI/CD）均已通过
- 合并冲突（Merge conflicts）已解决
- 分支已与目标分支保持同步

## 审查清单（Review Checklist）

在标记代码完成前：

- [ ] 代码可读性好，命名清晰
- [ ] 函数职责单一（< 50 行）
- [ ] 文件内聚性强（< 800 行）
- [ ] 无深层嵌套（> 4 层）
- [ ] 错误已显式处理
- [ ] 无硬编码密钥或凭证
- [ ] 无 `console.log` 或调试语句
- [ ] 新功能有对应测试
- [ ] 测试覆盖率达到 80% 最低要求

## 安全审查触发条件

**遇到以下情形，停下来并使用 security-reviewer 代理：**

- 认证（Authentication）或授权（Authorization）代码
- 用户输入处理
- 数据库查询
- 文件系统操作
- 外部 API 调用
- 加密（Cryptographic）操作
- 支付或金融代码

## 审查严重程度（Severity Levels）

| 级别 | 含义 | 处理方式 |
|-------|---------|--------|
| CRITICAL（严重） | 安全漏洞或数据丢失风险 | **阻断（BLOCK）** — 必须在合并前修复 |
| HIGH（高） | 缺陷或重大质量问题 | **警告（WARN）** — 建议在合并前修复 |
| MEDIUM（中） | 可维护性问题 | **提示（INFO）** — 考虑修复 |
| LOW（低） | 风格或次要建议 | **备注（NOTE）** — 可选 |

## 代理（Agent）使用

使用以下代理进行代码审查：

| 代理 | 用途 |
|-------|---------|
| **code-reviewer** | 通用代码质量、模式、最佳实践 |
| **security-reviewer** | 安全漏洞、OWASP Top 10 |
| **typescript-reviewer** | TypeScript/JavaScript 专项问题 |
| **python-reviewer** | Python 专项问题 |
| **go-reviewer** | Go 专项问题 |
| **rust-reviewer** | Rust 专项问题 |

## 审查工作流

```
1. 运行 git diff 了解变更内容
2. 先检查安全清单
3. 审查代码质量清单
4. 运行相关测试
5. 验证覆盖率 >= 80%
6. 使用对应代理进行详细审查
```

## 常见问题清单

### 安全

- 硬编码凭证（API keys、密码、token）
- SQL 注入（查询中字符串拼接）
- XSS 漏洞（未转义的用户输入）
- 路径穿越（未净化的文件路径）
- 缺少 CSRF 防护
- 认证绕过

### 代码质量

- 函数过大（> 50 行）— 拆分为更小的函数
- 文件过大（> 800 行）— 提取模块
- 深层嵌套（> 4 层）— 使用提前返回（early return）
- 缺少错误处理 — 显式处理
- 可变数据模式 — 优先使用不可变操作
- 缺少测试 — 补充测试覆盖

### 性能

- N+1 查询 — 使用 JOIN 或批量操作
- 缺少分页 — 为查询添加 LIMIT
- 无界查询 — 添加约束条件
- 缺少缓存 — 对耗时操作添加缓存

## 批准标准（Approval Criteria）

- **批准（Approve）**：无 CRITICAL 或 HIGH 问题
- **警告（Warning）**：仅有 HIGH 问题（谨慎合并）
- **阻断（Block）**：发现 CRITICAL 问题

## 与其他规则的关联

本规则配合以下规则使用：

- [testing.md](testing.md) — 测试覆盖率要求
- [security.md](security.md) — 安全检查清单
- [git-workflow.md](git-workflow.md) — 提交规范
- [agents.md](agents.md) — 代理委派
