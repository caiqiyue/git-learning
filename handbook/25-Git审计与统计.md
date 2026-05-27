# 25. Git 审计与统计

## 本节目标

学完这节后，你会：
- 使用 git log 高级用法查看提交历史
- 生成各种维度的代码统计报告
- 使用 git blame 和 git bisect 追查问题
- 理解 GitHub/GitLab 内置的统计功能
- 掌握团队协作中的审计技巧
- 快速查询常用审计命令

---

## 为什么需要 Git 审计

### 审计的重要性

**没有审计的项目：**

```
某天线上出了 Bug
        ↓
查了一圈不知道是谁改的
        ↓
花了 2 小时定位到具体的 commit
        ↓
发现是 3 周前的某个 "update" 提交
```

**有审计意识的项目：**

```
代码改动都留有痕迹：
- 每次提交都有清晰的 message
- 可以追溯到具体的作者和时间
- 可以统计每个模块的改动量
- 可以分析团队的贡献分布
```

### 审计能解决的问题

- 代码出现问题时，快速定位责任人
- 统计团队成员的贡献度
- 分析代码质量趋势
- 满足合规要求（SOX、GDPR 等）
- 追查安全事件

---

## 查看提交历史

### git log 基础回顾

```bash
# 查看完整历史
git log

# 简洁单行格式（日常最常用）
git log --oneline
```

### 格式化输出

```bash
# 自定义格式（适合脚本处理）
git log --format="%H|%an|%ae|%ad|%s" --date=iso

# 输出：
# a1b2c3d4e5f6|张三|zhangsan@example.com|2026-05-27 10:30:00 +0800|feat(auth): add login

# 常用格式符：
# %H  完整 commit hash
# %h  短 hash
# %an 作者姓名
# %ae 作者邮箱
# %ad 作者日期
# %s  提交信息
# %b  提交正文
```

**格式化一览表：**

| 格式符 | 含义 | 示例 |
|--------|------|------|
| `%H` | 完整 commit hash | a1b2c3d4e5f6 |
| `%h` | 短 commit hash | a1b2c3d |
| `%an` | 作者姓名 | 张三 |
| `%ae` | 作者邮箱 | zhangsan@example.com |
| `%ad` | 作者日期 | 2026-05-27 |
| `%ar` | 相对日期 | 2 days ago |
| `%s` | 提交标题 | feat(auth): add login |
| `%b` | 提交正文 | 详细描述 |
| `%d` | 分支和标签 | (HEAD -> master) |

### 图形化分支历史

```bash
# 显示分支图
git log --oneline --graph --all

# 常用组合（最推荐）
git log --oneline --graph --all -20
```

### 格式化示例

```bash
# 查看今天的所有提交
git log --format="%h %an %s" --since="2026-05-27"

# 查看最近一周的提交（相对时间）
git log --format="%h %an %s" --since="1 week ago"

# 查看某个作者的提交
git log --format="%h %s" --author="张三"

# 查看某个文件的改动历史
git log --format="%h %an %s" -- filename.js
```

---

## 代码统计

### git shortlog 统计提交数

```bash
# 按作者统计提交数（简洁格式）
git shortlog

# 输出：
# 张三 (10):
#      feat(auth): add login
#      feat(user): add profile
#      ...

# 李四 (8):
#      fix(payment): resolve null bug
#      ...
```

```bash
# 按提交数排序（数字顺序）
git shortlog -n

# 输出：
# 张三 (10)
# 李四 (8)
# 王五 (5)
```

```bash
# 只看 email 统计（适合邮件列表统计）
git shortlog -n --email

# 按邮箱前缀分组（适合统计公司贡献）
git shortlog -n --groupBy=email
```

### git diff --stat 文件变更统计

```bash
# 查看统计信息（新增/删除行数）
git diff --stat

# 输出：
#  src/auth/login.js   |  50 ++++++++++
#  src/auth/logout.js  |  20 -----
#  2 files changed, 50 insertions(+), 20 deletions(-)
```

```bash
# 查看最近 N 次提交的统计
git diff --stat HEAD~10..HEAD

# 查看某个分支的统计
git diff --stat master..feature-branch
```

### 时间范围统计

```bash
# 统计上周的代码改动
git diff --stat --since="1 week ago" --until="2026-05-27"

# 统计上个月的提交数量
git log --oneline --since="2026-05-01" --until="2026-05-26"

# 统计每月提交数
git log --format="%ad" --date=format:"%Y-%m" | sort | uniq -c
```

### 综合统计脚本

```bash
# 生成团队贡献报告
echo "=== 项目贡献统计 ==="
echo ""
echo "提交数统计："
git shortlog -n --email
echo ""
echo "代码变更统计："
git diff --stat --since="2026-05-01"
echo ""
echo "文件改动排行："
git diff --stat HEAD~50..HEAD | tail -n +4 | sort -k1 -n -r | head -10
```

---

## 追查问题

### git blame 分析代码责任人

**基本用法：**

```bash
# 查看文件的每一行是谁写的
git blame filename.js

# 输出：
# a1b2c3d4 (张三 2026-05-20)  1) const API_URL = 'https://api.example.com'
# e5f6g7h8 (李四 2026-05-21)  2)
# a1b2c3d4 (张三 2026-05-22)  3) async function login() {
```

**带行号查看：**

```bash
# 显示行号
git blame -l filename.js
```

**只看某个范围的行：**

```bash
# 查看第 10-20 行
git blame -L 10,20 filename.js
```

**注意：blame 显示的是最后一次修改该行的作者，不是最初的作者**

### git bisect 定位问题 commit

二分查找是定位 Bug 最有效的方法：

```bash
# Step 1：开始二分查找
git bisect start

# Step 2：标记当前版本是坏的（有 Bug）
git bisect bad

# Step 3：标记一个已知好的版本
git bisect good v1.0.0

# Git 会自动 checkout 到中间的 commit
# 我在这里测试，发现 Bug 还在
# 继续标记：
git bisect bad    # Bug 还在，说明问题在前半段
# 或者
git bisect good   # Bug 已修复，说明问题在后半段

# Git 自动继续查找，直到找到第一个有问题的 commit
# 最终输出：
# a1b2c3d is the first bad commit
# commit a1b2c3d
# Author: 张三
# Summary: fix: incorrectly handle null value
```

**自动二分查找：**

```bash
# 如果有测试脚本，可以自动查找
git bisect start
git bisect bad
git bisect good v1.0.0
git bisect run npm test

# Git 会自动运行测试，直到找到问题 commit
```

**结束二分查找：**

```bash
git bisect reset    # 回到原来的 HEAD
```

### git log -p 定位引入问题的版本

```bash
# 查看某个文件的历史改动
git log -p -- filename.js

# 查看某个 commit 的完整 diff
git log -p -1 abc1234

# 搜索包含特定关键词的 commit
git log -p --all -S "TODO: remove this"

# 查找删除某行代码的 commit
git log -p --all -S "deprecated function" --reverse
```

---

## GitHub 统计功能

### Insights / Analytics

访问 `https://github.com/用户名/仓库/ insights`

#### 贡献图（Contribution Graph）

```
┌─────────────────────────────────────────────┐
│ 2026 年贡献统计                              │
│                                             │
│ 5月                         6月             │
│ ░░▓▓░░▒▒▓░░░▓▓▒▒░░░░▓▓░░                   │
│ ░░▓▓▒▒▒▒▓▓░░▒▒▒▒▓▓░░░▓▓▒▒░░               │
│                                             │
│ 贡献最多：100 次/周                          │
└─────────────────────────────────────────────┘
```

**贡献图统计规则：**
- 提交到默认分支
- 提交 Pull Request
- 提交 Issue
- 仓库被标记星标

### 贡献者统计

```bash
# 查看贡献者列表
https://github.com/用户名/仓库/graphs/contributors

# 显示每个贡献者的：
# - 总提交数
# - 改动行数
# - 贡献趋势图
```

### 提交频率

```bash
# 查看每小时提交热力图
https://github.com/用户名/仓库/graphs/hourly-commit-activity

# 查看每周提交热力图
https://github.com/用户名/仓库/graphs/commit-activity
```

### GitLab 统计

访问 `https://gitlab.com/用户名/仓库/ insights`

---

## 团队协作审计

### 代码所有权

**保护关键文件：**

```bash
# 在 .gitignore 中添加保护规则
# 例如：保护配置文件不被随意修改
secure-config.yml
```

**GitHub Branch Protection：**

1. 访问仓库 → Settings → Branches
2. 点击 "Add rule"
3. 配置保护规则：
   - Require pull request reviews
   - Require signed commits
   - Require status checks to pass
   - Include administrators

### 分支保护审计

```bash
# 查看当前分支保护规则
# 访问 GitHub → 仓库 → Settings → Branches
```

**企业常用配置：**

| 规则 | 作用 |
|------|------|
| Require PR before merge | 必须通过 Code Review 才能合并 |
| Require signed commits | 所有提交必须 GPG 签名 |
| Require status checks | CI 必须通过 |
| Dismiss stale reviews | PR 更新后必须重新 Review |
| Include administrators | 管理员也要遵守规则 |

### 权限变更审计

```bash
# GitHub 查看权限变更
# 访问仓库 → Settings → Manage access

# 查看仓库操作日志（企业版）
# Settings → Audit log
```

**GitLab 审计日志：**

```bash
# 查看审计日志
# Admin Area → Monitor → Audit Logs
```

---

## 常用审计命令速查

### 提交历史查询

```bash
# 查看最近 10 条提交
git log --oneline -10

# 查看某个作者的提交
git log --oneline --author="张三"

# 查看某个文件的改动历史
git log --oneline -- filename.js

# 查看某个目录的所有提交
git log --oneline -- directory/

# 查看某个时间段的提交
git log --oneline --since="2026-05-01" --until="2026-05-26"

# 查看两个版本之间的提交
git log --oneline v1.0.0..v2.0.0
```

### 代码统计

```bash
# 按作者统计提交数
git shortlog -n

# 按作者统计（包含邮件）
git shortlog -n --email

# 查看文件变更统计
git diff --stat HEAD~10..HEAD

# 查看某人的代码改动行数
git diff --shortstat --author="张三" --since="2026-05-01"
```

### 问题定位

```bash
# 查看某文件的 blame
git blame filename.js

# 查看某文件特定行的 blame
git blame -L 10,20 filename.js

# 二分查找定位问题
git bisect start
git bisect bad
git bisect good v1.0.0
# ... 测试 ...
git bisect good/bad
git bisect reset

# 查看包含关键词的 commit
git log -p -S "关键词"
```

### 分支审计

```bash
# 查看所有分支
git branch -a

# 查看合并到 master 的所有分支
git branch --merged master

# 查看还没合并到 master 的分支
git branch --no-merged master

# 查看每个分支的最新提交
git branch -v

# 查看删除的分支
git reflog | grep -oP 'branch:.* -> deleted'
```

### 远程仓库审计

```bash
# 查看所有远程仓库
git remote -v

# 查看远程仓库的详细信息
git remote show origin

# 查看远程分支
git branch -r

# 查看谁在什么时候推送了什么
git reflog --date=relative
```

### GitHub CLI 审计

```bash
# 安装 GitHub CLI
# npm install -g gh

# 查看仓库活动
gh repo view --activity

# 查看贡献者
gh repo view --contributors

# 查看所有 Issue 和 PR
gh issue list
gh pr list

# 查看某个 PR 的审查意见
gh pr view 123 --json comments
```

---

## 企业级审计实践

### 建立审计机制

**1. commit message 规范（参考第 14 节）**

好的 commit message 是审计的基础：

```bash
# 好的 commit message 便于追溯
git log --oneline --grep="fix"

# 按模块查找
git log --oneline --grep="auth"
```

**2. 配置 CI 审计**

```yaml
# .github/workflows/audit.yml
name: Commit Audit
on:
  push:
    branches: [main]
  pull_request:
    types: [opened, synchronize]

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Generate audit report
        run: |
          echo "=== 提交审计报告 ===" >> $GITHUB_STEP_SUMMARY
          git log --format="%h,%an,%ae,%s" --since="2026-05-01" >> $GITHUB_STEP_SUMMARY
```

**3. 定期生成统计报告**

```bash
# 每周一生成上周的代码贡献报告
# 配合 cron 或 CI 定时任务
git log --format="%an,%ae,%s" --since="1 week ago" --until="2026-05-27"
```

### 合规审计要点

| 合规要求 | Git 实现 |
|---------|---------|
| 代码追溯 | 每条提交有唯一的 commit hash |
| 身份验证 | GPG 签名证明提交者身份 |
| 变更记录 | git log 完整记录每次改动 |
| 权限控制 | Branch protection + 代码审查 |
| 审计日志 | GitHub/GitLab 审计日志功能 |

---

## 本节总结

```
审计命令速查：

提交历史：
  git log --oneline              # 简洁历史
  git log --format="%H|%an|%s"   # 自定义格式
  git log --graph --all           # 图形化

代码统计：
  git shortlog -n                # 按作者统计提交数
  git diff --stat                # 变更统计
  git log --since/until          # 时间范围

问题定位：
  git blame filename.js          # 查看代码责任人
  git bisect start/bad/good      # 二分查找定位问题
  git log -p -S "关键词"          # 搜索关键词改动

GitHub 审计：
  仓库 → Insights → Contributors # 贡献者统计
  仓库 → Settings → Audit log    # 审计日志（企业版）

⚠️ 好习惯：
   - 每次 merge 前确认代码审查记录
   - 重要的安全改动要记录在 commit 中
   - 定期查看团队的贡献统计
   - 善用 CI 自动生成审计报告
```