---
description: PR 前将开发分支的多个 commit 压缩成一个，自动同步上游代码并生成 Conventional Commits 风格的提交信息（可选 emoji）
allowed-tools: Bash(git fetch, git checkout, git merge, git reset, git commit, git branch, git rev-parse, git log, git status, git diff, git remote, git merge-base), Read(**), Write(.git/COMMIT_EDITMSG)
argument-hint: [--remote <remote>] [--branch <branch>] [--no-sync] [--backup] [--no-verify] [--emoji] [--scope <scope>] [--type <type>]
# examples:
#   - /git-squash-commits                           # 自动推断，压缩所有提交
#   - /git-squash-commits --emoji                   # 使用 emoji 提交信息
#   - /git-squash-commits --scope ui --type feat    # 指定作用域和类型
#   - /git-squash-commits --no-sync                 # 仅压缩，不同步上游
#   - /git-squash-commits --backup                  # 压缩前创建备份分支
---

# Claude Command: Squash Commits (Git-only)

该命令用于**在提交 PR 前将开发分支的多个 commit 压缩成一个干净的 commit**，通过 **Git**：

- 自动检测上游分支（upstream/develop > origin/develop > origin/main > origin/master）
- 自动同步上游代码（fetch + merge，先解决冲突）
- 备份当前分支（可选，通过 `--backup` 控制）
- 压缩提交（`git reset --soft <upstream-branch>`）
- 生成 **Conventional Commits** 风格的提交信息（可选 emoji）
- 验证结果（显示压缩前后的提交对比）

---

## Usage

```bash
/git-squash-commits
/git-squash-commits --emoji
/git-squash-commits --scope ui --type feat
/git-squash-commits --no-sync
/git-squash-commits --backup
/git-squash-commits --remote upstream --branch develop
/git-squash-commits --no-verify --emoji
```

### Options

- `--remote <remote>`：指定远程仓库名称（如 `upstream`、`origin`）。省略则自动推断（优先 `upstream`，回退到 `origin`）。
- `--branch <branch>`：指定上游分支名称（如 `develop`、`main`、`master`）。省略则自动推断（优先 `develop`，回退到 `main`/`master`）。
- `--no-sync`：**仅压缩提交**，不同步上游代码（适用于已经同步过的场景）。
- `--backup`：在压缩前创建备份分支（命名为 `<当前分支>-backup-<时间戳>`）。
- `--no-verify`：跳过本地 Git 钩子（`pre-commit`/`commit-msg` 等）。
- `--emoji`：在提交信息中包含 emoji 前缀（省略则使用纯文本）。
- `--scope <scope>`：指定提交作用域（如 `ui`、`docs`、`api`），写入消息头部。
- `--type <type>`：强制提交类型（如 `feat`、`fix`、`docs` 等），覆盖自动判断。

> 注：如框架不支持交互式确认，可在 front-matter 中开启 `confirm: true` 以避免误操作。

---

## What This Command Does

1. **仓库/分支校验**
   - 通过 `git rev-parse --is-inside-work-tree` 判断是否位于 Git 仓库。
   - 检查当前分支状态；如处于 detached HEAD、rebase/merge 冲突状态，先提示处理后再继续。
   - 确保工作区干净（`git status --porcelain`），若有未提交改动，提示用户先提交或暂存。

2. **上游分支检测与验证**
   - **检测 remote**：
     - 若传入 `--remote` → 使用指定 remote。
     - 否则自动推断：`upstream` > `origin`（优先 `upstream`，若不存在则使用 `origin`）。
   - **检测 branch**：
     - 若传入 `--branch` → 使用指定 branch。
     - 否则自动推断：`develop` > `main` > `master`（检查远程分支是否存在）。
   - **验证上游分支**：
     - 执行 `git rev-parse --verify <remote>/<branch>` 验证分支存在性。
     - 若上游分支不存在，报错并退出。

3. **上游代码同步（可选）**
   - **若传入 `--no-sync`**：跳过此步骤，直接进入压缩流程。
   - **否则**：
     - 执行 `git fetch <remote>` 拉取最新代码。
     - 执行 `git merge <remote>/<branch>` 合并到当前分支。
     - **冲突处理**：
       - 若产生冲突，尝试自动解决（基于语义分析）。
       - 若自动解决失败，报告冲突文件并给出建议命令：
         ```bash
         # 手动解决冲突后执行：
         git add <冲突文件>
         git commit -m "resolve merge conflicts"
         # 然后重新运行此命令
         ```
       - 退出命令，等待用户手动处理。

4. **备份当前分支（可选）**
   - **若传入 `--backup`**：
     - 创建备份分支：`git branch <当前分支>-backup-<时间戳>`。
     - 输出备份分支名称供用户记录。
   - **否则**：跳过此步骤。

5. **统计要压缩的提交**
   - 执行 `git log --oneline <remote>/<branch>..HEAD` 统计提交数量。
   - 若提交数量为 0，提示用户当前分支没有新提交，退出命令。
   - 若提交数量为 1，询问用户是否继续（已经是单个提交，可能不需要压缩）。
   - 输出提交列表供用户确认：
     ```text
     检测到 5 个提交需要压缩：
     - abc1234 feat: add feature A
     - def5678 fix: typo in feature A
     - ghi9012 refactor: extract helper function
     - jkl3456 docs: update API docs
     - mno7890 chore: format code
     ```

6. **执行软重置（压缩提交）**
   - 执行 `git reset --soft <remote>/<branch>`。
   - **原理**：
     - `--soft` 参数只移动 HEAD 指针，不改变暂存区和工作区。
     - 所有代码更改保留在暂存区，等待创建新的提交。

7. **生成压缩后的提交信息**
   - **分析改动**：
     - 读取暂存区的改动（`git diff --cached`）。
     - 统计改动的文件、新增/删除行数、涉及的目录/模块。
   - **推断提交类型与作用域**：
     - 若传入 `--type`/`--scope` → 使用指定值。
     - 否则自动推断：
       - 根据改动文件路径推断 `scope`（如 `src/ui/` → `ui`）。
       - 根据改动内容推断 `type`（如新增功能 → `feat`，修复 bug → `fix`）。
   - **生成提交信息**：
     - 消息头：`[<emoji>] <type>(<scope>)?: <subject>`（首行 ≤ 72 字符，祈使语气，仅在使用 `--emoji` 时包含 emoji）。
     - 消息体：
       - 列出被压缩的原始提交标题（作为历史记录）。
       - 说明改动的动机、实现要点、影响范围。
       - 若有 BREAKING CHANGE，单独列出。
     - 根据 Git 历史提交的主要语言选择提交信息语言（中文/英文）。
   - **写入草稿**：
     - 将提交信息写入 `.git/COMMIT_EDITMSG`。

8. **创建新的压缩提交**
   - 执行 `git commit [--no-verify] -F .git/COMMIT_EDITMSG`。
   - **若提交失败**（如钩子报错），保留暂存区状态，输出错误信息并给出建议命令。

9. **验证与对比**
   - 执行 `git log --oneline --graph -5` 显示最近提交历史。
   - 对比压缩前后的提交数量：
     ```text
     压缩完成！
     - 原提交数量：5
     - 压缩后提交数量：1
     - 压缩提交 hash：abc1234
     - 压缩提交标题：feat(ui): implement user authentication flow
     ```
   - 若创建了备份分支，提醒用户：
     ```text
     备份分支已创建：feat-dev-backup-20260226-143022
     若需要恢复，请执行：git reset --hard feat-dev-backup-20260226-143022
     ```

---

## Best Practices

- **压缩时机**：在提交 PR 前压缩，保持 PR 历史干净整洁。
- **备份习惯**：首次使用建议添加 `--backup`，以防万一操作失误。
- **同步先行**：建议先同步上游代码（不使用 `--no-sync`），避免后续 PR 合并时产生冲突。
- **提交信息质量**：压缩后的提交信息应准确反映所有改动的目的，而非仅描述最后一次提交。
- **保留历史**：若原提交历史对审阅有价值（如详细的实现步骤），可在提交消息体中列出原始提交标题。

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

## Examples

**Good (使用 --emoji)**

- ✨ feat(ui): implement user authentication flow
- 🐛 fix(api): handle token refresh race condition
- 📝 docs: update API usage examples
- ♻️ refactor(core): extract retry logic into helper
- ✅ test: add unit tests for rate limiter
- 🔧 chore: squash commits before PR

**Good (不使用 --emoji)**

- feat(ui): implement user authentication flow
- fix(api): handle token refresh race condition
- docs: update API usage examples
- refactor(core): extract retry logic into helper
- test: add unit tests for rate limiter
- chore: squash commits before PR

**提交消息体示例（列出原始提交）**

```text
feat(ui): implement user authentication flow

实现用户认证流程，包括登录、注册、密码重置功能。

原始提交历史：
- abc1234 feat: add login form component
- def5678 fix: typo in login validation
- ghi9012 refactor: extract validation logic
- jkl3456 docs: add API documentation
- mno7890 chore: format code

主要改动：
- 新增 LoginForm、RegisterForm 组件
- 实现 JWT token 验证逻辑
- 添加密码强度检查
- 更新 API 文档

影响范围：
- src/components/auth/
- src/services/auth-service.ts
- docs/api/authentication.md
```

---

## Important Notes

- **仅使用 Git**：不调用任何包管理器/构建命令（无 `pnpm`/`npm`/`yarn` 等）。
- **尊重钩子**：默认执行本地 Git 钩子；使用 `--no-verify` 可跳过。
- **不改源码内容**：命令只读写 `.git/COMMIT_EDITMSG` 与暂存区；不会直接编辑工作区文件。
- **工作区必须干净**：压缩前必须提交或暂存所有改动，否则命令会报错退出。
- **备份建议**：
  - 首次使用建议添加 `--backup`，以防万一操作失误。
  - 备份分支命名格式：`<当前分支>-backup-<时间戳>`。
  - 恢复命令：`git reset --hard <备份分支>`。
- **同步上游的重要性**：
  - 建议先同步上游代码（不使用 `--no-sync`），避免后续 PR 合并时产生冲突。
  - 若已经同步过，可使用 `--no-sync` 跳过同步步骤。
- **压缩后的提交信息**：
  - 应准确反映所有改动的目的，而非仅描述最后一次提交。
  - 建议在消息体中列出原始提交标题，保留历史信息。
- **冲突处理**：
  - 若同步上游代码时产生冲突，命令会尝试自动解决。
  - 若自动解决失败，命令会报告冲突文件并退出，等待用户手动处理。
  - 解决冲突后，重新运行此命令即可继续压缩流程。
- **安全提示**：
  - 在 detached HEAD、rebase/merge 冲突状态下会先提示处理/确认再继续。
  - 若已经是单个提交，命令会询问用户是否继续（可能不需要压缩）。
- **可审可控**：如开启 `confirm: true`，每个实际 `git reset`/`git commit` 步骤都会进行二次确认。

---

## Edge Cases & Safety Guardrails

| 场景                       | 检测方式                                      | 处理策略                           |
| -------------------------- | --------------------------------------------- | ---------------------------------- |
| **不在 Git 仓库**          | `git rev-parse --is-inside-work-tree`         | 报错退出，提示用户初始化仓库       |
| **工作区有未提交改动**     | `git status --porcelain`                      | 报错退出，提示用户先提交或暂存     |
| **处于 detached HEAD**     | `git symbolic-ref -q HEAD`                    | 警告用户，询问是否继续             |
| **处于 rebase/merge 冲突** | 检查 `.git/MERGE_HEAD` 或 `.git/rebase-merge` | 报错退出，提示先解决冲突           |
| **上游分支不存在**         | `git rev-parse --verify <remote>/<branch>`    | 报错退出，列出可用分支             |
| **当前分支没有新提交**     | `git log <remote>/<branch>..HEAD`             | 提示用户，退出命令                 |
| **已经是单个提交**         | 提交数量 = 1                                  | 询问用户是否继续（可能不需要压缩） |
| **同步时产生冲突**         | `git merge` 返回非 0                          | 尝试自动解决，失败则报告冲突并退出 |
| **提交钩子失败**           | `git commit` 返回非 0                         | 保留暂存区状态，输出错误信息       |
| **用户中断操作**           | Ctrl+C                                        | 保留当前状态，输出中断提示         |

---

## Safety Rollback

若压缩操作失误，可通过以下方式回滚：

1. **若创建了备份分支**：

   ```bash
   git reset --hard <备份分支>
   ```

2. **若未创建备份分支，但压缩提交尚未推送**：

   ```bash
   # 查找原始提交 hash（通过 reflog）
   git reflog
   # 恢复到原始提交
   git reset --hard <原始提交 hash>
   ```

3. **若已推送到远程**：

   ```bash
   # 方式1：使用 revert（推荐，不改写历史）
   git revert <压缩提交 hash>

   # 方式2：强制回退（需要 force push，慎用）
   git reset --hard <原始提交 hash>
   git push --force
   ```

---

## FAQ

### Q1: 为什么要压缩提交？

**A**: 保持 PR 历史干净整洁，便于审阅者理解改动意图。多个零碎的提交（如"fix typo"、"format code"）会干扰审阅流程，压缩成一个有意义的提交可以提高代码审阅效率。

### Q2: 什么时候不应该压缩提交？

**A**:

- 若原提交历史对审阅有价值（如详细的实现步骤、重要的 milestone）。
- 若提交已经推送到远程且有其他人基于此分支开发（会导致历史冲突）。
- 若提交已经合并到主分支（不应改写已合并的历史）。

### Q3: 压缩后如何保留原始提交信息？

**A**: 命令会自动在提交消息体中列出原始提交标题，作为历史记录。你也可以手动编辑 `.git/COMMIT_EDITMSG` 添加更多细节。

### Q4: 压缩后可以继续开发吗？

**A**: 可以。压缩后的提交就像普通提交一样，你可以继续在此基础上开发新功能、修复 bug 等。

### Q5: 压缩后如何推送到远程？

**A**:

- 若分支**尚未推送**：直接 `git push -u origin <分支名>`。
- 若分支**已推送**：需要 **force push**（会改写远程历史）：
  ```bash
  git push --force-with-lease
  ```
  > 注意：force push 会影响其他基于此分支开发的人，使用前请确认无他人依赖此分支。

### Q6: 如何恢复压缩前的提交？

**A**:

- 若创建了备份分支：`git reset --hard <备份分支>`。
- 若未创建备份：通过 `git reflog` 查找原始提交 hash，然后 `git reset --hard <hash>`。

---

**最后更新时间**: 2026-02-26T00:00:00+00:00
