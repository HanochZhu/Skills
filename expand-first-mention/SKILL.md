---
name: expand-first-mention
description: >-
  On first mention in each conversation, expand new keywords and abbreviations
  with their full description so short forms are not misleading
  (e.g. OSS = Open Source Software（开源版 / 自托管 SDK）).
  Use when writing replies that introduce acronyms, jargon, product codes,
  or ambiguous short names; or when the user mentions 缩写、关键词首次展开、
  first mention, glossary, or expand abbreviations.
---

# 首次出现，写全称

将每一个对话中新遇到的关键词，比如OSS= Open Source Software（开源版 / 自托管 SDK这种，第一次输出是，输出完整的描述。避免有些缩写导致误导。

## 何时启用

写给用户看的内容时默认启用：回复、新写的文档、说明、计划。用户点名「展开缩写 / 关键词写全」时也启用。

不要用于：只改代码标识符；复述用户已经写全的同一句话；本对话里已经展开过的词再次出现。

## 规则

1. **范围是当前这一次对话**，不是跨会话词库。换对话后，关键词再次出现仍要第一次写全。
2. **第一次由你输出该词时**写完整描述；之后可用短称。
3. **完整描述要带上本语境含义**，不要只丢字典全称。多义词必须写清这次指哪一种。
4. **行内展开**，不要每条回复开头先甩一张术语表。一条回复里新词 ≥4 个时，可在段首加一行对照，其余仍行内写全。
5. 用户已经给出全称或明确定义时，沿用用户的定义，不要另造一套。

## 格式

默认：

```text
缩写 = 英文全称（中文释义 / 本语境补充）
```

- 无通行英文全称：`关键词（完整描述）`
- 无歧义且无中文必要：`缩写 = 英文全称` 即可
- 括号里写**这次对话里它实际指什么**，不是所有可能义项的罗列

## 要展开

- 缩写 / 首字母缩略，尤其多义（OSS、RAG、SDK、CAM、MCP）
- 领域黑话、产品内部名、容易和别的含义撞车的短词
- 本对话第一次出现、读者可能按另一种常见义理解的词

## 不要展开

- 本对话已经展开过的词（再用短称）
- 无歧义日常词、文件名、函数名、路径、URL
- 代码块里的标识符（不要改代码里的短名）
- 极常见且本语境只有一种读法的词（如 JSON、HTTP）；若本次被赋予特殊含义，仍要写全

## 例子

第一次：

- `OSS = Open Source Software（开源版 / 自托管 SDK）`
- `RAG = Retrieval-Augmented Generation（检索增强生成）`
- `CAM = Code Agent Memory（本仓库的代码图 + 解法记忆）`

同一对话后文：`OSS`、`RAG`、`CAM` 可直接用。

反例（会误导）：只写「先看 OSS 文档」——读者可能理解成对象存储，而不是开源版 SDK。

## 与其它技能

本技能只管**对用户可见文本的用词**；不改变探索落档、阅读或修复流程。
