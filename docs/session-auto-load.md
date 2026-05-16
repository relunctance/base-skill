# 平台 Session 自动加载机制对比

> 横向对比 Claude Code / OpenClaw / Hermes 三平台的 session 启动时自动加载 skill 能力

## 快速对比

| 平台 | 自动加载机制 | 配置位置 | 难度 |
|------|------------|---------|------|
| **Claude Code** | SessionStart Hook + agent prompt | `settings.json` → `hooks.SessionStart` | ⭐ 简单 |
| **OpenClaw** | `agents.defaults.skills` | `config.yaml` → `agents.defaults.skills` | ⭐ 最简单 |
| **Hermes** | 无（需 workaround） | — | — |

---

## Claude Code

### 机制

通过 `settings.json` 的 `hooks.SessionStart` 配置，session 启动时自动执行 agent prompt。

### 配置方式

```json
"hooks": {
  "SessionStart": [{
    "matcher": "",
    "hooks": [{
      "type": "agent",
      "prompt": "执行 /your-skill-name"
    }]
  }]
}
```

### 效果

每次会话开始，自动向 agent 发送 `/your-skill-name` 命令，触发 skill 激活。

### 文档参考

Claude Code 官方文档：Session Hooks

---

## OpenClaw

### 机制

通过 `config.yaml` 的 `agents.defaults.skills` 配置，为所有会话固定加载指定的 skills。

### 配置方式

```yaml
agents:
  defaults:
    skills:
      - target-skill
      - github
      - weather
```

### 行为规则

| 场景 | 行为 |
|------|------|
| agent 未设置 skills | 继承 `agents.defaults.skills` |
| agent 设置了 `skills: ["xxx"]` | 替换 defaults（不是合并） |
| agent 设置了 `skills: []` | 强制禁用所有 skills |

### 文档参考

OpenClaw 官方配置文档

---

## Hermes

### 当前状态

**无内置机制**。session 启动时没有任何自动加载 skill 的配置入口。

### 现状分析

- SOUL.md 路由表只在用户说触发词时才激活 skill
- Gateway hooks 是 fire-and-forget，无法修改 system prompt
- `auto_skill` 机制仅支持 per-channel / per-topic 配置，无法全局

### 官方 Issue

已向 Hermes 官方提 issue，期望支持：
- Option A: `gateway.auto_skills` 配置（最简单，类似 OpenClaw）
- Option B: `auto_trigger: session_start` frontmatter
- Option C: `session:start` hook 注入能力扩展

Issue 地址：https://github.com/NousResearch/hermes-agent/issues/26709

---

## 当前 Workaround（Hermes SOUL 层）

### Workaround 1：直接嵌入目标对齐指令（已实施）

在 base-skill 第八步安装时，将目标对齐逻辑直接写入 SOUL.md：

```markdown
### 目标对齐（每次对话首次回复前强制）

无论用户说什么，**每次对话的第一条回复前**必须先运行：

```bash
python3 ${SKILL_DIR}/target-skill/scripts/target-state.py get
```

如果输出有进行中的目标，**必须**在回复开头先汇报目标状态。
```

**局限性**：
- 依赖 LLM 服从度（LLM 可能不听）
- 不是真正的 skill 激活，只是行为指令
- 不适合需要完整 skill 功能的场景（如 github skill 的各种命令）

### Workaround 2：触发词覆盖（辅助）

扩充 SOUL.md 路由表的触发词，降低用户触发门槛：

```markdown
| 好的、了解了、嗯、继续、开始... | target-skill |
```

**局限性**：仍然依赖用户说话，不适合「无用户输入也要执行」的场景。

---

## 平台通用设计建议

base-skill 安装时，如果检测到不同平台，应该：

| 平台 | 安装后操作 |
|------|---------|
| **Claude Code** | 提示用户在 `settings.json` 中添加 SessionStart Hook |
| **OpenClaw** | 自动写入 `agents.defaults.skills` 到 `config.yaml`（如果用户授权） |
| **Hermes** | 写入「目标对齐」指令到 SOUL.md（Workaround 1）；提示用户等待官方支持 |

---

## base-skill 改进方向

### 短期（已实施）

- 安装时向 SOUL.md 写入「首次回复前强制目标对齐」指令
- 扩充 target-skill 触发词路由

### 中期（等官方）

- Hermes 官方接受 `gateway.auto_skills` 后，base-skill 安装时自动写入配置
- 支持 `auto_trigger: session_start` 后，target-skill 自身声明即可

### 长期

- 三平台统一的「session 启动自动加载」配置抽象层
- base-skill 安装时检测平台，自动选择最优方式配置
