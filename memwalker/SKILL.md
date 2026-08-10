---
name: memwalker
description: >-
  Apply MemWalker interactive reading: build a hierarchical summary tree over
  long text/code, then navigate it with reasoning, working memory, and revert.
  Use when the user mentions MemWalker, memory maze, interactive reading of long
  documents, context exceeds what fits in one pass, long-doc QA, or navigating
  large files/logs/codebases by summarize-then-walk.
---

# MemWalker（交互式长文阅读）

基于论文 *Walking Down the Memory Maze*（Chen et al., arXiv:2310.05029）：把有限上下文的模型当作**交互式阅读代理**，而不是一次性塞进全文。

## 何时启用

满足任一条件时启用：

- 单文件/文档/日志过长，无法一次可靠读完并作答
- 用户要求按 MemWalker / memory tree / interactive reading 处理
- 长文问答、长对话复盘、大模块代码定位，且检索/截断效果不稳
- 需要**可解释路径**（选了哪一段、为何回退、最终依据哪段原文）

短文件、已知路径的定点编辑、简单 grep 能直接命中：**不启用**。

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
| `explore-to-doc` | 探索过程落档；MemWalker 的 tree/trajectory 可写入同一 docs 或引用 `.memwalker/` |
| `auto-summary-context` | 压缩**当前对话**；MemWalker 处理的是**外部长材料** |

## 局限（论文 Limitations）

- 极长序列时建树成本高 → 可增大 segment、降低粒度，或只对相关子树建树
- 弱推理模型导航易级联出错 → 必须强制 Reasoning；弱模型可简化为少步检索
- 零样本 prompting；成功路径可事后沉淀为示例，但不默认微调

## 引用

Chen, H., Pasunuru, R., Weston, J., & Celikyilmaz, A. (2023). *Walking Down the Memory Maze: Beyond Context Limit through Interactive Reading*. arXiv:2310.05029. https://arxiv.org/abs/2310.05029
