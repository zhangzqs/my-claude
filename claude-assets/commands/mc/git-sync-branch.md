---
description: 从上游主分支拉取最新代码，同步到本地主分支，再合并/变基到当前工作分支；遇冲突自动解决
allowed-tools: Bash(git fetch, git checkout, git pull, git merge, git rebase, git branch, git rev-parse, git log, git status, git diff, git remote, git stash, git stash pop, git add, git rebase --continue, git merge --continue, git symbolic-ref), Read(**), Edit(**)
argument-hint: [--remote <remote>] [--branch <branch>] [--strategy merge|rebase] [--stash] [--no-resolve]
# examples:
#   - /sync-branch                                    # 全自动：检测远程、主分支、合并策略
#   - /sync-branch --remote origin --branch main      # 指定从 origin/main 同步
#   - /sync-branch --strategy rebase                  # 使用 rebase 而非 merge
#   - /sync-branch --stash                            # 自动暂存未提交改动后再同步
#   - /sync-branch --no-resolve                       # 遇冲突时暂停，不自动解决
---

# Claude Command: Sync Branch

从上游主分支拉取最新代码 → 同步到本地主分支 → 合并/变基到当前工作分支。
遇到合并冲突时，自动分析并尝试解决。

---

## Usage

```bash
/sync-branch
/sync-branch --remote origin --branch main
/sync-branch --strategy rebase
/sync-branch --stash
/sync-branch --no-resolve
```

### Options

- `--remote <remote>`：指定远程仓库名称（如 `origin`、`upstream`）。省略则**自动检测**。
- `--branch <branch>`：指定远程主分支名称（如 `main`、`master`、`develop`）。省略则**自动检测**。
- `--strategy merge|rebase`：指定合并策略。省略则**根据项目惯例自动推断**。
- `--stash`：同步前自动 `git stash` 暂存未提交的改动，完成后 `git stash pop` 恢复。
- `--no-resolve`：遇到冲突时**不自动解决**，仅报告冲突文件列表并暂停，等待用户手动处理。

---

## What This Command Does

### 1. 环境校验（Pre-flight Checks）

- 通过 `git rev-parse --is-inside-work-tree` 确认位于 Git 仓库内。
- 检测当前是否处于 rebase/merge/cherry-pick 冲突状态。若是，提示用户先处理当前冲突。
- 记录当前工作分支名称（`git symbolic-ref --short HEAD`）。
- 若当前分支就是主分支本身，提示用户并仅执行拉取更新（跳过合并步骤）。

### 2. 参数推断（Auto-detection）

若用户未指定参数，按以下策略自动推断：

#### Remote 检测

1. 执行 `git remote` 列出所有远程仓库。
2. 若存在 `upstream`，优先使用（fork 模式常见）。
3. 若仅有 `origin`，使用 `origin`。
4. 若存在多个且无 `upstream`，列出远程列表并**询问用户**选择。

#### Branch 检测

1. 执行 `git remote show <remote>` 或 `git symbolic-ref refs/remotes/<remote>/HEAD` 获取远程默认分支。
2. 若上述失败，按优先级依次检测远程是否存在：`main` → `master` → `develop`。
3. 若仍无法确定，**询问用户**。

#### Strategy 推断

1. **优先检测 PR 场景**（最高优先级）：
   - 执行 `git rev-parse --abbrev-ref @{u}` 检查当前分支是否有上游跟踪分支。
   - 若存在远程跟踪分支（如 `origin/feature-branch`），说明分支已推送。
   - 执行 `git log @{u}..HEAD` 检查本地是否有未推送的提交。
   - 若**已推送到远程**或**本地提交数 < 10**（疑似 PR 分支） → **强制使用 merge**。
   - 理由：避免改写已推送的历史，保护 PR 审阅流程。

2. 检查 `git config pull.rebase` 的值：
   - `true` → 使用 rebase。
   - `false` 或未设置 → 使用 merge。

3. 检查最近 20 条 `git log --oneline` 中 merge commit 与线性提交的比例：
   - merge commit 占比 > 30% → 使用 merge（项目惯例）。
   - 否则 → 使用 rebase。

4. 若仍不确定，默认使用 **merge**（更安全）。

### 3. 工作区保护（Working Tree Protection）

- 执行 `git status --porcelain` 检查工作区是否有未提交改动。
- 若有未提交改动：
  - 若传入 `--stash` → 自动执行 `git stash push -m "sync-branch: auto stash before sync"`。
  - 否则 → **询问用户**：是否自动暂存、手动处理、还是中止。

### 4. 同步本地主分支（Sync Local Main）

```text
步骤流程：
  a. git fetch <remote>                          # 拉取远程最新引用
  b. git checkout <main-branch>                  # 切换到本地主分支
  c. git merge <remote>/<main-branch> --ff-only  # 快进合并远程主分支
     - 若快进失败（本地主分支有独立提交），改用 git pull --rebase 或提示用户处理
  d. git checkout <original-branch>              # 切回原工作分支
```

### 5. 合并到当前分支（Merge/Rebase into Working Branch）

根据推断的策略执行：

#### merge 策略

```bash
git merge <main-branch>
```

#### rebase 策略

```bash
git rebase <main-branch>
```

### 6. 冲突处理（Conflict Resolution）

若合并/变基产生冲突：

#### 自动解决模式（默认）

1. 执行 `git status` 和 `git diff` 识别所有冲突文件。
2. 逐文件分析冲突：
   - **读取冲突文件完整内容**，理解上下文与双方意图。
   - **分析冲突标记**（`<<<<<<<`、`=======`、`>>>>>>>`），判断：
     - 双方改动是否互不相关（可合并）。
     - 是否存在语义冲突（逻辑矛盾）。
     - 是否涉及格式化差异（可安全取一方）。
   - **生成解决方案**并通过 `Edit` 工具修改文件，移除所有冲突标记。
3. 每个文件解决后，执行 `git add <file>` 标记为已解决。
4. 所有冲突解决后：
   - merge 策略 → `git merge --continue` 或 `git commit`。
   - rebase 策略 → `git rebase --continue`（可能需要多轮，逐 commit 处理）。
5. **向用户报告**每个冲突的解决方案与理由，供审阅。

#### 手动模式（`--no-resolve`）

1. 列出所有冲突文件及其冲突片段摘要。
2. 提示用户手动解决，并给出可用命令参考：
   - `git add <file>`：标记为已解决。
   - `git merge --continue` / `git rebase --continue`：继续合并/变基。
   - `git merge --abort` / `git rebase --abort`：中止操作。

### 7. 收尾（Post-sync）

- 若步骤 3 中执行了 `git stash`，此时执行 `git stash pop` 恢复工作区。
  - 若 `stash pop` 产生冲突，同样尝试自动解决或提示用户。
- 打印同步结果摘要。

---

## Output Summary

同步完成后，打印以下摘要信息：

```text
同步完成 ✓

  远程仓库：origin
  主分支：  main
  合并策略：merge
  工作分支：feature/my-feature

  更新情况：
  - 本地 main 更新了 12 个 commit（a1b2c3d → e4f5g6h）
  - 合并到 feature/my-feature 成功（无冲突 / 已解决 N 处冲突）

  冲突解决记录（如有）：
  1. src/config.ts — 双方均修改了 DEFAULT_TIMEOUT 值，采用上游值（30s → 60s）
  2. package.json — 依赖版本冲突，合并双方新增的依赖项

  后续建议：
  - 建议运行测试确认合并后功能正常
  - 若有问题可通过 git reset --hard HEAD~1 回退合并
```

---

## Safety & Guardrails

- **不修改远程仓库**：不执行 `git push`，所有操作仅限本地。
- **不丢弃改动**：未提交改动通过 stash 保护；不执行 `git checkout .` 或 `git clean`。
- **可回退**：合并/变基失败时提供 `--abort` 指令；成功后提供回退命令参考。
- **冲突解决透明**：自动解决的每处冲突都向用户报告理由，便于审阅。
- **不修改 Git 全局配置**：不执行 `git config --global` 等操作。
- **PR 场景保护**：检测到分支已推送到远程时，**强制使用 merge** 避免改写历史。
- **rebase 安全**：若用户强制指定 `--strategy rebase` 且分支已推送，发出警告提示需要 force push 的风险。

---

## Edge Cases

| 场景                         | 处理方式                                                  |
| ---------------------------- | --------------------------------------------------------- |
| 当前分支就是主分支           | 仅执行 `git pull`，跳过合并步骤                           |
| 本地主分支不存在             | 从远程 checkout 创建本地主分支                            |
| 远程不可达（网络问题）       | 报错并建议检查网络或 remote URL                           |
| detached HEAD 状态           | 提示用户先切换到具名分支                                  |
| 未提交的 merge/rebase 进行中 | 提示用户先完成或中止当前操作                              |
| stash pop 产生冲突           | 同样尝试自动解决或提示用户                                |
| rebase 多 commit 逐个冲突    | 逐个 commit 处理，每轮 `rebase --continue`                |
| 分支已推送 + 用户指定 rebase | 警告：将改写远程历史，需 force push，询问是否继续         |
| PR 场景（分支已推送）        | 自动使用 merge，忽略用户指定的 rebase（除非明确 --force） |

---

## Important Notes

- **仅使用 Git**：不调用任何包管理器/构建命令。
- **本地操作**：不会推送到远程，不会影响他人。
- **冲突解决质量**：自动解决基于代码语义分析，但仍建议用户运行测试验证。
- **大规模冲突**：若冲突文件数 > 10 或冲突行数过多，建议用户考虑手动处理或分步合并。
