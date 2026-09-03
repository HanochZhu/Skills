# Skills

可在 Cursor、Claude Code、Codex 等支持 [Agent Skills](https://agentskills.io) 标准的工具中复用的技能集合。

每个技能是一个目录，内含必需的 `SKILL.md`（YAML frontmatter + Markdown 说明）。

英文说明见 [README.md](./README.md)。

## 仓库内容

| 技能 | 说明 |
|------|------|
| [`explore-to-doc`](./explore-to-doc/) | 先读 `docs/index.md` 再决定是否探索；探索写入 `docs/explore/<topic>/`；设计/架构写入 `docs/design/`；计划写入 `docs/tasks/`；临时 review 写入 `docs/tmp/` |
| [`memwalker`](./memwalker/) | MemWalker 交互式阅读：对长文/代码建摘要树并带推理导航，突破单次上下文限制 |
| [`sandbox-fetch-optimize`](./sandbox-fetch-optimize/) | 沙盒优先的拉数优化：先量 baseline，再对比多方案，对比有收益才落地 |
| [`web-design`](./web-design/) | 网页视觉设计：从 PRD / URL / 截图 / 关键词先产出 `DESIGN.md`，确认后再生成 UI 代码。上游：[xiaopu-ai/web-design](https://github.com/xiaopu-ai/web-design) |

## 目录结构

```text
Skills/
├── README.md
├── README-cn.md
├── explore-to-doc/
│   └── SKILL.md
├── memwalker/
│   ├── SKILL.md
│   └── reference.md
├── sandbox-fetch-optimize/
│   ├── SKILL.md
│   └── reference.md
└── web-design/
    ├── SKILL.md
    ├── LICENSE
    ├── references/
    └── scripts/
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
cp -R explore-to-doc memwalker sandbox-fetch-optimize web-design ~/.claude/skills/
```

### 安装示例（项目）

```bash
mkdir -p .claude/skills
cp -R explore-to-doc memwalker sandbox-fetch-optimize web-design .claude/skills/
```

### 使用

- 自动：任务描述匹配 `description` 时由 Claude 选用
- 手动：在对话中输入 `/skill-name`（如 `/memwalker`）
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
cp -R explore-to-doc memwalker sandbox-fetch-optimize web-design ~/.codex/skills/

# 或安装到 Agent Skills 个人目录
mkdir -p ~/.agents/skills
cp -R explore-to-doc memwalker sandbox-fetch-optimize web-design ~/.agents/skills/
```

安装后**重启 Codex**（或新开会话），用 `/skills` 确认是否出现。

### 从 GitHub 安装（可选）

在 Codex 会话中使用内置 `$skill-installer`，指向本仓库中某个技能目录，例如：

```text
$skill-installer install https://github.com/HanochZhu/Skills/tree/main/memwalker
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
cp -R explore-to-doc memwalker sandbox-fetch-optimize web-design ~/.cursor/skills/
```

或项目级：

```bash
mkdir -p .cursor/skills
cp -R explore-to-doc memwalker sandbox-fetch-optimize web-design .cursor/skills/
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
| `explore-to-doc` | 先读 `docs/index.md`；探索写入 `docs/explore/<topic>/`；设计/架构写入 `docs/design/`；计划写入 `docs/tasks/`；临时 review 写入 `docs/tmp/`（不入索引） | 行为相同；workspace 级文档勿写入各子仓库自己的 `docs/` |
| `memwalker` | 构建/导航摘要树（可选落盘 `.memwalker/`） | 各工具行为相同 |
| `sandbox-fetch-optimize` | 沙盒对比拉数成本，score 赢了才落地 | 各工具行为相同 |
| `web-design` | 同样先出 `DESIGN.md` 再出代码；抓取参考站时用自带 Python / Playwright 脚本 | 各工具行为相同 |

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
