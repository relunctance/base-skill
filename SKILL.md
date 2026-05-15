---
name: base-skill
description: 当用户要求初始化开发环境、安装所有 skills、让 bot 智能起来时使用。引导 agent 自主识别当前平台，自动安装所有 14 个核心 skills（self-improving-agent / summarize-skill / darwin-skill / capability-evolver-skill / target-skill / honesty-skill / SkillForge / skill-created / find-skills / dir-skill / OpenSpec / task-split-skill / git-standards-skill / readme-skill），覆盖目标→规划→执行→纠错的完整闭环
version: "1.0.0"
author: relunctance
license: MIT
category: infrastructure
tags:
  - initialization
  - setup
  - skills
  - infrastructure
  - onboarding
metadata:
  hermes:
    platforms:
      claude_code: true
      openclaw: true
      hermes: true
      cursor: true
      vscode: true
    related_skills: [skill-created, find-skills, dir-skill]
---

# base-skill

> Agent 初始化套件 — 让任何 AI agent 快速具备完整开发能力

## 触发条件

用户说：
- `安装所有 skills`
- `初始化开发环境`
- `让我 bot 智能起来`
- `安装 base-skill`
- `初始化 skills`
- `setup all skills`

## 概述

本 skill 引导 agent 识别当前平台，自动安装 14 个核心 skills，覆盖：

| 能力维度 | Skills |
|---------|--------|
| **目标层** | target-skill（目标追踪） |
| **规划层** | task-split-skill（任务拆解）+ OpenSpec（开发规范） |
| **执行层** | SkillForge（skill 框架）+ dir-skill（项目结构） |
| **质量层** | honesty-skill（开发原则）+ capability-evolver（监控）+ self-improving-agent（纠错） |
| **知识层** | summarize-skill（内容提炼） |
| **技能层** | skill-created（skill 工厂）+ find-skills（技能发现）+ darwin-skill（技能进化） |

安装完成后，agent 具备：**目标追踪 → 任务拆解 → 规范开发 → 自我纠错 → 持续进化** 的完整闭环。

## 共享模块

`base-skill/scripts/platform_detect.py` 是跨平台路径检测的核心模块，所有 skill 均可复用：

```python
# 复用方式（在目标 skill 的脚本中）
import sys
from pathlib import Path
sys.path.insert(0, str(Path(__file__).resolve().parents[2] / "base-skill" / "scripts"))
from platform_detect import detect, skill_dir, config_dir, state_file

platform = detect()              # 'hermes' | 'openclaw' | 'claude_code' | 'codex' | 'cursor' | 'vscode' | 'unknown'
skill_dir('target-skill')        # skill 安装目录（平台相关）
config_dir('my-skill')           # skill 配置目录（平台相关，profile/workspace 隔离）
state_file('.my-state.json')     # 状态文件路径（profile/workspace 隔离）
```

| 函数 | 返回 | 说明 |
|------|------|------|
| `detect()` | `'hermes'` 等 | 检测当前平台 |
| `skill_dir(name)` | `Path` | skill 安装目录 |
| `config_dir(name)` | `Path` | skill 配置目录（存放状态文件） |
| `state_file(name)` | `Path` | 状态文件路径（自动隔离到 profile/workspace） |

## 安装目标清单

以下 14 个 skills 按顺序安装：

```
1.  https://github.com/peterskoett/self-improving-agent
2.  https://github.com/relunctance/summarize-skill
3.  https://github.com/alchaincyf/darwin-skill
4.  https://github.com/relunctance/capability-evolver-skill
5.  https://github.com/relunctance/target-skill
6.  https://github.com/relunctance/honesty-skill
7.  https://github.com/tripleyak/SkillForge
8.  https://github.com/relunctance/skill-created
9.  https://github.com/relunctance/find-skills
10. https://github.com/relunctance/dir-skill
11. https://github.com/Fission-AI/OpenSpec
12. https://github.com/relunctance/task-split-skill
13. https://github.com/relunctance/git-standards-skill
```

## 安装流程

> ⚠️ **Hermes 特有检查**：安装 skills 后必须验证 `external_dirs` 配置！

### 第一步：检测当前平台

```bash
# 检测 Hermes Agent
if command -v hermes &>/dev/null || [ -d "$HOME/.hermes/skills" ]; then
  PLATFORM="hermes"
elif [ -d "$HOME/.openclaw/skills" ]; then
  PLATFORM="openclaw"
elif command -v claude &>/dev/null; then
  PLATFORM="claude_code"
elif [ -d "$HOME/.cursor/rules" ] || [ -f "$HOME/.cursor/mcp.json" ]; then
  PLATFORM="cursor"
elif [ -d "$HOME/.windsurf" ]; then
  PLATFORM="windsurf"
else
  PLATFORM="unknown"
fi
echo "Detected platform: $PLATFORM"
```

### 第二步：识别 skills 安装目录

| 平台 | 安装目录 |
|------|---------|
| Hermes Agent | `~/.hermes/skills/` |
| OpenClaw | `~/.openclaw/skills/` |
| Claude Code | `~/.claude/skills/` |
| Cursor | `.cursor/rules/` 或项目内 |
| VSCode | `.vscode/.claude/rules/` |

```bash
SKILLS_DIR="$HOME/.hermes/skills"
case "$PLATFORM" in
  hermes)    SKILLS_DIR="$HOME/.hermes/skills" ;;
  openclaw)  SKILLS_DIR="$HOME/.openclaw/skills" ;;
  claude_code) SKILLS_DIR="$HOME/.claude/skills" ;;
  cursor)    SKILLS_DIR="$(pwd)/.cursor/rules" ;;
  *)         echo "Unsupported platform: $PLATFORM"; exit 1 ;;
esac
mkdir -p "$SKILLS_DIR"
```

### 第二步（Hermes 特有）：验证 external_dirs 配置

```bash
# Hermes 必须检查 external_dirs 是否包含 skills 目录
if [ "$PLATFORM" = "hermes" ]; then
  CONFIG="$HOME/.hermes/config.yaml"
  if [ -f "$CONFIG" ]; then
    # 检查 external_dirs 是否为空或缺失
    if grep -q "external_dirs:\s*\[\]" "$CONFIG" 2>/dev/null || ! grep -q "external_dirs:" "$CONFIG" 2>/dev/null; then
      echo "⚠️  external_dirs 为空，正在修复..."
      # 备份
      cp "$CONFIG" "$CONFIG.bak"
      # 修复：添加 external_dirs 指向 ~/.hermes/skills
      sed -i 's/external_dirs:\s*\[\]/external_dirs:\n    - \/home\/gql\/.hermes\/skills/' "$CONFIG"
      echo "✅ external_dirs 已修复"
      # 提示重启
      echo "🔄 请执行: systemctl --user restart hermes-gateway-<profile> 使配置生效"
    else
      echo "✅ external_dirs 配置正确"
    fi
  fi
fi
```

### 第三步：克隆安装所有 skills

```bash
SKILLS_DIR="~/.hermes/skills"
mkdir -p "$SKILLS_DIR"
cd "$SKILLS_DIR"

REPOS=(
  "https://github.com/peterskoett/self-improving-agent"
  "https://github.com/relunctance/summarize-skill"
  "https://github.com/alchaincyf/darwin-skill"
  "https://github.com/relunctance/capability-evolver-skill"
  "https://github.com/relunctance/target-skill"
  "https://github.com/relunctance/honesty-skill"
  "https://github.com/tripleyak/SkillForge"
  "https://github.com/relunctance/skill-created"
  "https://github.com/relunctance/find-skills"
  "https://github.com/relunctance/dir-skill"
  "https://github.com/Fission-AI/OpenSpec"
  "https://github.com/relunctance/task-split-skill"
  "https://github.com/relunctance/git-standards-skill"
  "https://github.com/relunctance/readme-skill"
)

for repo in "${REPOS[@]}"; do
  name=$(basename "$repo")
  if [ -d "$name" ]; then
    echo "⏭️  $name 已存在，跳过"
  else
    echo "📦 安装 $name ..."
    git clone "$repo" "$name"
    echo "✅ $name 安装完成"
  fi
done
```

### 第五步：OpenClaw 特殊处理

OpenClaw 的 gateway 从 `~/.openclaw/extensions/<name>/dist/` 加载，skill 文件需软链或复制到正确位置：

```bash
if [ "$PLATFORM" = "openclaw" ]; then
  mkdir -p ~/.openclaw/extensions
  for skill in "$SKILLS_DIR"/*/; do
    name=$(basename "$skill")
    # skill 入口文件软链到 extensions 目录
    mkdir -p "~/.openclaw/extensions/$name/dist"
    if [ -f "$skill/SKILL.md" ]; then
      cp "$skill/SKILL.md" "~/.openclaw/extensions/$name/dist/"
    fi
  done
  echo "🔄 重启 openclaw-gateway 使配置生效"
  systemctl --user restart openclaw-gateway
fi
```

### 第六步：验证安装

```bash
echo "=== 验证安装 ==="
for skill in "$SKILLS_DIR"/*/; do
  name=$(basename "$skill")
  # OpenSpec 没有 SKILL.md，是工具集不是 Hermes skill，跳过检查
  if [ "$name" = "OpenSpec" ]; then
    echo "⚠️  $name（工具集，无 SKILL.md — 正常）"
    continue
  fi
  if [ -f "$skill/SKILL.md" ]; then
    echo "✅ $name"
  else
    echo "❌ $name（缺少 SKILL.md）"
  fi
done
echo "共安装 $(ls -d $SKILLS_DIR/*/ 2>/dev/null | wc -l) 个 skills"
```

### 第七步（Hermes 特有）：验证 external_dirs 生效

```bash
if [ "$PLATFORM" = "hermes" ]; then
  CONFIG="$HOME/.hermes/config.yaml"
  if grep -q "external_dirs:\s*\[\]" "$CONFIG" 2>/dev/null; then
    echo "❌ external_dirs 为空，skills 不会被加载！"
    echo "   请修复 $CONFIG 中的 external_dirs 配置"
    exit 1
  else
    echo "✅ external_dirs 配置正确"
  fi
fi
```

### 第八步：写入 honesty 规则 + Skill 路由表到 SOUL.md

安装完 skills 后，将 honesty 规则和 Skill 路由表追加到 SOUL.md（避免重复追加）：

```bash
HONESTY_RULES='
# 禁止瞎编（honesty-skill 核心准则）

## 核心准则

- **不知道**：直接说「我不清楚」，绝不捏造角色/组织/工作流/路径/配置
- **不瞎编**：涉及身份/角色/组织/工作流时，必须先查证，不确定则说不知道
- **不臆测**：结论必须有依据，无依据不臆测，不打包票
- **标注前提**：不确定时主动标注前提、风险，不说「绝对/保证/100%」
- **编造禁止**：禁止编造事实/代码/参数/文档/出处/版本号/路径

## 自问清单（每次回答前）

在开口之前，问自己：
1. 我真的知道这个吗？
2. 我有没有查证过？
3. 如果我不确定，我是否说了「我不清楚，需要先确认」？

## 子 agent 委托约束

委托子 agent 时，必须在 context 中携带：
```
[强制约束] 禁止瞎编：不知道就说不知道，涉及未核实的信息必须标注前提。
```
'

SKILL_ROUTE_TABLE='
## Skill 路由（每次任务前必查）

收到用户任务时，先对照这张表，决定是否需要加载对应 skill。

| 用户说/诉求 | 应加载的 skill |
|------------|--------------|
| 设定目标、追踪目标、目标跑偏了 | target-skill |
| 拆解任务、分解步骤 | task-split-skill |
| 创建新的 skill | skill-created |
| 检索已有 skill、该用哪个 skill | find-skills |
| 总结内容、提炼要点 | summarize-skill |
| 初始化项目结构 | dir-skill |
| 优化/进化现有 skill | darwin-skill |
| 检查 README、改进文档 | readme-skill |
| Git 规范、commit message | git-standards-skill |
| 不知道该用什么 skill | find-skills |
| 开发原则、避免瞎编 | honesty-skill |
| 分析日志、诊断错误、健康评分 | capability-evolver-skill |

**执行流程**：收到任务 → 对照路由表 → 决定是否 load skill → 再执行
'

# 写入 SOUL.md（平台相关路径）
if [ "$PLATFORM" = "hermes" ]; then
  SOUL_DIR="$HOME/.hermes/profiles/baijie"
elif [ "$PLATFORM" = "openclaw" ]; then
  SOUL_DIR="$HOME/.openclaw/workspace"
else
  SOUL_DIR="$HOME"
fi

SOUL_FILE="$SOUL_DIR/SOUL.md"

# 检查是否已有 honesty 核心准则节
if [ -f "$SOUL_FILE" ] && grep -q "禁止瞎编" "$SOUL_FILE"; then
  echo "⏭️  SOUL.md 已包含 honesty 规则，跳过"
else
  echo "$HONESTY_RULES" >> "$SOUL_FILE"
  echo "✅ honesty 规则已写入 $SOUL_FILE"
fi

# 检查是否已有 Skill 路由节
if [ -f "$SOUL_FILE" ] && grep -q "Skill 路由" "$SOUL_FILE"; then
  echo "⏭️  SOUL.md 已包含 Skill 路由表，跳过"
else
  echo "$SKILL_ROUTE_TABLE" >> "$SOUL_FILE"
  echo "✅ Skill 路由表已写入 $SOUL_FILE"
fi
```

## 各平台详细安装说明

### Hermes Agent

```bash
SKILLS_DIR="$HOME/.hermes/skills"
mkdir -p "$SKILLS_DIR"
cd "$SKILLS_DIR"
git clone https://github.com/peterskoett/self-improving-agent
git clone https://github.com/relunctance/summarize-skill
# ... 其余 repos
```

### OpenClaw

```bash
SKILLS_DIR="$HOME/.openclaw/skills"
mkdir -p "$SKILLS_DIR"
cd "$SKILLS_DIR"
git clone https://github.com/peterskoett/self-improving-agent
# ... 其余 repos

# 软链到 extensions
for skill in */; do
  mkdir -p "$HOME/.openclaw/extensions/${skill%/}/dist"
  cp "$skill/SKILL.md" "$HOME/.openclaw/extensions/${skill%/}/dist/"
done
systemctl --user restart openclaw-gateway
```

### Claude Code

```bash
# Claude Code 使用 /plugin 命令
/plugin marketplace add https://github.com/peterskoett/self-improving-agent.git
/plugin marketplace add https://github.com/relunctance/summarize-skill.git
# ... 其余 repos
```

### Cursor

```bash
# 复制 SKILL.md 到项目 .cursor/rules/ 目录
mkdir -p .cursor/rules
git clone --depth 1 https://github.com/peterskoett/self-improving-agent /tmp/self-improving-agent
cp /tmp/self-improving-agent/SKILL.md .cursor/rules/
# ... 其余 repos
```

## 安装后验证

安装完成后，确认以下能力已就绪：

- [ ] `target-skill` — 目标追踪可用
- [ ] `task-split-skill` — 任务拆解可用
- [ ] `OpenSpec` — 开发规范可用（`/opsx:propose` 等命令）
- [ ] `honesty-skill` — 开发原则可用
- [ ] `capability-evolver-skill` — 日志分析可用
- [ ] `self-improving-agent` — 出错记录可用
- [ ] `summarize-skill` — 内容总结可用
- [ ] `skill-created` — skill 工厂可用
- [ ] `find-skills` — 技能检索可用
- [ ] `dir-skill` — 项目结构可用
- [ ] `darwin-skill` — 技能进化可用
- [ ] `SkillForge` — skill 框架可用

## 踩坑记录

| 坑 | 说明 | 解决方案 |
|----|------|---------|
| OpenClaw gateway 不加载 skill | 路径不对 | skill 文件必须放在 `~/.openclaw/extensions/<name>/dist/` |
| OpenClaw 需要重启 | 修改 skill 后不生效 | `systemctl --user restart openclaw-gateway` |
| git clone 超时 | 国内网络访问 GitHub 慢 | 配置代理 `git config --global http.proxy http://192.168.1.109:10808` |
| 重复安装 | 目录已存在 | 安装前检查 `[ -d "$SKILLS_DIR/$name" ]`，存在则跳过 |
| 子模块 clone 失败 | OpenSpec 有子模块 | `git clone --recursive https://github.com/Fission-AI/OpenSpec` |
| Hermès skill 路径错误 | 用了错误的 ~ 路径 | `SKILLS_DIR="$HOME/.hermes/skills"` 而非 `~./hermes/skills` |
