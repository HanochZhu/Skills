---
name: expand-first-mention
description: >-
  First-mention expansion format for ambiguous abbreviations and jargon
  (e.g. OSS = Open Source Software（开源版 / 自托管 SDK）).
  Use when the user asks to 展开缩写、关键词写全、unify glossary style,
  or edit this format / the always-on first-mention rule.
  Daily first-mention behavior is the always-on Cursor rule, not this skill.
---

# 首次出现，写全称

将每一个对话中新遇到的关键词，比如OSS= Open Source Software（开源版 / 自托管 SDK这种，第一次输出是，输出完整的描述。避免有些缩写导致误导。

日常遵守靠 Cursor always-on rule（`cursor-rule.mdc`）。本 skill 只统一格式和反例；不要把它当成每条回复都要加载的流程。

## 判定

只展开这些：

- 多义缩写（同一短称有两种以上常见读法）
- 本仓库 / 产品内部名
- 读者可能按另一种常见义理解的短称

不要展开：无歧义日常词、JSON / HTTP、文件名、代码标识符、本对话已经展开过的词。

## 格式

```text
缩写 = 英文全称（中文释义 / 本语境补充）
```

- 无通行英文全称：`关键词（完整描述）`
- 括号里写**这次对话里它实际指什么**，不是所有义项

## 例子

第一次：`OSS = Open Source Software（开源版 / 自托管 SDK）`

反例：只写「先看 OSS 文档」——读者可能理解成对象存储。

同一对话后文用短称。
