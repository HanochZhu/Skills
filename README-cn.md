# Skills

可在 Cursor、Claude Code、Codex 等支持 [Agent Skills](https://agentskills.io) 标准的工具中复用的技能集合。

每个技能是一个目录，内含必需的 `SKILL.md`（YAML frontmatter + Markdown 说明）。

英文说明见 [README.md](./README.md)。

## 仓库内容

| 技能 | 说明 |
|------|------|
| [`auto-summary-context`](./auto-summary-context/) | 新问题开始时检查 context 用量；≥70% 时先压缩对话历史再继续 |
| [`memory-skill`](./memory-skill/) | 对话结束后整理关键词总结到 `talk_summary.md`，降低后续 token 消耗 |

> **注意**：`auto-summary-context` 原文依赖 Cursor 的 `/summarize`。在 Claude Code / Codex 中请改用对应工具的压缩/总结命令（见下方「工具差异」）。

## 目录结构

```text
Skills/
├── README.md
├── README-cn.md
├── auto-summary-context/
│   └── SKILL.md
└── memory-skill/
    └── SKILL.md
```

## 快速开始

```bash
git clone git@github.com:HanochZhu/Skills.git
cd Skills
```

然后按目标工具把技能目录放到对应路径（见下）。

---

## Claude Code

Claude Code 会自动扫描技能目录；放好后重启会话或开新会话即可。

| 范围 | 路径 | 适用 |
|------|------|------|
| 个人（全局） | `~/.claude/skills/<skill-name>/SKILL.md` | 所有项目 |
| 项目 | `.claude/skills/<skill-name>/SKILL.md` | 当前仓库 |

### 安装示例（个人）

```bash
mkdir -p ~/.claude/skills
cp -R auto-summary-context memory-skill ~/.claude/skills/
```

### 安装示例（项目）

```bash
mkdir -p .claude/skills
cp -R auto-summary-context memory-skill .claude/skills/
```

### 使用

- 自动：任务描述匹配 `description` 时由 Claude 选用
- 手动：在对话中输入 `/skill-name`（如 `/memory-skill`）
- 排查：`/doctor` 可检查技能是否被加载或描述被截断

官方文档：[Claude Code Skills](https://code.claude.com/docs/en/skills)

---

## Codex（OpenAI Codex CLI）

Codex 同样识别含 `SKILL.md` 的技能目录。

| 范围 | 路径 | 适用 |
|------|------|------|
| 个人（常见） | `~/.codex/skills/<skill-name>/SKILL.md` | 当前用户 |
| 个人（Agent 约定） | `~/.agents/skills/<skill-name>/SKILL.md` | 跨项目 |
| 项目 | `.agents/skills/<skill-name>/SKILL.md` | 当前仓库 |

### 手动安装（推荐）

```bash
# 安装到 Codex 用户技能目录
mkdir -p ~/.codex/skills
cp -R auto-summary-context memory-skill ~/.codex/skills/

# 或安装到 Agent Skills 个人目录
mkdir -p ~/.agents/skills
cp -R auto-summary-context memory-skill ~/.agents/skills/
```

安装后**重启 Codex**（或新开会话），用 `/skills` 确认是否出现。

### 从 GitHub 安装（可选）

在 Codex 会话中使用内置 `$skill-installer`，指向本仓库中某个技能目录，例如：

```text
$skill-installer install https://github.com/HanochZhu/Skills/tree/main/memory-skill
```

（需仓库可访问。）

### 使用

- `/skills`：浏览已安装技能
- `$skill-name` 或 `/use skill-name`：显式加载
- 描述匹配时也可隐式触发

---

## Cursor

| 范围 | 路径 | 适用 |
|------|------|------|
| 个人 | `~/.cursor/skills/<skill-name>/SKILL.md` | 所有项目 |
| 项目 | `.cursor/skills/<skill-name>/SKILL.md` | 当前仓库 |

> 不要写入 `~/.cursor/skills-cursor/`（Cursor 内置技能目录）。

### 安装示例

```bash
mkdir -p ~/.cursor/skills
cp -R auto-summary-context memory-skill ~/.cursor/skills/
```

或项目级：

```bash
mkdir -p .cursor/skills
cp -R auto-summary-context memory-skill .cursor/skills/
```

Cursor 官方约定为 `.cursor/skills/`（复数）。本仓库为面向多工具的统一分发结构。

---

## 一键安装脚本示例

将仓库根目录下所有技能装到 Claude Code、Codex、Cursor 个人目录：

```bash
#!/usr/bin/env bash
set -euo pipefail
ROOT="$(cd "$(dirname "$0")" && pwd)"

for dest in \
  "$HOME/.claude/skills" \
  "$HOME/.codex/skills" \
  "$HOME/.cursor/skills"
do
  mkdir -p "$dest"
  for skill in "$ROOT"/*/SKILL.md; do
    name="$(basename "$(dirname "$skill")")"
    rm -rf "$dest/$name"
    cp -R "$ROOT/$name" "$dest/$name"
    echo "Installed $name -> $dest/$name"
  done
done
```

保存为 `install.sh` 后执行：`chmod +x install.sh && ./install.sh`。

---

## 工具差异（重要）

| 技能 | Cursor | Claude Code / Codex |
|------|--------|---------------------|
| `auto-summary-context` | 可直接用 `/summarize` | 无 `/summarize` 时，改用工具自带的 compact / summarize / 开新会话等等价能力；可按需改写 `SKILL.md` 中的命令名 |
| `memory-skill` | 写入工作区 `talk_summary.md` | 行为相同；注意总结文件路径是否符合团队约定 |

## 校验

安装后确认：

```bash
# Claude Code
ls ~/.claude/skills/*/SKILL.md

# Codex
ls ~/.codex/skills/*/SKILL.md

# Cursor
ls ~/.cursor/skills/*/SKILL.md
```

每个技能目录必须包含名为 **`SKILL.md`**（大小写敏感）的文件，且 frontmatter 含 `name` 与 `description`。

## 来源

导出供多工具复用。按需修改后请保持 `SKILL.md` 的 Agent Skills 兼容格式。
