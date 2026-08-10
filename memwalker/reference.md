# MemWalker reference

Paper: [arXiv:2310.05029](https://arxiv.org/abs/2310.05029) — *Walking Down the Memory Maze: Beyond Context Limit through Interactive Reading*.

Prompts below follow Appendix A.1 of the paper, adapted for agent use (not only story QA).

## Construction prompts

### Leaf (segment → summary)

```text
[TEXT_OF_SEGMENT]

Summarize the above text comprehensively into a fluent passage.
Preserve proper nouns, numbers, APIs, file/symbol names, and constraints.
```

### Non-leaf (children → parent)

```text
[SUMMARIES]

Compress each summary into a much shorter summary while keeping
discriminative cues useful for later navigation (who/what/where, key claims).
```

If concatenated child summaries exceed the budget, apply the non-leaf prompt again before attaching to the parent.

## Navigation prompts

### Triage (non-leaf)

```text
The following passage(s) are the summaries of different parts of the material.
To answer the question: [QUERY]
Which of the following summary is MOST LIKELY to contain information about the answer?
First provide reasoning to compare the summaries before you make the decision.

Summary 0: [CHILD_SUMM_NODE_0]
Summary 1: [CHILD_SUMM_NODE_1]
…
Summary N: [CHILD_SUMM_NODE_N]

Working memory (optional):
[WORKING_MEMORY]

Reply with the passage number as your action. You MUST choose one summary number
and reply with the following format:
###################################
Reasoning: …
Action: 0 / 1 / 2 / …
###################################

If none look relevant and you previously descended from a parent, you may instead:
###################################
Reasoning: …
Action: -1
###################################
(Action -1 = revert to parent.)
```

### Leaf

```text
Read the text below and answer the question.

Background / working memory:
[WORKING_MEMORY]

Main text:
"""
[TEXT_OF_SEGMENT]
"""

Question: [QUERY]
[OPTIONS if any]

If the answer CANNOT be inferred from the text above, reply with action -1.
If the answer CAN be inferred, reply with action -2, plus reasoning and the final answer.
You are ONLY allowed to reply with action -2 or -1.

###################################
Reasoning: …
Action: -2 or -1
Answer: …
###################################
```

## Suggested tree file layout

```markdown
# MemWalker tree — <source>

- source: `path/or/uri`
- built: YYYY-MM-DD
- segment_size: …
- max_children: …

## Working Memory
- …

## Trajectory
- …

## Nodes

### N0 (root, L2)
- children: N1, N2
- summary: …

### N1 (L1)
- children: N3, N4
- summary: …

### N3 (leaf, L0)
- source_span: `file:start-end`
- summary: …
- text_ref: (read on demand; do not embed full text unless small)
```

## Design notes from the paper

- Construction is **query-independent** → cache and reuse across questions on the same source.
- Stronger instruction-following models benefit from **explicit reasoning** before each action; weaker models may get worse when forced to reason.
- Working memory along the path is important (paper: ~5–13% accuracy drop without it).
- Revert enables recovery from early wrong branches (stray ~15–20%; high recovery when reasoning works).
- Prefer reading only needed branches; successful paths often process well under 100% of original tokens.
