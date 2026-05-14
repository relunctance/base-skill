# base-skill

> Agent 初始化套件 — 让任何 AI agent 快速具备完整开发能力

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GitHub stars](https://img.shields.io/github/stars/relunctance/base-skill)](https://github.com/relunctance/base-skill/stargazers)

## 触发条件

- `安装所有 skills`
- `初始化开发环境`
- `让我 bot 智能起来`
- `安装 base-skill`
- `初始化 skills`

## 功能特性

- **自动平台检测** — 识别 Hermes / OpenClaw / Claude Code / Cursor / VSCode
- **一键安装** — 同时安装 14 个核心 skills
- **完整能力闭环** — 覆盖目标→规划→执行→纠错→进化的每个环节

## 安装的 14 个核心 Skills

| 维度 | Skills |
|------|--------|
| 目标层 | target-skill |
| 规划层 | task-split-skill + OpenSpec |
| 执行层 | SkillForge + dir-skill |
| 质量层 | honesty-skill + capability-evolver + self-improving-agent |
| 知识层 | summarize-skill |
| 技能层 | skill-created + find-skills + darwin-skill + git-standards-skill + readme-skill |

## 支持的平台

| 平台 | 安装目录 |
|------|---------|
| Hermes Agent | `~/.hermes/skills/` |
| OpenClaw | `~/.openclaw/skills/` |
| Claude Code | `~/.claude/skills/` |
| Cursor | `.cursor/rules/` |
| VSCode | `.vscode/.claude/rules/` |

## 安装后验证

- [ ] target-skill — 目标追踪
- [ ] task-split-skill — 任务拆解
- [ ] OpenSpec — 开发规范
- [ ] honesty-skill — 开发原则
- [ ] capability-evolver-skill — 日志分析
- [ ] self-improving-agent — 出错记录
- [ ] summarize-skill — 内容总结
- [ ] skill-created — skill 工厂
- [ ] find-skills — 技能检索
- [ ] dir-skill — 项目结构
- [ ] darwin-skill — 技能进化
- [ ] SkillForge — skill 框架
- [ ] git-standards-skill — Git 规范
- [ ] readme-skill — README 美化
- [ ]  — 提交前质检

## 相关 Skills

- [skill-created](https://github.com/relunctance/skill-created) — 单个 skill 创建工厂
- [find-skills](https://github.com/relunctance/find-skills) — 全平台 skill 检索
- [dir-skill](https://github.com/relunctance/dir-skill) — 项目结构生成

## 许可证

MIT