---
name: safety-guard
description: 在生产系统上作业或以自主模式运行 Agent 时，使用此技能防止破坏性操作。
origin: ECC
---

# Safety Guard（安全守卫）— 防止破坏性操作

## 使用场景

- 在生产系统上作业时
- Agent 以自主模式运行（全自动模式）时
- 希望将编辑限制在特定目录时
- 执行敏感操作（迁移、部署、数据变更）时

## 工作原理

三种保护模式：

### 模式 1：谨慎模式（Careful Mode）

在执行破坏性命令之前进行拦截并发出警告：

```
监控的命令模式：
- rm -rf（尤其是 /、~ 或项目根目录）
- git push --force
- git reset --hard
- git checkout .（丢弃所有更改）
- DROP TABLE / DROP DATABASE
- docker system prune
- kubectl delete
- chmod 777
- sudo rm
- npm publish（意外发布）
- 任何带有 --no-verify 的命令
```

检测到上述命令时：展示该命令的作用，请求确认，并建议更安全的替代方案。

### 模式 2：冻结模式（Freeze Mode）

将文件编辑锁定在特定目录树内：

```
/safety-guard freeze src/components/
```

任何在 `src/components/` 之外的写入（Write）或编辑（Edit）操作都会被拦截并附上说明。适合需要 Agent 专注于某一区域、不触碰其他代码的场景。

### 模式 3：守卫模式（Guard Mode，谨慎 + 冻结组合）

两种保护同时启用，为自主 Agent 提供最高级别的安全保障。

```
/safety-guard guard --dir src/api/ --allow-read-all
```

Agent 可以读取任何内容，但只能写入 `src/api/`。破坏性命令在任何地方都会被拦截。

### 解除保护

```
/safety-guard off
```

## 实现原理

使用 PreToolUse 钩子拦截 Bash、Write、Edit 和 MultiEdit 工具调用，在允许执行前对命令/路径进行规则检查。

## 集成说明

- 默认为 `codex -a never` 会话启用
- 与 ECC 2.0 中的可观测性风险评分配合使用
- 将所有被拦截的操作记录到 `~/.claude/safety-guard.log`
