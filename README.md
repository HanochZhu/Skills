# Skills

A collection of reusable Agent Skills for tools that support the [Agent Skills](https://agentskills.io) standard, including Cursor, Claude Code, and Codex.

Each skill is a directory with a required `SKILL.md` file (YAML frontmatter + Markdown instructions).

中文说明见 [README-cn.md](./README-cn.md)。

## Contents

| Skill | Description |
|-------|-------------|
| [`explore-to-doc`](./explore-to-doc/) | Read `docs/index.md` before exploring; exploration docs under `docs/explore/<topic>/`; design/architecture under `docs/design/`; plans under `docs/tasks/`; temporary reviews under `docs/tmp/` |
| [`memwalker`](./memwalker/) | MemWalker interactive reading: summary tree + reasoned navigation for long docs/code beyond one context pass |
| [`review-fix`](./review-fix/) | Verify review findings against current code, reproduce each confirmed issue with a failing test, then fix only after reproduction succeeds |
| [`sandbox-fetch-optimize`](./sandbox-fetch-optimize/) | Sandbox-first data-fetch optimization: measure baseline cost, compare multiple schemes, adopt only when the comparison wins |
| [`web-design`](./web-design/) | Spec-first web visual design: produce `DESIGN.md` from PRD / URL / screenshot / keywords, then generate UI code. Upstream: [xiaopu-ai/web-design](https://github.com/xiaopu-ai/web-design) |

## Layout

```text
Skills/
├── README.md
├── README-cn.md
├── explore-to-doc/
│   └── SKILL.md
├── memwalker/
│   ├── SKILL.md
│   └── reference.md
├── review-fix/
│   └── SKILL.md
├── sandbox-fetch-optimize/
│   ├── SKILL.md
│   └── reference.md
└── web-design/
    ├── SKILL.md
    ├── LICENSE
    ├── references/
    └── scripts/
```

## Quick start

```bash
git clone git@github.com:HanochZhu/Skills.git
cd Skills
```

Then copy each skill directory into the path expected by your tool (below).

---

## Claude Code

Claude Code scans skill directories automatically. Restart or start a new session after installing.

| Scope | Path | Applies to |
|-------|------|------------|
| Personal (global) | `~/.claude/skills/<skill-name>/SKILL.md` | All projects |
| Project | `.claude/skills/<skill-name>/SKILL.md` | Current repo |

### Personal install

```bash
mkdir -p ~/.claude/skills
cp -R explore-to-doc memwalker review-fix sandbox-fetch-optimize web-design ~/.claude/skills/
```

### Project install

```bash
mkdir -p .claude/skills
cp -R explore-to-doc memwalker review-fix sandbox-fetch-optimize web-design .claude/skills/
```

### Usage

- Automatic: Claude picks a skill when the task matches its `description`
- Manual: type `/skill-name` (e.g. `/memwalker`)
- Diagnose: `/doctor` to check whether skills load or descriptions are truncated

Docs: [Claude Code Skills](https://code.claude.com/docs/en/skills)

---

## Codex (OpenAI Codex CLI)

Codex also discovers directories that contain `SKILL.md`.

| Scope | Path | Applies to |
|-------|------|------------|
| Personal (common) | `~/.codex/skills/<skill-name>/SKILL.md` | Current user |
| Personal (Agent convention) | `~/.agents/skills/<skill-name>/SKILL.md` | Cross-project |
| Project | `.agents/skills/<skill-name>/SKILL.md` | Current repo |

### Manual install (recommended)

```bash
# Codex user skills
mkdir -p ~/.codex/skills
cp -R explore-to-doc memwalker review-fix sandbox-fetch-optimize web-design ~/.codex/skills/

# Or Agent Skills personal directory
mkdir -p ~/.agents/skills
cp -R explore-to-doc memwalker review-fix sandbox-fetch-optimize web-design ~/.agents/skills/
```

**Restart Codex** (or open a new session), then confirm with `/skills`.

### Install from GitHub (optional)

In a Codex session, use the built-in `$skill-installer` pointed at a skill directory in this repo:

```text
$skill-installer install https://github.com/HanochZhu/Skills/tree/main/memwalker
```

(Requires the repo to be public/accessible.)

### Usage

- `/skills` — list installed skills
- `$skill-name` or `/use skill-name` — load explicitly
- Implicit trigger when the task matches the skill `description`

---

## Cursor

| Scope | Path | Applies to |
|-------|------|------------|
| Personal | `~/.cursor/skills/<skill-name>/SKILL.md` | All projects |
| Project | `.cursor/skills/<skill-name>/SKILL.md` | Current repo |

> Do not write into `~/.cursor/skills-cursor/` (Cursor’s built-in skills directory).

### Install

```bash
mkdir -p ~/.cursor/skills
cp -R explore-to-doc memwalker review-fix sandbox-fetch-optimize web-design ~/.cursor/skills/
```

Project-scoped:

```bash
mkdir -p .cursor/skills
cp -R explore-to-doc memwalker review-fix sandbox-fetch-optimize web-design .cursor/skills/
```

Cursor’s conventional path is `.cursor/skills/` (plural). This repo is the shared distribution layout for multiple tools.

---

## One-shot install script

Install every skill in this repo into Claude Code, Codex, and Cursor personal directories:

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

Save as `install.sh`, then: `chmod +x install.sh && ./install.sh`.

---

## Tool differences

| Skill | Cursor | Claude Code / Codex |
|-------|--------|---------------------|
| `explore-to-doc` | Read `docs/index.md` first; exploration docs under `docs/explore/<topic>/`; design/architecture under `docs/design/`; plans under `docs/tasks/`; temporary reviews under `docs/tmp/` (not indexed) | Same behavior; keep workspace-level docs out of per-package `docs/` |
| `memwalker` | Builds/navigates a summary tree (optional `.memwalker/` artifacts) | Same behavior across tools |
| `review-fix` | Verify review vs current code; failing test first; fix only after RED | Same behavior across tools |
| `sandbox-fetch-optimize` | Sandbox-first fetch cost compare; adopt only when score wins | Same behavior across tools |
| `web-design` | Same spec-first flow (`DESIGN.md` then code); URL crawl uses bundled Python scripts / Playwright | Same behavior across tools |

## Verify

```bash
# Claude Code
ls ~/.claude/skills/*/SKILL.md

# Codex
ls ~/.codex/skills/*/SKILL.md

# Cursor
ls ~/.cursor/skills/*/SKILL.md
```

Each skill directory must contain a file named **`SKILL.md`** (case-sensitive) with frontmatter fields `name` and `description`.

## Source

Exported for multi-tool reuse. Keep `SKILL.md` compatible with the Agent Skills format when you edit skills.
