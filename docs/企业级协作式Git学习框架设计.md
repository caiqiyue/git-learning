# 企业级协作式 Git 学习框架设计

## 一、整体架构（四层递进）

```
第④层 | 治理层   → 权限模型 · 审计日志 · CHERRY-PICK 事故恢复
第③层 | 协作层   → PR/MR 流程 · CODE REVIEW · 冲突解决 · PROTECTED BRANCHES
第②层 | 分支层   → GITFLOW / GITHUB FLOW / TRUNK-BASED · FEATURE FLAG
第①层 | 基础层   → 三大区域 · RESTORE / STASH · 交互式暂存 · 工作流核心命令
```

---

## 二、第①层 基础层

### 1.1 三大区域

在企业协作中，Git 的三大区域是多人协同的"缓冲区"与"记录系统"。理解这个模型，是掌握 Git 协作逻辑的起点。

| 区域 | 说明 |
|------|------|
| 工作区（Working Directory） | 开发者直接编辑文件的区域，你本地磁盘上的代码 |
| 暂存区（Staging Area） | 通过 `git add` 准备提交的文件集合，代表"我准备好让这部分接受审查" |
| 版本库（Repository） | 提交后的版本历史存储，团队的共识来源 |

**三区域流转：**
```
工作区 → git add → 暂存区 → git commit → 版本库
```

**企业场景**：
- 工作区是"实验场"，随意试错，不影响他人
- 暂存区代表"通过自测、准备好接受 Code Review"
- 版本库是团队共享的知识记录，合并到主分支意味着通过了团队审核

### 1.2 基础命令（新版 Git 优先）

```bash
git init                              # 初始化仓库
git clone <url>                       # 克隆远程仓库
git status                            # 查看文件状态
git add <file>                        # 暂存单个文件
git add .                             # 暂存所有文件
git add -p                            # 交互式暂存（企业核心）逐块选择代码块暂存
git commit -m "feat(scope): subject"   # 生成版本
git log --oneline                     # 查看提交历史（单行）
git log --graph --oneline -20         # 图形化分支历史
```

### 1.3 新版命令（替代旧版）

| 旧命令 | 新命令 | 说明 |
|--------|--------|------|
| `git checkout -- <file>` | `git restore <file>` | 丢弃工作区修改 |
| `git checkout --staged <file>` | `git restore --staged <file>` | 取消暂存 |
| `git checkout -b <branch>` | `git switch -c <branch>` | 创建并切换分支 |
| `git checkout <branch>` | `git switch <branch>` | 切换分支 |

### 1.4 回滚操作

```bash
# 版本回滚（HEAD 指针移动）
git reset --hard <version>            # 硬回滚，代码直接回退
git reset --soft HEAD~1              # 软回滚，保留暂存区
git reset --mixed HEAD~1             # 混合回滚，默认行为

# 工作区回滚（修改但未暂存）
git restore <file>                    # 丢弃工作区修改

# 暂存区回滚（已暂存但未提交）
git restore --staged <file>          # 取消暂存

# 找回被删除的提交（reset 后找回）
git reflog                            # 查看所有操作记录（含 reset）
git reset --hard <commit-hash>        # 通过 reflog 找到的 hash 恢复
```

### 1.5 STASH 暂存

```bash
git stash -m "wip: feature/xxx - doing something"  # 暂存当前工作进度
git stash list                                       # 查看所有 stash
git stash pop                                        # 恢复最近一次 stash 并删除
git stash apply stash@{0}                            # 恢复但不删除（需要手动 drop）
git stash drop stash@{0}                             # 删除指定 stash
```

### 1.6 查询与追溯

```bash
git log -S "functionName" --oneline          # 搜索提交历史中包含特定字符串的提交
git blame -L 10,20 file.ts                  # 逐行追溯代码责任人
git show <commit-hash>                      # 查看某个提交的具体改动
git diff HEAD~5 --name-only                 # 查看最近 5 次提交改动的文件列表
```

---

## 三、第②层 分支层

### 2.0 分支原理

**核心概念：分支是指向 commit 对象的轻量指针**

Git 的分支本质上只是一个指向某个 commit 对象的可变指针（reference），它本身不存储任何文件内容。

```
commit A (abc123) <-- HEAD <-- main
commit B (def456) <-- feature
```

当你创建一个新分支时，Git 做的仅仅是创建一个新的指针，指向当前的 commit。如果当前有 5 万个文件，改动其中 10 个，Git 不会复制 5 万个文件——它只记录你改动的 10 个文件的变化。这正是 Git 轻量且高效的核心原因。

**对象存储机制：**
- Git 使用 SHA-1 内容寻址存储
- 相同文件内容只存储一份（blob 对象按 hash 去重）
- commit 对象存储树快照（文件索引），而非差异
- 新提交只存储有变更的文件，未变更的文件通过指针引用已有对象

**分支切换本质：**
- `git switch feature` 只是将 HEAD 指针指向已有 commit
- 不复制文件，不移动磁盘文件，切换极快

### 2.1 GitFlow 工作流（中小型团队）

```
                    ┌─ hotfix/xxx（从 main 切出，紧急修复）
                   ↗              ↘ merge（PR 审查后）
main ────●────────────────────────●────────────────
              ↑                   ↑ merge
           release/              develop
              ↑ merge            ↑
                   ↖───────────●──── feature/xxx（从 develop 切出）
                                 ↑
                          feature/xxx 完成，PR 合并回 develop
```

**分支职责：**

| 分支 | 命名规范 | 生命周期 | 合并目标 |
|------|---------|---------|---------|
| main | `main` | 长期 | 受保护，需 PR |
| develop | `develop` | 长期 | 受保护，需 PR |
| release | `release/x.y.z` | 短期 | 合并回 main + develop |
| feature | `feature/xxx` | 中期 | 合并回 develop |
| hotfix | `hotfix/问题描述` | 合并回 main（紧急绕过 develop）|

### 2.1.1 分支命名规范

| 类型 | 命名格式 | 示例 |
|-----|---------|------|
| 功能分支 | `feature/描述性名称` | `feature/user-auth`, `feature/payment-gateway` |
| 修复分支 | `fix/问题描述` | `fix/login-timeout`, `fix/memory-leak` |
| 热修复分支 | `hotfix/问题描述` | `hotfix/payment-crash`, `hotfix/security-patch` |
| 发布分支 | `release/x.y.z` | `release/1.2.0`, `release/2.0.0-rc1` |
| 实验分支 | `experiment/名称` | `experiment/new-algorithm` |
| 重构分支 | `refactor/描述` | `refactor/database-layer` |

> 企业实践：分支名中包含 ticket ID（如 `feature/JIRA-1234-user-auth`）便于追溯。
> 生产环境必须对 main/develop 设置 Protected Branches，详见 4.3。

### 2.2 Trunk-Based（DevOps 成熟团队）

- 所有开发在 `main` 或 ≤ 2 天的短期分支内完成
- `feature/xxx` → 代码审查 → 直接合并回 `main`
- 适合小团队（≤ 10 人）、CI/CD 成熟的组织

### 2.3 分支操作核心命令

```bash
git branch                              # 查看本地分支
git branch -a                           # 查看所有分支（含远程）
git switch -c feature/xxx               # 创建并切换（新命令）
git switch main                         # 切回主分支
git push -u origin feature/xxx          # 首次推送并设置上游分支
git fetch --all                         # 拉取所有远程更新（不合并）
git pull --rebase origin develop        # 拉取并变基（保持线性历史）
git merge <branch>                      # 合并分支
git rebase <branch>                     # 变基（重写提交历史）
git branch -d <branch>                 # 删除已合并分支
git branch -D <branch>                  # 强制删除分支
```

### 2.4 新版 FETCH / PULL / REBASE 组合

```bash
git fetch origin develop                # 仅拉取引用，不合并
git log origin/develop --oneline       # 查看远程分支提交
git diff develop..origin/develop       # 对比本地与远程差异

git pull --rebase origin develop        # 拉取并变基（避免分叉）
# 等价于：git fetch + git rebase

# pull 但不用 rebase（会产生分叉）
git pull origin develop                 # = git fetch + git merge
```

---

## 四、第③层 协作层（PR/MR 全流程）

### 4.1 完整 PR/MR 协作流程

```
┌─────────────────────────────────────────────────────────────────┐
│                     PR / MR 协作全流程                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  开发者A                    平台（GitHub/GitLab）           开发者B│
│  ─────────                  ────────────────────         ─────────│
│                                                                 │
│  1. git switch -c feature/payment                             │
│  2. 编码，git add -p / git commit -m "feat(payment): ..."     │
│  3. git push -u origin feature/payment  ──── push ───→       │
│                                                                 │
│                              [创建 PR/MR]                        │
│                              [指定 Reviewers]                   │
│                              [关联 Issue]                        │
│                              [设置 labels / milestone]           │
│                                                                 │
│                              PR 可合并？                         │
│                                  ↓ 是                           │
│                        开发者B 进行 Code Review                  │
│                        [Approve / Request Changes / Comment]   │
│                                                                 │
│                        [通过审查，Merge（ Squash or Merge ）]   │
│                                                               → │
│                                                                 │
│  4. git switch develop                                        │
│  5. git pull --rebase origin develop                           │
│  （同步最新代码，继续下一轮开发）                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 CODE REVIEW 辅助命令

```bash
# 查看他人分支
git fetch origin
git switch -t origin/feature/payment    # 创建跟踪远程分支的本地分支

# 查看差异
git diff main..origin/feature/payment   # 对比 main 与 PR 分支差异
git diff --stat origin/main..HEAD       # 查看改动统计（多少文件改动）

# 查看特定提交
git show <commit-hash>                  # 查看某个提交的具体改动
git log origin/feature/payment --oneline  # 查看他人分支提交历史

# 本地测试他人分支代码
git switch -c test-remote origin/feature/payment
# ... 测试功能 ...
# 测试完毕，切换回原分支，删除测试分支
git switch my-branch && git branch -D test-remote
```

### 4.3 PROTECTED BRANCHES（企业必设）

| 保护规则 | 说明 |
|---------|------|
| 禁止直接 push | 必须通过 PR 合并 |
| 至少 N 个 Approve | 通常要求 1-2 人审查通过 |
| CI 必须通过 | 测试、构建必须 green |
| 冲突必须解决 | 合入前必须 rebase 或手动解决冲突 |
| Linear history（可选） | 禁止 merge commit，强制 squash |

**GitHub CLI 配置示例：**

```bash
gh api repos/org/repo/branches/main/protection -X PUT \
  -f required_pull_request_reviews=2 \
  -f enforce_admins=true \
  -f required_status_checks='{"strict":true,"contexts":["ci/pipeline"]}'
```

### 4.4 冲突解决策略

| 场景 | 策略 | 命令 | 结果 |
|------|------|------|------|
| 保持线性历史 | rebase | `git pull --rebase` | 提交历史干净，但会改写 hash |
| 多人同时开发 | merge | `git pull` | 保留真实合并过程，有分叉 |
| 精确摘取提交 | cherry-pick | `git cherry-pick <hash>` | 独立摘取，不影响原分支 |

**rebase 与 merge 的选择原则：**
- **公共分支（main/develop）**：用 merge，保留真实历史
- **个人功能分支**：用 rebase，保持线性历史
- **绝对不要**：在已 push 到远程的分支上做 rebase（除非确定只有你一个人在用）

---

## 五、第④层 治理层

### 5.1 权限模型

```bash
# 企业角色与权限对照表

| 角色 | 权限范围 |
|------|---------|
| Owner | 全部权限，包含删除仓库、修改仓库设置 |
| Maintainer | 管理分支保护、代码审查规则，不可删除仓库 |
| Developer | 推送自己的分支、创建 PR、评论 |
| Reporter | 仅读权限（外部协作者） |
| Guest | 仅读 Issue，不可看代码（可选，GitLab 支持，GitHub 最低权限为 Reporter） |
```

### 5.2 CHERRY-PICK 事故恢复（企业核心）

```bash
# 摘取特定提交到当前分支（不改变原分支）
git cherry-pick <commit-hash>             # 摘取单个提交
git cherry-pick -x <commit-hash>          # 摘取并在 footer 记录原始 commit
git cherry-pick <hash1> <hash2> <hash3>   # 摘取多个提交

# 摘取但暂不提交（用于手动筛选改动）
git cherry-pick --no-commit <commit-hash>

# 在 cherry-pick 过程中遇到冲突
git add .
git cherry-pick --continue                # 继续摘取流程
git cherry-pick --abort                   # 取消整个 cherry-pick，恢复原状
```

### 5.3 REFLOG 事故恢复（救命命令）

```bash
git reflog                                # 列出所有操作历史
# 输出示例：
# a1b2c3d HEAD@{0}: reset: moving to HEAD~3
# b2c3d4e HEAD@{1}: commit: feat(payment): add payment service
# c3d4e5f HEAD@{2}: checkout: switching to feature/payment

# 误删分支后恢复
git reflog
# 找到分支最后一次 commit 的 hash（如 b2c3d4e）
git switch -c feature/restored b2c3d4e    # 从该 hash 重建分支

# 误 reset 后恢复
git reset --hard HEAD@{1}                  # 恢复到上一个位置
```

### 5.4 TAG 版本管理

```bash
git tag -a v1.0.0 -m "release: first production release"  # 创建附注标签
git tag v1.0.1 <commit-hash>                           # 在特定提交上打标签
git push origin --tags                                  # 推送所有标签
git push origin v2.0.0                                  # 推送单个标签
git tag                                                 # 查看本地标签
git tag -d v1.0.0                                       # 删除本地标签
git ls-remote --tags origin                             # 查看远程标签
git checkout v2.0.0                                    # 拉取指定版本（进入 detached HEAD）
```

### 5.5 审计与统计

```bash
git log --author="zhangsan@company.com"   # 按作者筛选提交
git log --since="2024-01-01"             # 按时间筛选提交
git log --until="2024-12-31"             # 截止时间筛选
git log --grep="fix:"                     # 按提交 message 关键词筛选
git shortlog -sn                          # 贡献者统计（提交数排行）
git shortlog -sn --after="2024-01-01"    # 按时段统计贡献者
git log --oneline --since="2024-01-01" | wc -l  # 统计一段时间的提交数
```

### 5.6 BISECT 二分查找问题提交

```bash
git bisect start                          # 开始二分查找
git bisect bad                            # 标记当前版本有 bug
git bisect good v2.0.0                    # 标记某个版本是好的

# Git 自动 checkout 中间版本，测试是否为 bug 版本
# 测试后：
git bisect good                           # 此版本正常
# 或
git bisect bad                            # 此版本有问题

# 重复直到找到问题提交
git bisect reset                          # 结束二分查找，恢复 HEAD
```

---

## 六、COMMIT MESSAGE 企业规范（贯穿层）

### 6.1 格式规范（Conventional Commits）

```
<type>(<scope>): <subject>

[optional body]

[optional footer - BREAKING CHANGE 或关联 Issue]
```

**第一行规则：**
- 最多 72 字符
- 用**现在时**："add" 不是 "added"，"fix" 不是 "fixed"
- 句尾不加句号
- 标题即结论，不需要完整句子

### 6.2 TYPE 完整列表

| Type | 何时使用 | 示例 |
|------|---------|------|
| `feat` | 新功能（用户可见） | `feat(payment): add WeChat pay integration` |
| `fix` | 修复 bug | `fix(auth): resolve JWT token expiry edge case` |
| `refactor` | 重构（不改行为） | `refactor(user): extract validation to utility` |
| `perf` | 性能优化 | `perf(query): add index on orders.user_id` |
| `test` | 新增/修复测试 | `test(order): add unit test for discount calc` |
| `docs` | 文档变更 | `docs(api): update user endpoint signature` |
| `style` | 格式（不影响功能） | `style(format): format with prettier` |
| `build` | 构建 / 依赖变更 | `build(deps): upgrade react to 18.2` |
| `ci` | CI 配置变更 | `ci: add SonarQube stage to pipeline` |
| `chore` | 杂项（工具/配置） | `chore: update .gitignore` |
| `revert` | 回退提交 | `revert: revert feat/payment due to regression` |

### 6.3 SCOPE 规范

Scope 是**受影响模块/包/服务名**：

```
feat(user): add password reset flow          ✓
feat(auth): add password reset flow          ✓
feat: add password reset flow                ✗ 太泛

feat(order-service): add bulk export API     ✓（微服务场景）
feat(shared-lib): add HttpClient wrapper    ✓（公共库场景）
```

### 6.4 BODY 与 FOOTER

**Body（复杂改动必须写）：**

```
feat(payment): support split payment for orders over 1000

- Add PaymentStrategy interface
- Implement AlipayStrategy and WechatStrategy
- Refactor PaymentController to use strategy pattern
- Add integration tests for each payment gateway

Closes #234
```

**Footer — BREAKING CHANGE：**

```
refactor(api): change /user response structure

BREAKING CHANGE: /user endpoint now returns {data: User} instead of User directly.
All clients must update their parsers before upgrading.

Closes #199
```

### 6.5 COMMITLINT CI 自动检查

```yaml
# .commitlintrc.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [2, 'always', ['feat', 'fix', 'refactor', 'perf', 'test', 'docs', 'style', 'build', 'ci', 'chore', 'revert']],
    'scope-case': [2, 'always', 'lower-case'],
    'subject-full-stop': [2, 'never', '.'],
    'subject-max-length': [2, 'always', 72]
  }
};
```

---

## 七、场景库（企业真实场景）

### 场景 1：项目初始化 + 分支建立

**背景：** Lead 张三创建电商平台仓库，配置 protected branches

```bash
# 张三初始化项目
git clone https://github.com/org/ecommerce-platform.git
cd ecommerce-platform

# 创建 develop 分支（开发主干）
git switch -c develop
git push -u origin develop

# 配置 branch protection rules
gh api repos/org/ecommerce-platform/branches/main/protection \
  -X PUT -f required_pull_request_reviews=1 -f enforce_admins=true
```

### 场景 2：功能开发完整流程

**背景：** 李四开发用户认证功能，从创建分支到 PR 合并

```bash
# 1. 同步 develop 最新代码
git switch develop
git pull --rebase origin develop

# 2. 创建功能分支
git switch -c feature/user-auth

# 3. 编码并暂存（交互式暂存，企业好习惯）
git add -p                    # 逐块确认暂存
git commit -m "feat(auth): add JWT authentication

- Add JWT token generation and validation
- Implement token refresh mechanism
- Add /auth/login and /auth/refresh endpoints

Closes #42"

# 4. 首次推送（GitHub 自动提示创建 PR）
git push -u origin feature/user-auth

# 5. PR 合并后，删分支并同步 develop
git switch develop
git pull --rebase origin develop
git branch -d feature/user-auth
```

### 场景 3：Code Review 完整流程（Reviewer 视角）

**背景：** 王五被指定为 `feature/user-auth` 的 Reviewer

```bash
# 1. 获取 PR 分支
git fetch origin
git switch -t origin/feature/user-auth    # 创建跟踪分支

# 2. 查看提交历史和改动
git log main..feature/user-auth --oneline
git diff main..feature/user-auth

# 3. 逐文件、逐行审查
git show <commit-hash>                   # 查看单个提交的改动

# 4. 在 GitHub 上提交 Review（Approve / Request Changes / Comment）
# 5. 测试（可选）
git switch -c test-pr origin/feature/user-auth
# ... 测试 ...
git switch - && git branch -D test-pr
```

### 场景 4：合并后同步最新代码

**背景：** 赵六的 `feature/report` 分支已合并，他要继续开发下一个功能

```bash
git switch develop
git pull --rebase origin develop

# 开始新功能
git switch -c feature/new-feature
```

### 场景 5：处理多人同时开发的冲突

**背景：** 赵六和王五同时在 `feature/payment` 分支开发，赵六先合并，导致王五需要 rebase

```bash
# 王五的操作
git status
# 输出：Your branch and 'origin/feature/payment' have diverged

# 先暂存工作
git stash -m "wip: payment integration"

# 强制更新到远程最新
git fetch origin
git reset --hard origin/feature/payment

# 恢复工作（可能有冲突需要解决）
git stash pop
git add -p
git commit -m "feat(payment): complete payment integration"
git push --force-with-lease origin feature/payment
```

### 场景 6：hotfix 紧急修复

**背景：** 生产环境出现严重 bug，必须立即修复

```bash
# 从 main 创建 hotfix（绕过 develop）
git switch main
git pull --rebase origin main
git switch -c hotfix/critical-bug-fix

# 修复并提交
git add -p
git commit -m "fix!: critical payment null pointer

BREAKING CHANGE: This fix changes the payment response structure.

Closes #119"

# 创建 PR 到 main（设置 label: hotfix，优先审查）
git push -u origin hotfix/critical-bug-fix
```

### 场景 7：功能暂停，插入紧急任务

**背景：** 李四正在 `feature/payment` 分支开发 60%，突然需要修一个紧急 bug

```bash
# 1. 暂存当前工作
git add -A
git stash push -m "wip: feature/payment - 60% done, payment controller"

# 2. 记录当前位置
git log --oneline -1
# 输出：a1b2c3d feat(payment): add payment service skeleton

# 3. 切到 develop 创建 hotfix
git switch develop
git switch -c hotfix/urgent-bug-fix
git add -p
git commit -m "fix(cache): resolve Redis session leak

Closes #119"
git push -u origin hotfix/urgent-bug-fix

# 4. hotfix PR 合并后，恢复 payment 开发
git switch feature/payment
git stash pop
# 如果有冲突，解决后继续开发
```

### 场景 8：cherry-pick 部分提交

**背景：** 王五在 `feature/report-export` 分支实现了一个通用工具，张三想只摘取其中的工具类合入 `feature/dashboard`

```bash
# 找到要摘取的 commit
git log origin/feature/report-export --oneline
# 输出：
# e5f6a7b feat(report): add Excel export with dynamic columns
# c3d4e5f refactor(report): extract ExportStrategy interface

# 切换到自己的分支
git switch feature/dashboard

# 摘取 interface 那个提交（不是全量）
git cherry-pick -x c3d4e5f

# 如果有冲突，解决后：
git add .
git cherry-pick --continue
```

### 场景 9：误删分支恢复

**背景：** 赵六不小心删除了本地 `feature/invoice生成` 分支，但远程还在

```bash
# 立即用 reflog 找到分支的最后一次 commit
git reflog
# 输出：
# a1b2c3d HEAD@{0}: checkout: switching to feature/invoice生成
# b2c3d4e HEAD@{1}: commit: feat(invoice): add PDF generation

# 从那个 commit 重建分支
git switch -c feature/invoice生成 b2c3d4e

# 推送回远程
git push -u origin feature/invoice生成
```

### 场景 10：rebase 冲突中断抢救

**背景：** 赵六执行 `git rebase develop` 后遇到冲突，不知道下一步

```bash
# 1. 解决冲突文件（VSCode 会标注）
# 编辑文件，删除 <<<<<<< ======= >>>>>>> 标记

# 2. 暂存解决后的文件
git add <resolved-file>

# 3. 继续 rebase
git rebase --continue

# 如果 rebase 进行不下去，想撤销：
git rebase --abort

# 想修改某个提交的消息：
git rebase -i HEAD~5
# 把要修改的 commit 前的 pick 改成 edit
git commit --amend
git rebase --continue
```

### 场景 11：Fork 工作流（跨团队外部贡献）

**背景：** 外部开发者向公司公共组件库贡献代码，没有直接 push 权限

```bash
# 1. Fork 仓库到自己的账号

# 2. 克隆自己 fork 的仓库
git clone https://github.com/zhangsan/ui-components.git
cd ui-components

# 3. 添加上游仓库
git remote add upstream https://github.com/company/ui-components.git
git remote -v

# 4. 保持 fork 与上游同步
git fetch upstream
git switch main
git merge upstream/main
git push origin main

# 5. 开发功能分支
git switch -c feature/new-button
git add -p
git commit -m "feat(ui): add Button component with loading state"
git push origin feature/new-button

# 6. 创建 Pull Request（目标选 company/ui-components:main）

# 7. 如果维护者要求修改：
git add -p
git commit --amend
git push --force-with-lease origin feature/new-button

# 8. PR 合并后，同步上游
git switch main
git pull --rebase upstream main
git push origin main
git branch -d feature/new-button
```

### 场景 12：使用 Beyond Compare 解决冲突

**背景：** 大型项目冲突较多，需要可视化工具辅助

```bash
# 1. 下载并安装 Beyond Compare

# 2. 配置 Git 使用 Beyond Compare 作为合并工具
git config --local merge.tool bc3
git config --local mergetool.path '/usr/local/bin/bcomp'
git config --local mergetool.keepBackup false

# 3. 遇到冲突时，调用合并工具
git mergetool

# Beyond Compare 会打开三路合并视图：
# 左侧：本地版本（ours）
# 中间：基础版本（base）
# 右侧：远程版本（theirs）
# 选择或手动编辑最终结果
```

---

## 八、命令速查表（企业高频）

### 日常操作（每天用）

```bash
git switch -c <branch>              # 创建并切换分支（新命令）
git add -p                           # 交互式暂存（企业必学）
git restore <file>                   # 丢弃工作区修改（新命令）
git restore --staged <file>          # 取消暂存（新命令）
git pull --rebase origin dev         # 拉取并变基
git status                           # 查看状态
git log --oneline -10                # 查看最近 10 条提交
```

### 分支协作（每天用）

```bash
git branch -a                        # 查看所有分支
git push -u origin <branch>           # 首次推送并跟踪
git push --force-with-lease          # 安全 force push
git fetch --all                      # 拉取所有远程引用
git diff main..<branch>              # 查看 PR 分支差异
```

### PR / Code Review（高频）

```bash
git fetch origin <branch>            # 获取他人分支
git log main..<branch>                # 查看 PR 分支提交历史
git show <commit-hash>               # 查看某个提交的具体改动
git cherry-pick <hash>                # 摘取特定提交
```

### 事故恢复（关键时刻用）

```bash
git reflog                           # 找回所有操作
git reset --hard ORIG_HEAD           # 恢复到最后正确状态
git bisect start                     # 二分查找问题提交
git stash                            # 暂存工作进度
```

### 团队治理（偶尔用）

```bash
git tag -a v2.0.0 -m ""              # 创建版本标签
git push origin --tags              # 推送所有标签
git shortlog -sn                     # 贡献者统计
git log --author="name"              # 按作者筛选提交
git blame <file>                     # 逐行代码责任人
```