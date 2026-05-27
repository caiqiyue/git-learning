# 18. Git 钩子（Hooks）

## 本节目标

学完这节后，你会：
- 理解 Git 钩子是什么以及存放在哪里
- 掌握常见的钩子类型及其触发时机
- 学会编写简单的钩子脚本（Shell 和 Python）
- 理解如何跳过钩子
- 了解团队协作中如何共享钩子（husky + lint-staged）
- 理解企业级 CI 中钩子验证的作用

---

## Git 钩子是什么

**Git 钩子是特定事件触发时自动运行的脚本**，位于 `.git/hooks/` 目录中。

```
提交代码时发生了什么：
  
  你执行 git commit -m "fix bug"
              ↓
  Git 自动触发 pre-commit 钩子
              ↓
  pre-commit 检查代码风格... 通过 ✓
              ↓
  Git 自动触发 commit-msg 钩子
              ↓
  commit-msg 检查提交信息格式... 通过 ✓
              ↓
  实际执行 commit（提交到仓库）
              ↓
  Git 自动触发 post-commit 钩子
              ↓
  完成！
```

**钩子的价值：** 在代码进入仓库之前做最后一次质量检查，发现问题及时阻止。

---

## 钩子存放位置

### 本地钩子目录

```bash
# 钩子目录位置
仓库根目录/.git/hooks/

# 查看钩子目录内容
ls -la .git/hooks/
# 输出：
# pre-commit         ← 可执行脚本
# pre-commit.sample  ← 示例文件（带 .sample 后缀，不会执行）
```

### 钩子文件命名

```bash
# 钩子文件名 = 触发时机
# 可用钩子类型见下方「常见钩子类型」一节

# 示例：这些钩子会被 Git 自动触发
pre-commit           # 提交前执行
prepare-commit-msg   # 准备提交信息前
commit-msg           # 提交信息编辑后
post-commit          # 提交完成后
pre-push            # 推送前
pre-rebase          # 变基前
```

### 跨平台注意

```bash
# Windows Git Bash / Linux / macOS
# 钩子脚本可以是 Shell、Python、Ruby、批处理等

# Windows 原生（CMD/PowerShell）
# 使用 .bat 或 .cmd 扩展名
# pre-commit.bat

# 示例：Windows Git Bash 环境下使用 Shell 脚本
#!/bin/sh  # Shebang，类 Unix 系统

# 跨平台建议：
# 1. 优先使用 Shell 脚本（跨平台兼容性好）
# 2. 如果需要更复杂逻辑，使用 Python（跨平台）
# 3. 避免使用 Windows 批处理脚本（不可移植）
```

---

## 常见的钩子类型

### pre-commit：提交前检查

**触发时机：** `git commit` 执行后，实际创建提交之前。

**典型用途：**
- 检查代码风格
- 运行 ESLint、Prettier
- 执行单元测试（快速检查）
- 检查文件变更是否合理

```bash
#!/bin/sh
# pre-commit 钩子示例

# 获取暂存区的文件列表
FILES=$(git diff --cached --name-only --diff-filter=ACM)

# 检查是否有文件被暂存
if [ -z "$FILES" ]; then
  exit 0  # 没有暂存文件，直接通过
fi

# 检查暂存文件是否包含 console.log
echo "$FILES" | grep -q "console.log"
if [ $? -eq 0 ]; then
  echo "Error: 提交包含 console.log，请移除后提交"
  exit 1  # 退出码 1 表示拒绝提交
fi

echo "pre-commit 检查通过"
exit 0  # 退出码 0 表示通过
```

**注意：** pre-commit 只能看到暂存区（staged）的文件，即 `git add` 后的状态。

### prepare-commit-msg：编辑提交信息前

**触发时机：** 默认提交信息生成后，用户编辑之前。

**典型用途：**
- 自动填充提交信息的模板
- 自动添加 ticket ID
- 在多成员团队中自动添加提交者信息

```bash
#!/bin/sh
# prepare-commit-msg 钩子示例

# 参数：$1 = 临时文件路径（包含默认提交信息）
COMMIT_MSG_FILE=$1
BRANCH=$(git symbolic-ref --short HEAD 2>/dev/null || echo "")

# 自动填充 ticket ID（如果分支名包含）
if [[ "$BRANCH" =~ ^feature/([A-Z]+-[0-9]+) ]]; then
  TICKET="${BASH_REMATCH[1]}"
  # 在第一行后添加空白行和 TICKET
  sed -i "1a\\${TICKET}" "$COMMIT_MSG_FILE"
fi

exit 0
```

**跨平台注意：**
- `sed -i` 在 macOS 上需要 `sed -i ''`，但在 Git Bash on Windows 使用 Linux 语法
- 如果需要跨平台兼容，建议使用 Python 编写钩子脚本

### commit-msg：验证提交信息格式

**触发时机：** 用户编辑完提交信息后，提交创建之前。

**典型用途：**
- 强制 Conventional Commits 格式
- 检查提交信息长度
- 确保填写了必填项

```bash
#!/bin/sh
# commit-msg 钩子示例：强制 Conventional Commits 格式

COMMIT_MSG=$(cat $1)  # $1 是提交信息文件路径
PATTERN="^(feat|fix|docs|style|refactor|perf|test|chore|revert)(\(.+\))?: .+"

if ! echo "$COMMIT_MSG" | grep -qE "$PATTERN"; then
  echo "错误：提交信息格式不符合规范"
  echo ""
  echo "正确格式：<type>(<scope>): <subject>"
  echo "例如：feat(auth): add login functionality"
  echo "允许的 type：feat, fix, docs, style, refactor, perf, test, chore, revert"
  exit 1
fi

exit 0
```

### post-commit：提交完成后

**触发时机：** 提交创建完成后（无论成功失败都会执行）。

**典型用途：**
- 发送通知（如 Slack、邮件）
- 触发 CI
- 记录提交时间戳
- 自动备份

```bash
#!/bin/sh
# post-commit 钩子示例：提交后打印提示

echo "========= 提交完成 ========="
echo "提交人：$(git config user.name)"
echo "提交 SHA：$(git rev-parse HEAD)"
echo "提交分支：$(git symbolic-ref --short HEAD 2>/dev/null || echo ' detached HEAD')"
echo "=============================="

exit 0
```

### pre-push：推送前检查

**触发时机：** `git push` 执行时，实际推送之前。

**典型用途：**
- 检查本地分支是否落后远程
- 检查是否包含敏感信息
- 运行更耗时的测试（比 pre-commit 更全面）
- 阻止推送到受保护分支

```bash
#!/bin/sh
# pre-push 钩子示例：检查是否落后远程

# 获取要推送的分支和远程
REMOTE_BRANCH=$(git symbolic-ref --short HEAD)
REMOTE_NAME=origin
REMOTE_URL=$(git remote get-url $REMOTE_NAME 2>/dev/null || echo "")

# 获取远程分支最新提交
REMOTE_SHA=$(git ls-remote $REMOTE_NAME $REMOTE_BRANCH 2>/dev/null | cut -f1)

if [ -z "$REMOTE_SHA" ]; then
  # 远程没有这个分支，第一次推送，跳过检查
  exit 0
fi

# 获取本地分支最新提交
LOCAL_SHA=$(git rev-parse HEAD)

# 检查是否落后
if [ "$LOCAL_SHA" != "$REMOTE_SHA" ]; then
  BEHIND=$(git log $LOCAL_SHA..$REMOTE_SHA --oneline | wc -l)
  if [ $BEHIND -gt 0 ]; then
    echo "错误：本地分支落后远程 $BEHIND 个提交"
    echo "请先执行 git pull 拉取最新代码"
    echo ""
    echo "如果确实需要强制推送（不推荐），请使用："
    echo "  git push --force-with-lease $REMOTE_NAME $REMOTE_BRANCH"
    exit 1
  fi
fi

exit 0
```

**注意：** pre-push 在 Windows Git Bash 下需要正确处理 `cut` 命令。Git for Windows 内置了 Unix 工具，基本兼容。

### pre-rebase：变基前

**触发时机：** `git rebase` 执行前。

**典型用途：**
- 阻止对已推送的提交进行变基
- 要求变基前先 stash 当前更改

```bash
#!/bin/sh
# pre-rebase 钩子示例：阻止变基已推送的提交

# 获取变基的目标分支
UPSTREAM=$1
BRANCH=$2

if [ -z "$BRANCH" ]; then
  # 正在变基到指定分支，不做检查
  exit 0
fi

# 检查是否有未推送的提交
UNPUSHED=$(git log origin/main..HEAD --oneline 2>/dev/null | wc -l)
if [ $UNPUSHED -gt 0 ]; then
  echo "警告：你有 $UNPUSHED 个提交尚未推送到远程"
  echo "对这些提交进行变基是危险的"
  echo "如果确认要继续，请按 Ctrl+C 取消此钩子，然后重试"
  read -p "继续执行变基？(y/n) " confirm
  if [ "$confirm" != "y" ]; then
    exit 1
  fi
fi

exit 0
```

---

## 钩子脚本示例

### Shell 版本

```bash
#!/bin/sh
# 文件名：pre-commit
# 位置：.git/hooks/pre-commit

# 设置检查规则
ERRORS=""

# 检查是否有调试代码
for FILE in $(git diff --cached --name-only); do
  if grep -q "console.log" "$FILE" 2>/dev/null; then
    echo "错误：$FILE 包含 console.log"
    ERRORS="yes"
  fi
done

# 检查是否有 TODO 但没有标注负责人
for FILE in $(git diff --cached --name-only); do
  if grep -q "TODO" "$FILE" 2>/dev/null; then
    # 允许 TODO 但必须有 @xxx 标注
    if ! grep -q "@[a-zA-Z]" "$FILE" 2>/dev/null; then
      echo "警告：$FILE 包含 TODO 但没有 @标注负责人"
    fi
  fi
done

# 返回检查结果
if [ "$ERRORS" = "yes" ]; then
  echo "提交被 pre-commit 钩子阻止"
  exit 1
fi

echo "pre-commit 检查通过"
exit 0
```

### Python 版本

```python
#!/usr/bin/env python3
# 文件名：pre-commit
# 位置：.git/hooks/pre-commit

import subprocess
import sys
import re

def get_staged_files():
    """获取暂存的文件列表"""
    result = subprocess.run(
        ['git', 'diff', '--cached', '--name-only', '--diff-filter=ACM'],
        capture_output=True,
        text=True
    )
    return [f.strip() for f in result.stdout.split('\n') if f.strip()]

def check_file(filename):
    """检查单个文件"""
    errors = []
    
    try:
        with open(filename, 'r', encoding='utf-8') as f:
            content = f.read()
            
            # 检查 console.log
            if re.search(r'console\.log', content):
                errors.append(f'{filename}: 包含 console.log')
                
            # 检查 console.error
            if re.search(r'console\.error', content):
                errors.append(f'{filename}: 包含 console.error')
                
            # 检查 debugger
            if re.search(r'\bdebugger\b', content):
                errors.append(f'{filename}: 包含 debugger 语句')
                
            # 检查硬编码的敏感信息
            sensitive_patterns = [
                (r'password\s*=\s*["\'][^"\']+["\']', '硬编码密码'),
                (r'api[_-]?key\s*=\s*["\'][^"\']+["\']', '硬编码 API Key'),
                (r'secret\s*=\s*["\'][^"\']+["\']', '硬编码 Secret'),
            ]
            
            for pattern, desc in sensitive_patterns:
                if re.search(pattern, content, re.IGNORECASE):
                    errors.append(f'{filename}: {desc}')
                    
    except Exception as e:
        print(f'警告：无法检查 {filename}: {e}')
        
    return errors

def main():
    files = get_staged_files()
    if not files:
        print('没有暂存文件，跳过检查')
        return 0
        
    all_errors = []
    for filename in files:
        errors = check_file(filename)
        all_errors.extend(errors)
        
    if all_errors:
        print('=' * 50)
        print('pre-commit 检查失败：')
        for error in all_errors:
            print(f'  - {error}')
        print('=' * 50)
        return 1
        
    print('pre-commit 检查通过')
    return 0

if __name__ == '__main__':
    sys.exit(main())
```

**注意：** Python 钩子需要目标机器安装 Python。Windows 默认不安装 Python，但 Git for Windows 自带了 Python 环境（msys2）。

---

## 如何跳过钩子

### 使用 --no-verify

```bash
# 跳过所有钩子，直接提交
git commit -m "紧急修复" --no-verify

# 跳过所有钩子，直接推送
git push origin main --no-verify
```

**注意：** `--no-verify` 应该谨慎使用，只在你确认跳过钩子没问题的情况下使用。

### 正确使用场景

```bash
# 场景 1：本地测试时快速提交
git commit -m "temp commit" --no-verify

# 场景 2：合并 PR 时（Review 已经通过，不需要重复检查）
git merge feature/xxx --no-verify

# 场景 3：cherry-pick 时（提交信息格式可能不同）
git cherry-pick abc123 --no-verify
```

### 错误使用场景

```bash
# ❌ 错误：不应该用 --no-verify 来绕过真正的代码问题
git commit -m "fix null pointer" --no-verify
# 为什么是错的：因为 pre-commit 检查发现了问题，你就应该修复代码，而不是跳过检查
```

**跨平台注意：** `--no-verify` 在所有平台上行为一致，没有跨平台差异。

---

## 团队协作中共享钩子

### 问题：本地钩子不提交到仓库

```bash
# 你的本地钩子
.git/hooks/pre-commit  ← 这个文件不会被提交到仓库

# 问题：团队成员的钩子不统一
# 小王有 pre-commit 检查代码风格
# 小李没有，什么检查都没有
```

### 解决方案：Husky

**Husky 是一个在项目中管理 Git 钩子的工具**，让你把钩子配置文件提交到仓库，团队成员安装依赖时自动配置钩子。

### Husky 安装和配置

```bash
# 1. 安装 husky（作为开发依赖）
npm install husky --save-dev

# 2. 初始化 husky（创建 .husky/ 目录）
npx husky install

# 3. 添加 git hooks 配置到 package.json
npm pkg set scripts.prepare="husky install"

# 4. 创建一个钩子
npx husky add .husky/pre-commit "npm test"

# 5. 提交 husky 配置（.husky/ 目录下的文件会被提交到仓库）
git add .husky/pre-commit
git commit -m "chore: add husky pre-commit hook"
```

### Husky + lint-staged 完整配置

```bash
# 1. 安装 lint-staged（只检查暂存文件）
npm install lint-staged --save-dev

# 2. 配置 lint-staged（package.json）
# {
#   "lint-staged": {
#     "*.js": ["eslint --fix", "prettier --write"],
#     "*.css": ["stylelint --fix"],
#     "*.md": ["markdownlint --fix"]
#   }
# }

# 3. 修改 pre-commit 钩子
npx husky add .husky/pre-commit "npx lint-staged"

# 4. 最终的 .husky/pre-commit 文件内容
# #!/usr/bin/env sh
# . "$(dirname -- "$0")/_/husky.sh"
# npx lint-staged

# 5. 安装配置
npm install

# 以后每次 git commit 都会：
#   1. Husky 触发 pre-commit
#   2. lint-staged 只检查暂存的文件
#   3. 对这些文件运行 ESLint、Prettier 等
#   4. 只有检查通过才真正提交
```

### 钩子内容示例

```bash
# .husky/pre-commit 文件内容
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

echo "Running pre-commit checks..."

# 运行 lint-staged
npx lint-staged

# 如果 lint-staged 失败，退出码为 1，阻止提交
if [ $? -ne 0 ]; then
  echo "pre-commit 检查失败，请修复上述问题后重试"
  exit 1
fi

echo "pre-commit 检查通过"
```

**注意：** Husky 需要 Git 2.9+ 版本。在 Windows 上使用 Husky 时，确保项目在 Git 管理的目录下，并且使用 Git Bash 或 WSL 执行命令。

### 团队协作流程

```bash
# 新成员加入项目
git clone git@github.com:owner/repo.git
cd repo
npm install        # npm install 会自动执行 prepare 脚本
npx husky install  # 安装 husky 钩子（如果 package.json 中有 prepare 脚本则自动执行）

# 现在新成员的本地仓库也有和你一样的钩子了
```

### 禁用/启用钩子

```bash
# 临时禁用钩子（所有钩子都跳过）
git commit -m "temp" --no-verify

# 或者通过配置禁用
git config core.hooksPath /dev/null  # Unix 系统
git config core.hooksPath ""        # 重置为默认

# 查看当前钩子路径
git config core.hooksPath
```

---

## 企业级 CI 中的钩子验证

### 问题：本地钩子可以被跳过

```bash
# 团队成员可以跳过本地钩子
git commit --no-verify
# 或者直接禁用本地钩子
git config core.hooksPath /dev/null
```

### CI 中的钩子验证是最后防线

```
提交流程：
  
  开发者本地                CI 服务器（真正的守门员）
     ↓                            ↓
  pre-commit ──────────────▶  CI 环境重新检查
  (可跳过)                        ↓
     ↓                       lint / test / build
  commit                        ↓
  (可跳过)                   全部通过 ✓
     ↓                       发布到环境
  push                      （任何失败都会阻止）
     ↓
  remote
```

### CI 中复现钩子检查

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      # CI 中安装 husky 并执行钩子
      - run: npm ci
      - run: npm run prepare  # husky install
      - run: npm run lint
      
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci
      - run: npm run test
      
  build:
    runs-on: ubuntu-latest
    needs: [lint, test]
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci
      - run: npm run build
```

### CI 中强制 commitlint

```yaml
# .github/workflows/commitlint.yml
name: Commitlint

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  commitlint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0  # 全量克隆，因为 commitlint 需要看所有提交历史
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          
      - run: npm ci
      - run: npm install @commitlint/cli @commitlint/config-conventional
      - run: npx commitlint --from HEAD~1 --to HEAD --verbose
```

### 常见 CI 钩子检查项

| 检查项 | 本地钩子 | CI 钩子 | 说明 |
|--------|---------|---------|------|
| 代码风格（ESLint） | ✅ 可跳过 | ✅ 必须通过 | 格式问题 |
| 提交信息格式 | ✅ 可跳过 | ✅ 必须通过 | 不规范的信息会被拒绝 |
| 单元测试 | ✅ 可跳过 | ✅ 必须通过 | 测试失败阻止合并 |
| 安全扫描 | ❌ 通常没有 | ✅ 必须通过 | 检测敏感信息泄露 |
| 依赖审计 | ❌ 通常没有 | ✅ 必须通过 | 检测有漏洞的依赖 |

**注意：** CI 中的钩子检查是强制性的，不能被跳过。即使开发者在本地使用 `--no-verify`，CI 也会执行完整的检查流程。

---

## 本节总结

```
Git 钩子位置：.git/hooks/

常见钩子类型：
  pre-commit        提交前检查（代码风格、测试）
  prepare-commit-msg 编辑提交信息前（自动填充）
  commit-msg        验证提交信息格式
  post-commit       提交完成后（通知、备份）
  pre-push          推送前检查（分支状态检查）
  pre-rebase        变基前（警告已推送提交）

跳过钩子：
  git commit --no-verify   # 跳过所有提交钩子
  git push --no-verify      # 跳过所有推送钩子

团队共享钩子：
  husky + lint-staged       # 统一团队钩子配置

企业 CI：
  CI 中复现钩子检查          # 本地钩子可跳过，CI 强制执行
  commitlint 在 CI 中强制    # 提交信息不规范会被 CI 拒绝
```