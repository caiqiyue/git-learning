# 22. Git 工作树（Worktree）

## 本节目标

学完这节后，你会：
- 理解 Git Worktree 的概念
- 掌握基本命令：add、list、remove、prune
- 了解常见使用场景
- 理解注意事项和限制
- 对比其他方案（stash、fork、多个仓库）

---

## Git Worktree 是什么

Git Worktree 允许我们**同时在同一个仓库的不同分支上工作**。每个 worktree 都是仓库的一个独立工作目录，它们共享同一个 Git 数据库。

### 基本概念

在没有 worktree 之前，如果我们同时需要在两个分支上工作，必须：
- 切换分支（`git checkout`）
- 或者用 `git stash` 暂存当前更改

有了 worktree，我们可以**同时**打开多个工作目录，每个目录对应不同的分支。

```
传统方式（单工作目录）：
  main ──●──●  (正在开发 feature-A)
           └─ 需要同时处理 feature-B 怎么办？
              → 必须来回切换或暂存

Worktree 方式（多工作目录）：
  main ──●──●  (worktree-1 正在开发 feature-A)
        │
        └─●──●  (worktree-2 正在开发 feature-B)
              ↑
              两个目录同时工作，互不干扰
```

### 工作原理

```bash
# 查看仓库目录结构
my-project/
├── .git/
│   └── objects/          # 共享的 Git 对象数据库
├── worktree-feature-A/   # worktree 1（main 分支）
│   └── .git              # 指向主仓库的 .git
└── worktree-feature-B/    # worktree 2（feature-B 分支）
    └── .git              # 同样是文件引用，指向主仓库
```

实际上，worktree 的 `.git` 是一个文件，不是目录：

```bash
# worktree 中的 .git 内容
gitdir: /path/to/main/repo/.git/worktrees/worktree-name
```

### 解决的问题

| 问题 | 没有 Worktree | 有 Worktree |
|------|-------------|-------------|
| 同时处理两个分支 | 必须来回切换 | 两个目录同时打开 |
| 工作进度冲突 | 必须 stash | 各分支独立保留 |
| Code Review 准备 | 切换分支 | 保持分支同时准备 PR |
| 紧急修复冲突 | 先 stash 再切 | 直接切新分支 |

---

## 为什么需要 Worktree（对比 stash）

### Stash 的局限性

`git stash` 是一个临时存储区，但它有一些限制：

| 局限性 | 说明 |
|------|------|
| 只适合临时场景 | stash 久了容易被遗忘 |
| 只能有一个 stash | 多任务时需要多个 stash |
| 上下文丢失 | stash 后再 pop容易丢失修改 |
| 难以协作 | stash 是本地操作，不共享 |

### Worktree 的优势

```
Worktree 的价值：
  1. 同时打开多个分支
  2. 每个分支的工作独立保留
  3. 不需要来回切换
  4. IDE 可以同时打开多个目录
  5. 工作进度清晰可见
```

### 典型对比场景

**场景：在开发 feature-A 时，收到 PR review 请求**

**用 Stash：**
```bash
# 暂存当前修改
git stash
# 切换到 PR 分支
git checkout pr-branch
# review ...
# review 完成后
git checkout feature-A
git stash pop
# 继续开发
```

**用 Worktree：**
```bash
# 在新目录创建 worktree
git worktree add ../review-pr pr-branch
# 现在有两个目录：
# - my-project/ (feature-A 开发中)
# - review-pr/   (PR review 中)
# review 完成后
git worktree remove ../review-pr
# 继续开发，状态完全保留
```

**注意：** worktree 比 stash 更适合需要频繁切换的场景。stash 是临时解决方案，worktree 是持久解决方案。

---

## 基本命令

### git worktree add

创建新的 worktree。

#### 基本语法

```bash
# 基于当前分支创建 worktree
git worktree add ../new-feature

# 基于指定分支创建 worktree
git worktree add ../hotfix-release origin/hotfix

# 创建新分支并创建 worktree
git worktree add -b feature/new-ui ../feature-ui
# -b: 创建新分支

# 从特定 commit 创建 worktree
git worktree add ../old-version v1.0.0
# 基于标签 v1.0.0 创建 worktree
```

#### 工作原理

```bash
# 创建 worktree
git worktree add ../worktree-feature

# Git 会：
# 1. 创建一个新目录 ../worktree-feature
# 2. 切换到指定分支（默认是当前分支）
# 3. 在新目录 checkout 所有文件
# 4. 在主仓库的 .git/worktrees/ 中注册
```

#### 常用选项

```bash
# -b <branch>: 创建新分支
git worktree add -b feature/login ../feature-login

# -B <branch>: 如果分支已存在，强制切换到这个分支
git worktree add -B main ../checkout-main

# --no-checkout: 创建但不 checkout（目录为空）
git worktree add --no-checkout ../empty-worktree

# --track: 跟踪远程分支
git worktree add -t origin/develop ../develop-worktree
```

### git worktree list

列出所有 worktree。

```bash
git worktree list
# 输出格式：
# /path/to/main/repo          abc1234 [main]
# /path/to/worktree-feature   def5678 [feature-A]
# /path/to/review-pr         ghi9012 [pr-branch]

# 每个 worktree 显示：
# - 路径
# - 所在的 commit hash
# - 当前分支名
```

### git worktree remove

删除 worktree。

```bash
# 删除 worktree（如果工作区干净）
git worktree remove ../worktree-feature

# 强制删除（有未提交的更改）
git worktree remove --force ../worktree-feature
# --force: 即使有未合并的更改也删除

# 删除但不删除工作目录（只取消注册）
git worktree remove --keep-working-tree ../worktree-feature
```

**注意：** 删除 worktree 前请确保工作已经提交或合并。强制删除可能导致未保存的工作丢失。

### git worktree prune

清理已失效的 worktree 引用。

```bash
# 列出将要清理的内容（不实际清理）
git worktree prune --dry-run

# 执行清理
git worktree prune

# 常用场景：
# - 手动删除了 worktree 目录后
# - .git/worktrees 中的引用损坏时
```

### 其他相关命令

```bash
# 移动 worktree（Git 2.17+）
git worktree move ../old-path ../new-path

# 查看 worktree 详情
git worktree list --verbose
# 显示更多信息

# 锁定 worktree（防止被删除）
git worktree lock ../worktree-feature

# 解锁 worktree
git worktree unlock ../worktree-feature
```

---

## 常见使用场景

### 场景 1：同时处理功能开发和 PR Review

**场景描述：**
- 我正在开发 `feature/payment`
- 同时收到同事的 PR 需要 review
- 不想暂停当前开发

**解决方案：**

```bash
# 1. 首先确保当前工作已提交或暂存
git add .
git commit -m "WIP: working on payment feature"

# 2. 创建 PR 的 worktree
git worktree add ../review-user-auth origin/feature/user-auth

# 3. 在新目录进行 review
cd ../review-user-auth
# 阅读代码、测试、提交 review 意见

# 4. review 完成后
cd ../feature/payment
git worktree remove ../review-user-auth
# 继续开发 payment 功能
```

### 场景 2：并行处理多个特性分支

**场景描述：**
- 需要同时在 `feature/A` 和 `feature/B` 上工作
- 两个功能有依赖关系，需要对照
- 不确定哪个先完成

**解决方案：**

```bash
# 主仓库：feature/A
git worktree add ../feature-A-wt feature/A

# 新 worktree：feature/B
git worktree add ../feature-B-wt feature/B

# 现在有三个工作目录：
# - my-project/       (可能用于协调或 main)
# - feature-A-wt/     (feature A)
# - feature-B-wt/     (feature B)

# 可以同时打开两个 IDE 窗口对照
```

### 场景 3：紧急修复时快速切换

**场景描述：**
- 正在开发新功能（未完成）
- 线上出现紧急 bug 需要立即修复
- 不想丢失当前开发进度

**解决方案：**

```bash
# 1. 当前 worktree（未完成开发）
# 不用 stash，直接创建新 worktree

# 2. 基于 main 创建 hotfix worktree
git worktree add ../hotfix-critical -b hotfix/critical-bug origin/main

# 3. 在 hotfix worktree 修复 bug
cd ../hotfix-critical
# 修复、测试、提交
git commit -m "fix: resolve critical payment bug"
git push origin hotfix/critical-bug

# 4. 创建 PR 合并
# ...

# 5. 回到原 worktree 继续开发
cd ../my-project
# 继续之前的未完成工作
```

**注意：** 这个场景体现了 worktree 的核心价值——不用stash，不用中断当前工作，直接开一个新分支处理紧急任务。

### 场景 4：准备 Release 文档

**场景描述：**
- 需要对照两个版本写文档
- 确认两个版本之间的差异

**解决方案：**

```bash
# 为两个版本创建 worktree
git worktree add ../v1.0.0-docs v1.0.0
git worktree add ../v1.1.0-docs v1.1.0

# 两个目录可以同时打开
# 对照差异，写文档
```

### 场景 5：在不同分支间对比测试

**场景描述：**
- 在 `main` 分支测试正常
- 但 `feature/new-auth` 分支的测试失败了
- 需要同时运行两个测试进行对比

**解决方案：**

```bash
# 主仓库测试
cd my-project
npm test

# feature 分支创建 worktree 测试
git worktree add ../test-feature feature/new-auth
cd ../test-feature
npm test
# 对比结果
```

---

## 注意事项

### 同一个分支不能在多个 worktree 中同时检出

**这是 Git 的硬性限制。**

```bash
# 假设 main 分支已经在主仓库 worktree 中检出
cd my-project && git checkout main

# 尝试再基于 main 创建 worktree
git worktree add ../another-main main
# 报错：
# fatal: 'main' is already being worked on in worktree at '/path/to/my-project'
```

**解决方案：**
```bash
# 如果确实需要两个 main 工作目录
# 1. 在主仓库创建新的分支
git worktree add ../main-copy -b main-copy

# 或者
# 2. 在主仓库创建一个临时分支
git branch temp-main-backup
git worktree add ../another-main temp-main-backup
```

### 清理不再需要的 worktree

**建议：** 完成工作后及时清理 worktree。

```bash
# 列出所有 worktree
git worktree list

# 查看哪些 worktree 的分支已被合并
git branch --merged

# 删除已经合并的 worktree
git worktree remove ../merged-feature-worktree

# 批量清理已失效的 worktree（如果手动删除了目录）
git worktree prune
```

### 共享 Git 对象

**重要：** 所有 worktree 共享同一个 `.git` 目录。这意味着：
- 在一个 worktree 中的提交在其他 worktree 中立即可见
- `git push` 在任一 worktree 中都可以推送
- 但 `git pull` 只更新当前 worktree 的分支

### 对主仓库的影响

```bash
# 在 worktree 中提交
cd ../feature-worktree
git commit -m "feat: add new feature"

# 在主仓库或其他 worktree 中立即可以看到
cd ../main-repo
git log --oneline
# 会显示新的提交
```

### 跨平台注意

| 场景 | 说明 |
|------|------|
| Windows 路径 | worktree 路径使用反斜杠或正斜杠均可 |
| 路径长度 | Windows 路径长度有限制，worktree 过深可能导致问题 |
| 权限问题 | Windows 上确保对 worktree 目录有写入权限 |
| 符号链接 | 建议使用相对路径 |

```bash
# 跨平台建议：使用正斜杠路径
git worktree add ../my-worktree feature/my-branch

# Windows 上也可以工作
# Git 会自动处理路径分隔符
```

---

## 对比其他方案

### Worktree vs Stash

| 特性 | Worktree | Stash |
|------|----------|-------|
| 持久性 | 持久保存 | 临时存储 |
| 多任务 | 支持多个独立 worktree | 多个 stash 但难管理 |
| 可见性 | 工作状态清晰可见 | 隐藏在工作栈中 |
| 共享 | 不共享，各 worktree 独立 | 不共享，本地操作 |
| 适用场景 | 长时间多任务 | 临时切换 |
| 协作 | 不适合（因为是本地） | 不适合 |

**选择建议：**
- 临时切换 → stash
- 长期多任务 → worktree

### Worktree vs Fork

| 特性 | Worktree | Fork |
|------|----------|------|
| 仓库关系 | 同一仓库的不同分支 | 独立仓库，fork 关系 |
| 分支共享 | 分支在仓库内共享 | 每个 fork 有独立分支 |
| 协作方式 | 通常单人多分支 | 多人多仓库 |
| 合并方式 | git merge | pull request |
| 权限 | 同一权限 | 独立权限 |
| 适用场景 | 个人多任务 | 团队协作 |

**选择建议：**
- 个人多分支 → worktree
- 团队隔离开发 → fork
- 团队同一仓库协作 → 分支 + PR

### Worktree vs 多个仓库克隆

| 特性 | Worktree | 多个仓库克隆 |
|------|----------|-------------|
| Git 对象 | 共享，节省空间 | 各自独立，占用空间 |
| 分支操作 | 可以 push/pull | 各自独立 |
| 提交同步 | 提交立即同步 | 需要 fetch |
| 维护成本 | 低（自动同步） | 高（需要手动维护多个仓库） |
| 适用场景 | 个人多分支 | 完全隔离的开发环境 |

**选择建议：**
- 同一项目多分支 → worktree
- 完全独立开发环境 → 多个克隆

### 方案对比总结

```
何时用 Worktree：
  ✓ 同时处理多个分支
  ✓ 需要同时打开多个 IDE 窗口
  ✓ 不想用 stash
  ✓ 需要保持工作进度清晰

何时用 Stash：
  ✓ 临时切换去做别的事
  ✓ 工作未完成无法提交
  ✓ 一次性切换

何时用 Fork：
  ✓ 需要团队隔离开发
  ✓ 贡献到别人的仓库
  ✓ 需要独立权限

何时用多个仓库克隆：
  ✓ 需要完全独立的环境
  ✓ 避免操作失误影响其他分支
  ✓ 实验性完全破坏性操作
```

---

## 常见问题

**Q：一个仓库可以创建多少个 worktree？**
A：理论上没有限制，但建议保持合理的数量以便于管理。

**Q：worktree 中的提交会影响主仓库吗？**
A：会的。所有 worktree 共享同一个 Git 数据库，提交在仓库内是共享的。

**Q：可以删除正在使用的 worktree 吗？**
A：可以，但会导致未保存的工作丢失。确保所有更改已提交。

**Q：worktree 可以嵌套吗？**
A：不可以。worktree 不能在另一个 worktree 内创建。

**Q：如何批量清理所有 worktree？**
A：需要逐个删除，没有一键清理的命令。可以先列出，然后用脚本批量删除。

**Q：worktree 支持 bare repository 吗？**
A：不完全支持。bare repository 本身不是用来 checkout 的，所以不适用于 worktree。

**Q：在 worktree 中 git status 会不会显示其他 worktree 的状态？**
A：不会。每个 worktree 独立维护自己的暂存区和工作区状态。

---

## 本节总结

```
Worktree 概念：
  - 同一仓库的多个工作目录
  - 共享 Git 对象数据库
  - 每个 worktree 可以检出不同分支

基本命令：
  git worktree add <path> [-b <branch>]    # 创建 worktree
  git worktree list                        # 列出所有 worktree
  git worktree remove <path> [--force]     # 删除 worktree
  git worktree prune                       # 清理失效引用

使用场景：
  - 同时处理功能开发和 PR review
  - 并行处理多个特性分支
  - 紧急修复时快速切换
  - 不同分支间的对比测试

重要限制：
  - 同一分支不能在多个 worktree 中同时检出
  - 删除前请确保工作已提交

对比其他方案：
  - Stash：临时切换用 stash，长期多任务用 worktree
  - Fork：团队协作用 fork，个人多分支用 worktree
  - 多仓库克隆：完全隔离用多仓库，否则用 worktree
```

**企业实践：**
1. 当需要同时处理多个分支时，优先考虑 worktree
2. 及时清理已完成的 worktree
3. 在团队中推广 worktree 的使用，减少 stash 的滥用
4. 建立 worktree 命名规范（如 `wt/feature-name`）
5. 在 CI/CD 中注意处理 worktree 场景