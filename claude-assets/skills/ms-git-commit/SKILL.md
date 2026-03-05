---
name: ms-git-commit
description: 仅用 Git 分析改动并自动生成 conventional commit 信息（可选 emoji）；必要时建议拆分提交，默认运行本地 Git 钩子（可 --no-verify 跳过）；智能推断是否推送到远程
allowed-tools: Read(**), Exec(git status, git diff, git add, git restore --staged, git commit, git push, git pull, git rev-parse, git config), Write(.git/COMMIT_EDITMSG)
argument-hint: "[--no-verify] [--all] [--amend] [--signoff] [--emoji] [--scope <scope>] [--type <type>] [--push] [--no-push]"
# examples:
#   - /ms-git-commit # 分析改动，生成提交，智能推断是否推送
#   - /ms-git-commit --all # 暂存所有改动并提交
#   - /ms-git-commit --push # 强制推送（即使是新分支）
#   - /ms-git-commit --no-push # 仅提交，不推送
#   - /ms-git-commit --no-verify # 跳过 Git 钩子检查
#   - /ms-git-commit --emoji # 在提交信息中包含 emoji
#   - /ms-git-commit --scope ui --type feat # 指定作用域和类型
#   - /ms-git-commit --amend --signoff # 修补上次提交并签名
---

# Claude Skill: Commit (Git-only)

该命令在**不依赖任何包管理器/构建工具**的前提下，仅通过 **Git**：

- 读取改动（staged/unstaged）
- 判断是否需要**拆分为多次提交**
- 为每个提交生成 **Conventional Commits** 风格的信息（可选 emoji）
- 按需执行 `git add` 与 `git commit`（默认运行本地 Git 钩子；可 `--no-verify` 跳过）
- **智能推断是否推送**：若分支已有远程跟踪分支，自动推送；若是新分支，询问用户

---

## Usage

```bash
/ms-git-commit
/ms-git-commit --push
/ms-git-commit --no-push
/ms-git-commit --no-verify
/ms-git-commit --emoji
/ms-git-commit --all --signoff
/ms-git-commit --amend
/ms-git-commit --scope ui --type feat --emoji
```

### Options

- `--no-verify`：跳过本地 Git 钩子（`pre-commit`/`commit-msg` 等）。
- `--all`：当暂存区为空时，自动 `git add -A` 将所有改动纳入本次提交。
- `--amend`：在不创建新提交的情况下**修补**上一次提交（保持提交作者与时间，除非本地 Git 配置另有指定）。
- `--signoff`：附加 `Signed-off-by` 行（遵循 DCO 流程时使用）。
- `--emoji`：在提交信息中包含 emoji 前缀（省略则使用纯文本）。
- `--scope <scope>`：指定提交作用域（如 `ui`、`docs`、`api`），写入消息头部。
- `--type <type>`：强制提交类型（如 `feat`、`fix`、`docs` 等），覆盖自动判断。
- `--push`：**强制推送**到远程（即使是新分支，不询问）。
- `--no-push`：**仅提交**，不推送到远程。

> 注：如框架不支持交互式确认，可在 front-matter 中开启 `confirm: true` 以避免误操作。

---

## What This Skill Does

1. **仓库/分支校验**
   - 通过 `git rev-parse --is-inside-work-tree` 判断是否位于 Git 仓库。
   - 读取当前分支/HEAD 状态；如处于 rebase/merge 冲突状态，先提示处理冲突后再继续。

2. **改动检测**
   - 用 `git status --porcelain` 与 `git diff` 获取已暂存与未暂存的改动。
   - 若已暂存文件为 0：
     - 若传入 `--all` → 执行 `git add -A`。
     - 否则提示你选择：继续仅分析未暂存改动并给出**建议**，或取消命令后手动分组暂存。

3. **拆分建议（Split Heuristics）**
   - 按**关注点**、**文件模式**、**改动类型**聚类（示例：源代码 vs 文档、测试；不同目录/包；新增 vs 删除）。
   - 若检测到**多组独立变更**或 diff 规模过大（如 > 300 行 / 跨多个顶级目录），建议拆分提交，并给出每一组的 pathspec（便于后续执行 `git add <paths>`）。

4. **提交信息生成（Conventional 规范，可选 Emoji）**
   - 自动推断 `type`（`feat`/`fix`/`docs`/`refactor`/`test`/`chore`/`perf`/`style`/`ci`/`revert` …）与可选 `scope`。
   - 生成消息头：`[<emoji>] <type>(<scope>)?: <subject>`（首行 ≤ 72 字符，祈使语气，仅在使用 `--emoji` 时包含 emoji）。
   - 生成消息体：要点列表（动机、实现要点、影响范围、BREAKING CHANGE 如有）。
   - 根据 Git 历史提交的主要语言选择提交信息语言。优先检查最近提交主题（例如 `git log -n 50 --pretty=%s`）判断中文/英文；若无法判断，则回退到仓库主要语言或英文。
   - 将草稿写入 `.git/COMMIT_EDITMSG`，并用于 `git commit`。

5. **执行提交**
   - 单提交场景：`git commit [-S] [--no-verify] [-s] -F .git/COMMIT_EDITMSG`
   - 多提交场景（如接受拆分建议）：按分组给出 `git add <paths> && git commit ...` 的明确指令；若允许执行则逐一完成。

6. **智能推送（Push Intelligence）**
   - **检测推送需求**：
     - 执行 `git rev-parse --abbrev-ref @{u}` 检查当前分支是否有远程跟踪分支。
     - 若有跟踪分支（如 `origin/feature-branch`） → **自动推送**。
     - 若无跟踪分支（新分支） → **询问用户**：是否推送并设置上游？
   - **推送策略**：
     - 若传入 `--push` → 强制推送（即使是新分支，不询问）。
     - 若传入 `--no-push` → 跳过推送。
     - 若传入 `--amend` → 检测到修补提交且分支已推送，警告需要 force push，询问是否继续。
   - **推送失败处理**：
     - 若推送失败（如远程有新提交），自动执行 `git pull --rebase`（保持线性历史）。
     - 若 pull 产生冲突，尝试自动解决（基于语义分析）。
     - 若自动解决失败，报告冲突文件并给出建议命令。
     - 解决冲突后，重新推送。

7. **安全回滚**
   - 如误暂存，可用 `git restore --staged <paths>` 撤回暂存（命令会给出指令，不修改文件内容）。
   - 如误推送，给出回滚命令：`git revert <commit-sha>` 或 `git reset --hard HEAD~1 && git push --force`（需确认）。

---

## Best Practices for Commits

- **Atomic commits**：一次提交只做一件事，便于回溯与审阅。
- **先分组再提交**：按目录/模块/功能点拆分。
- **清晰主题**：首行 ≤ 72 字符，祈使语气（如 "add… / fix…"）。
- **正文含上下文**：说明动机、方案、影响范围、风险与后续工作。
- **遵循 Conventional Commits**：`<type>(<scope>): <subject>`。

---

## Type 与 Emoji 映射（使用 --emoji 时）

- ✨ `feat`：新增功能
- 🐛 `fix`：缺陷修复（含 🔥 删除代码/文件、🚑️ 紧急修复、👽️ 适配外部 API 变更、🔒️ 安全修复、🚨 解决告警、💚 修复 CI）
- 📝 `docs`：文档与注释
- 🎨 `style`：风格/格式（不改语义）
- ♻️ `refactor`：重构（不新增功能、不修缺陷）
- ⚡️ `perf`：性能优化
- ✅ `test`：新增/修复测试、快照
- 🔧 `chore`：构建/工具/杂务（合并分支、更新配置、发布标记、依赖 pin、.gitignore 等）
- 👷 `ci`：CI/CD 配置与脚本
- ⏪️ `revert`：回滚提交
- 💥 `feat`：破坏性变更（`BREAKING CHANGE:` 段落中说明）

> 若传入 `--type`/`--scope`，将**覆盖**自动推断。
> 仅在指定 `--emoji` 标志时才会包含 emoji。

---

## Guidelines for Splitting Commits

1. **不同关注点**：互不相关的功能/模块改动应拆分。
2. **不同类型**：不要将 `feat`、`fix`、`refactor` 混在同一提交。
3. **文件模式**：源代码 vs 文档/测试/配置分组提交。
4. **规模阈值**：超大 diff（示例：>300 行或跨多个顶级目录）建议拆分。
5. **可回滚性**：确保每个提交可独立回退。

---

## Examples

### Good (使用 --emoji)

- ✨ feat(ui): add user authentication flow
- 🐛 fix(api): handle token refresh race condition
- 📝 docs: update API usage examples
- ♻️ refactor(core): extract retry logic into helper
- ✅ test: add unit tests for rate limiter
- 🔧 chore: update git hooks and repository settings
- ⏪️ revert: revert "feat(core): introduce streaming API"

### Good (不使用 --emoji)

- feat(ui): add user authentication flow
- fix(api): handle token refresh race condition
- docs: update API usage examples
- refactor(core): extract retry logic into helper
- test: add unit tests for rate limiter
- chore: update git hooks and repository settings
- revert: revert "feat(core): introduce streaming API"

### Split Example

- `feat(types): add new type defs for payment method`
- `docs: update API docs for new types`
- `test: add unit tests for payment types`
- `fix: address linter warnings in new files` ←（如你的仓库有钩子报错）

---

## Important Notes

- **仅使用 Git**：不调用任何包管理器/构建命令（无 `pnpm`/`npm`/`yarn` 等）。
- **尊重钩子**：默认执行本地 Git 钩子；使用 `--no-verify` 可跳过。
- **不改源码内容**：命令只读写 `.git/COMMIT_EDITMSG` 与暂存区；不会直接编辑工作区文件。
- **智能推送**：
  - 若分支已有远程跟踪分支 → 自动推送。
  - 若是新分支 → 询问是否推送并设置上游（`git push -u`）。
  - 使用 `--push` 强制推送，`--no-push` 跳过推送。
- **推送失败自动修复**：
  - 远程有新提交时，自动 `git pull --rebase` 并重新推送。
  - 冲突会尝试自动解决（基于语义分析）。
- **安全提示**：在 rebase/merge 冲突、detached HEAD 等状态下会先提示处理/确认再继续。
- **可审可控**：如开启 `confirm: true`，每个实际 `git add`/`git commit`/`git push` 步骤都会进行二次确认。
- **amend + push 警告**：若使用 `--amend` 且分支已推送，会警告需要 force push，询问是否继续。
