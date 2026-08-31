---
name: memwalker
description: >-
  Apply MemWalker interactive reading: build a hierarchical summary tree over
  long text/code, then navigate it with reasoning, working memory, and revert.
  Scope: long docs/logs/transcripts/large modules that exceed one reliable read;
  long-doc QA needing an explainable path. Not for short files, known-path edits,
  or simple grep hits. Use when the user mentions MemWalker, memory maze,
  interactive reading, or summarize-then-walk navigation.
---

# MemWalker（交互式长文阅读）

基于论文 *Walking Down the Memory Maze*（Chen et al., arXiv:2310.05029）：把有限上下文的模型当作**交互式阅读代理**，而不是一次性塞进全文。

## 适用范围

MemWalker 只处理**外部长材料**上的「先建摘要树、再按 query 行走」；不压缩当前对话，也不替代定点检索。

### 适用材料

| 类型 | 典型对象 | 说明 |
|------|----------|------|
| 长文档 | 设计稿、规格、论文、会议纪要、API 手册 | 需跨多段综合或定位细节 |
| 长日志 / dump | 构建日志、运行日志、trace、崩溃栈合并文本 | 关键词检索易漏上下文 |
| 长对话 / transcript | 导出的聊天记录、客服会话、评审纪要 | 答案分散在多轮中 |
| 大模块代码 | 单文件过长，或跨多文件的子系统 | 目录/文件/函数块可映射为树节点 |
| 用户点名 | 明确要求 MemWalker / memory tree / interactive reading | 即使材料中等长度也可启用 |

### 规模阈值（经验）

- **建议启用**：单源已超过一次 `Read` 的可靠窗口（约数千行 / 数万 tokens），或「截断后读」会明显丢线索
- **可选用**：材料中等，但问题依赖多段对照，且需要可回溯的阅读路径
- **不必启用**：材料可一次读完；或路径/符号已知，用 `Grep`/`Glob`/`Read` 即可命中

### 适用任务

- 长文问答（事实抽取、条件/约束汇总、多处证据对齐）
- 大模块理解与定位（「某能力在哪实现」「调用链大致落在哪几处」）
- 需要**可解释路径**的阅读（选了哪支、为何回退、最终依据哪段原文）
- 同一长源上的**多轮不同 query**（Stage 1 树可缓存复用）

### 明确不适用

- 短文件、片段级问答、已知路径的定点编辑
- 简单符号/字符串检索就能直接命中的问题
- 需要**改代码/写文档**本身（可先 MemWalker 定位，再交回普通编辑流程）
- **当前会话** context 爆满（不要对本对话建 MemWalker 树）
- 目标是把探索过程沉淀成项目文档（用 `explore-to-doc`；可将 tree/trajectory 引用过去）

### 何时启用（触发条件）

满足任一即可启用：

- 材料落在上方「适用材料」且达到规模阈值，无法一次可靠读完并作答
- 用户要求按 MemWalker / memory tree / interactive reading 处理
- 长文问答 / 长对话复盘 / 大模块定位，且检索或截断效果不稳
- 需要可解释的导航轨迹与原文依据

## 两阶段流程（必须）

```
Task Progress:
- [ ] Stage 1: 构建 memory tree（与 query 无关，可缓存）
- [ ] Stage 2: 从根节点导航（依赖 query）
- [ ] 输出：答案 + 导航轨迹 + 依据片段
```

### Stage 1 — Memory tree construction

1. 将长文本切成可放入上下文的 **segments**（默认约 800–1500 tokens/段；代码可按逻辑块或固定行数切）。
2. 对每个 segment 生成 **leaf summary**（全面、流畅，保留专有名词、数字、API/符号名）。
3. 将若干 leaf summaries **分组再总结**为更高层节点，直到生成 **root**。
4. 每个节点保存：`id`、`level`、`summary`、`children`、叶子对应的 `source_span`（路径/行号/偏移）。

默认树参数（可按材料调整）：

| 参数 | 建议 | 说明 |
|------|------|------|
| segment size | 1000 tokens / ~80–120 行代码 | 过大丢细节，过小树太深 |
| max children / parent | 5–8 | 过多单步决策变难 |
| 缓存 | 同一源文件可复用树 | 换 query 不必重建 |

摘要提示模板见 [reference.md](reference.md)。

### Stage 2 — Navigation

从 **root** 开始，每步先写 **Reasoning**，再选 **Action**。

**非叶节点（triage）**

- 输入：query + 各 child summary（+ working memory）
- 动作：选择最可能含答案的 child 编号；必要时 **revert** 回父节点
- 必须先比较各 summary，再决策

**叶节点（leaf）**

- 输入：segment 原文 + query + working memory
- 动作：
  - `-2`：信息足够 → 给出答案
  - `-1`：不足 → revert 到父节点，换路径

**Working memory**

- 沿路径携带已访问节点的关键摘要/事实（截断到上下文可容纳）
- 禁止只看当前 children 而丢掉路径上下文（论文中无 working memory 会明显掉点）

**失败与恢复**

- 输出格式非法：最多重试 3 次；仍失败则该步标记无效并 revert 或终止
- 走错路径：用 revert 回退（论文约 15–20% 路径会 stray，多数可恢复）
- 连续无效后：报告「无法从当前树可靠作答」，并给出已探索路径与建议（加深 segment / 重建树 / 缩小范围）

导航提示模板见 [reference.md](reference.md)。

## Agent 落地方式（Cursor 工具）

不必实现独立服务；用工具模拟树与行走：

1. **建树**：用 `Read`（分段 offset/limit）或按目录/模块分层；把节点写到工作区临时文件，如 `.memwalker/<source-slug>-tree.md`（或用户指定路径）。
2. **导航**：每步只加载当前节点所需 children summaries 或 leaf 原文；更新同一文件中的 `Working Memory` 与 `Trajectory` 小节。
3. **代码库**：目录/模块 ≈ 高层节点；文件 ≈ 中层；函数/类块 ≈ leaf。用 `Grep`/`Glob` 辅助定位，但仍走「摘要比较 → 进入 → 必要时回退」循环，避免盲目全文扫。
4. **交付**：最终回答必须附带简短导航轨迹（节点序列 + 是否 revert）和原文依据位置。

## 输出格式

```markdown
## Answer
<最终答案>

## Trajectory
1. root → child N — <一句理由>
2. … → leaf / revert
3. …

## Evidence
- `path` (lines/span): <支撑结论的要点>

## Tree artifact
- `.memwalker/<slug>-tree.md`（若已落盘）
```

## 与其它技能

| 技能 | 关系 |
|------|------|
| `explore-to-doc` | 探索过程落档到 `docs/explore/<topic>/`；MemWalker 的 tree/trajectory 可引用过去或留在 `.memwalker/` |

## 局限（论文 Limitations）

- 极长序列时建树成本高 → 可增大 segment、降低粒度，或只对相关子树建树
- 弱推理模型导航易级联出错 → 必须强制 Reasoning；弱模型可简化为少步检索
- 零样本 prompting；成功路径可事后沉淀为示例，但不默认微调

## 引用

Chen, H., Pasunuru, R., Weston, J., & Celikyilmaz, A. (2023). *Walking Down the Memory Maze: Beyond Context Limit through Interactive Reading*. arXiv:2310.05029. https://arxiv.org/abs/2310.05029
