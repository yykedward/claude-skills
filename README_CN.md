# Claude Code Skills

一组增强 Claude Code 开发工作流的技能。

## 开发流程

| Skill | 用途 |
|-------|------|
| `superpowers:brainstorming` | 创造性工作前的需求探索与方案设计 |
| `superpowers:writing-plans` | 为多步骤任务编写结构化实现计划 |
| `superpowers:executing-plans` | 在独立会话中按审查点执行实现计划 |
| `superpowers:test-driven-development` | 实现功能或修复 Bug 前强制先编写测试 |
| `superpowers:subagent-driven-development` | 使用子智能体并行执行实现计划中的独立任务 |
| `superpowers:dispatching-parallel-agents` | 并行处理多个无依赖关系的独立任务 |

## 质量与审查

| Skill | 用途 |
|-------|------|
| `review` | 审查 Pull Request 的代码变更 |
| `security-review` | 对当前分支的变更进行安全审查 |
| `superpowers:requesting-code-review` | 完成任务或主要功能后请求代码审查 |
| `superpowers:receiving-code-review` | 处理代码审查反馈，强调技术严谨和验证 |
| `superpowers:verification-before-completion` | 声称完成前运行验证命令并提供证据 |
| `simplify` | 审查代码的复用性、质量和效率并修复问题 |

## 调试与决策

| Skill | 用途 |
|-------|------|
| `superpowers:systematic-debugging` | 发现 Bug 或测试失败时先诊断根因再修复 |
| `agent-debate` | 设计决策有多种可行方案时启动结构化辩论并裁决 |

## 工作流与配置

| Skill | 用途 |
|-------|------|
| `init` | 初始化 CLAUDE.md 作为代码库文档 |
| `update-config` | 配置 Claude Code 设置、Hook、权限和环境变量 |
| `keybindings-help` | 自定义键盘快捷键和组合键绑定 |
| `fewer-permission-prompts` | 扫描历史记录生成白名单以减少权限提示 |
| `schedule` | 创建定时或一次性的远程智能体任务 |
| `loop` | 定时循环运行某个提示词或命令 |
| `superpowers:using-git-worktrees` | 为特性开发创建隔离的 Git 工作树 |
| `superpowers:finishing-a-development-branch` | 开发完成后提供合并、PR 或清理的选项 |

## API 与扩展

| Skill | 用途 |
|-------|------|
| `claude-api` | 构建、调试和优化 Claude API / Anthropic SDK 应用 |
| `superpowers:writing-skills` | 创建、编辑和验证技能文件 |

## 入门

| Skill | 用途 |
|-------|------|
| `superpowers:using-superpowers` | 技能系统入口：如何查找和使用技能 |

## 使用方式

在 Claude Code 中输入 `/<skill-name>` 调用对应技能。
