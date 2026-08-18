---
name: codely-git-commit
description: >-
  Commit via `codely --example-prompt git-commit -y`, then push.
  Use when the user asks to commit, git commit, 提交, or commit and push.
---

# Codely Git Commit

本仓库提交**必须**走 Codely example prompt，禁止直接 `git commit`。

## 流程

1. 在包含改动的 Git 仓库目录执行（多仓时分别处理）。
2. `git status` / `git diff` 逐文件对照用户本次要求，区分「本次修改相关」与无关改动。
3. **只 `git add` 本次修改相关的路径**（显式列出文件）。无关代码一律不 add：
   - 禁止 `git add .` / `git add -A` / `git add --all` / `git add -u`
   - 工作区里其它需求、调试、格式化、未要求的文件或 hunk 都保持未暂存
   - 同一文件混有无关改动时：不要整文件 add；先说明混杂内容，只处理能单独拆出的相关文件，否则停下来问用户
   - 不要暂存密钥（`.env`、`credentials.json` 等）
   - add 后用 `git diff --cached --name-only` 核对暂存清单；出现无关文件则 `git restore --staged` 去掉
   - 没有任何相关暂存则停止，不要空提交
4. 在该仓库目录执行（命令原文，不要改写成 `git commit`）：

```bash
codely --example-prompt git-commit -y
```

5. 该命令审查 **staged** 变更、按 conventional commits 生成英文说明并提交。等其结束后检查 `git status`：
   - 提交成功且用户要求提交（本 skill 默认随后 push）：`git push`
   - 无新 commit / 失败：不要 push，报告错误并停止
6. 不要 `--no-verify`、不要 amend（除非用户明确要求且满足安全条件）、不要 force push。

## 注意

- `git-commit` 只看暂存区；未 `git add` 会提示无 staged changes。
- `-y` 即 `--yolo`，自动确认 Codely 工具调用。
- 需要较长超时（通常数十秒到数分钟）。
