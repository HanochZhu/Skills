---
name: git-commit
description: >-
  Standardized git commit and push workflow: selective staging, conventional commits message generation, safe git commit, and push.
  Use when the user asks to commit, git commit, 提交, commit and push, or 提交并推送.
---

# Git Commit

通用 Git 提交与推送工作流规范：严格的选区暂存、标准 Conventional Commits 说明生成、安全提交与按需推送。
适用于任何 Agent 环境（Cursor、Claude Code、Codex、Windsurf、终端等），不依赖特定 CLI 工具。

## 核心原则

1. **精准暂存，严禁全量**：只暂存与本次任务直接相关的改动，绝不滥用 `git add .`。
2. **提交信息规范化**：严格遵循 Conventional Commits 规范，统一使用英文，包含 Header、Body 说明与 Bullet points 清单。
3. **安全提交与防御性校验**：默认遵守 pre-commit hook，严禁随意 `--no-verify`，严禁未经用户明确同意的 `amend` 或 force push。

---

## 工作流

```
Git Commit Workflow:
- [ ] 1. 检查工作区状态（git status / git diff）
- [ ] 2. 精准暂存相关文件（git add <file>...，排除无关文件与敏感信息）
- [ ] 3. 校验暂存区（git diff --cached --name-only）
- [ ] 4. 审查暂存差异（git diff --cached）
- [ ] 5. 生成 Conventional Commit 信息（Header < 50 chars + Body + Bullets）
- [ ] 6. 执行提交（git commit -m ...）
- [ ] 7. 按需推送并汇报（git push / 检查 upstream）
```

---

## 详细步骤

### 1. 检查工作区状态

在包含改动的 Git 仓库目录下执行（多仓库或 monorepo 时先进入对应的目标仓库目录）：

```bash
git status
```

逐文件对照本次用户需求，清晰区分：
- 「本次修改相关」的代码与资源
- 「无关改动」：例如其它需求的临时修改、调试日志、未要求的代码格式化、本地临时配置等

### 2. 精准暂存（Selective Staging）

**只暂存与本次修改明确相关的路径**（必须显式指定文件或目录）：

- **严禁全量暂存**：禁止使用 `git add .`、`git add -A`、`git add --all`、`git add -u`。
- **保留无关改动未暂存**：工作区中未被本次需求包含的文件或 hunk 必须保持未暂存状态。
- **严禁暂存敏感信息**：绝对不能暂存包含秘钥、Token、密码的文件（如 `.env`、`*.pem`、`credentials.json`、`id_rsa`、`*.key` 等）。
- **单文件混杂处理**：若同一文件内同时包含相关改动和无关修改：
  - 优先交互式暂存（如 `git add -p`）或拆分暂存；
  - 若无法安全剥离，必须向用户明确说明混杂内容，征得确认后再处理，不要整文件盲目 add。

### 3. 校验暂存区

暂存后立即核对已暂存清单：

```bash
git diff --cached --name-only
```

- 若发现混入了无关文件或误暂存文件，立即移出暂存区：
  ```bash
  git restore --staged <unrelated_file>
  ```
- **空暂存拦截**：若暂存区没有任何相关文件，立即停止操作，向用户说明原因，绝不进行空提交（禁止 `git commit --allow-empty`，除非用户特别要求）。

### 4. 审查暂存变更与生成 Commit Message

运行以下命令审查待提交的完整差异：

```bash
git diff --cached
```

#### Commit Message 结构规范

Commit 说明**统一使用英文**以保证跨团队与开源项目的一致性。结构必须包含三大部分：

1. **Header（首行标题）**：
   - 格式：`<type>(<optional scope>): <subject>`
   - 长度：必须**小于 50 字符**。
   - 语气：使用祈使句（如 `add`, `fix`, `update`, `refactor`，不用 `added`, `fixing`）。
   - 标点：末尾**不加句号**。
   - 常见 type：
     - `feat`: 新功能或新特性
     - `fix`: 修复 bug
     - `docs`: 仅文档变更
     - `style`: 代码格式调整（空格、分号等，不改变运行逻辑）
     - `refactor`: 重构（既非新增功能也非修复 bug）
     - `perf`: 性能优化
     - `test`: 增加或修正测试用例
     - `chore`: 构建过程、辅助工具或依赖更新
     - `ci`: CI/CD 配置与脚本调整

2. **Body（上下文与动机段落）**：
   - 与 Header 之间保留一个空行。
   - 长度在 200 词以内，说明本次修改的**背景与目的**：解决了什么问题、为什么需要修改、关键技术决策或背景上下文。

3. **Bullet Points（主要变更清单）**：
   - 与 Body 之间保留一个空行。
   - 每一条使用 `- ` 开头，重点突出、条理清晰。
   - 尽量一行一条，按模块或逻辑分组列出关键修改点。

#### 格式范例

```text
feat(auth): add JWT token refresh endpoint

This commit introduces a dedicated /auth/refresh endpoint to support silent token renewal for frontend clients. It extends the existing auth handler to validate refresh tokens against the session store, improving session longevity without requiring re-login.

- Add POST /auth/refresh route handler and middleware validation
- Implement refresh token verification and expiration checks
- Add integration tests covering valid, expired, and revoked refresh tokens
- Update OpenAPI schema documentation for auth endpoints
```

### 5. 执行提交（git commit）

为确保换行与跨平台终端一致性，使用多段 `-m` 参数执行提交：

```bash
git commit -m "<Header>" \
  -m "<Body paragraph>" \
  -m "- Bullet 1" \
  -m "- Bullet 2"
```

#### 提交安全规则

- **正常执行 pre-commit hooks**：默认必须允许仓库配置的 Git Hook 正常执行。
- **Hook 失败处理**：如果 pre-commit hook 报错失败，**必须立即终止流程并报错**。不要为了提交而盲目追加 `--no-verify`；也不要自行乱改代码，应将 hook 报错清晰汇报给用户。
- **禁止随意 amend**：除非用户明确要求（且上一个 commit 尚未 push），否则禁止使用 `git commit --amend`。
- **禁止跳过校验**：严禁无理由使用 `--no-verify`。

### 6. 推送与同步（git push）

检查提交结果：

```bash
git status
git log -1 --stat
```

- **推送时机**：
  - 若用户明确要求推送（如「提交并推送」、「commit and push」、「push」）或默认包含 push 诉求：执行 `git push`。
  - 若当前分支尚未关联上游分支，检查当前分支名并使用：
    ```bash
    git push -u origin <current-branch>
    ```
- **严禁 force push**：绝对禁止未经用户极其明确要求的 `git push --force` 或 `git push --force-with-lease`。
- **失败处置**：若 push 失败（如遇到冲突或远端领先），停止并提示用户先 pull/rebase，切勿暴力覆盖。

### 7. 最终汇报

提交与推送完成后，向用户简洁汇报：
- **Commit SHA**（短哈希）
- **Branch**（所在分支）
- **Commit Message**（标题与主要摘要）
- **Remote Push 状态**（已推送分支与远端状态，或未推送提示）
