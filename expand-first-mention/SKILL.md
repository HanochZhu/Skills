---
name: expand-first-mention
description: >-
  Unify first-mention glossary format. Use when user asks to 展开缩写、关键词写全,
  or edit this format / always-on rule. Daily expansion is always-on.md /
  cursor-rule.mdc, not this skill.
disable-model-invocation: true
---

对话里新出现的多义缩写 / 内部名，第一次写全：`缩写 = 英文全称（本语境释义）`。无通行英文全称用 `关键词（完整描述）`。后文短称。

不展开：无歧义词、JSON/HTTP、文件名、代码标识符、本对话已展开过的。

例：`OSS = Open Source Software（开源版 / 自托管 SDK）`。日常贴 `always-on.md`；Cursor 用 `cursor-rule.mdc`。
