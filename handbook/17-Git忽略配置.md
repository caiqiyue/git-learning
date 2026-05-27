# 17. Git 忽略配置（.gitignore）

## 本节目标

学完这节后，你会：
- 理解 .gitignore 的作用和工作原理
- 掌握 .gitignore 的语法规则
- 学会常见的忽略模式
- 理解全局 .gitignore 的配置方法
- 掌握已跟踪文件如何取消跟踪
- 了解 .gitignore 和 .gitkeep 的区别
- 会使用 GitHub 官方忽略模板

---

## .gitignore 的作用和原理

### 什么是 .gitignore

**.gitignore 是一个文本文件，告诉 Git 忽略指定文件/文件夹，不将它们纳入版本控制。**

```
没有被 .gitignore：
  代码仓库包含 500MB 的 node_modules
  每次克隆都要下载 500MB 无关紧要的依赖
  
使用了 .gitignore：
  node_modules/ 被忽略
  代码仓库只包含源代码，体积只有 5MB
  开发者 clone 后运行 npm install 即可
```

### 工作原理

```
.gitignore 规则匹配的是「未被跟踪」的文件。

文件生命周期：
  1. 新建文件（untracked）→ 匹配 .gitignore → 被忽略
  2. 已跟踪文件（tracked）→ .gitignore 对它无效
  
重要：
  - .gitignore 对「未跟踪文件」才起作用
  - 已经提交到仓库的文件，即使添加到 .gitignore 也不会自动消失
  - 需要额外步骤来取消跟踪已跟踪文件
```

### .gitignore 文件位置

```
项目根目录的 .gitignore：
  e:/my-project/.gitignore    ← 作用于整个项目

.git/info/exclude（仅本地）：
  e:/my-project/.git/info/exclude   ← 仅对本仓库生效，不提交到远程
  常用于个人临时文件

全局 .gitignore（跨平台配置方式不同）：
  ~/.gitignore_global    ← 对所有本地仓库生效
```

**跨平台注意：** 在 Windows 上，`~` 通常对应 `C:\Users\用户名`，完整路径是 `C:\Users\Administrator\.gitignore_global`。在 macOS/Linux 上是 `/home/username/.gitignore_global`。

---

## 语法规则

### 基础规则

```gitignore
# 这是注释（以 # 开头）

# 忽略指定文件
credentials.json
*.log

# 忽略指定文件夹（末尾带 /）
node_modules/
dist/
build/

# 忽略所有同名文件（任意路径）
*.log              ← 忽略所有 .log 文件

# 忽略根目录的指定文件（前面带 /）
/credentials.json  ← 只忽略根目录的 credentials.json，不忽略 sub/credentials.json

# 忽略文件夹下的指定文件（前面不带 /）
config.json        ← 忽略所有目录下的 config.json，包括 sub/config.json

# 忽略文件夹下的所有文件（使用 **）
build/**/*         ← 忽略 build 目录下的所有文件（包括子目录）
```

### 进阶语法

```gitignore
# 否定规则（!）—— 打破忽略
*.log              ← 忽略所有 .log 文件
!important.log     ← 但 important.log 除外

# 否定规则用于恢复被忽略的目录
node_modules/      ← 忽略 node_modules
!/node_modules/lib ← 但 node_modules/lib 保留

# 双星号（**）—— 匹配任意层级目录
**/*.log           ← 任意目录下的 .log 文件
build/**/*.js      ← build 目录及其子目录下所有 .js 文件

# 单星号（*）—— 匹配任意字符（不含 /）
*.js               ← 所有 .js 文件
file-*.txt         ← file-开头，-任意字符，.txt 结尾

# 问号（?）—— 匹配单个任意字符
file-?.txt         ← file-1.txt、file-a.txt 等

# 方括号（[]）—— 匹配单个指定字符
file-[ab].txt      ← file-a.txt、file-b.txt
file-[0-9].txt     ← file-0.txt 到 file-9.txt
```

### 规则优先级

```gitignore
# 优先级：从上到下，后面的规则覆盖前面的

# 先忽略所有 .log
*.log

# 但 error.log 除外
!error.log

# 结果：所有 .log 被忽略，但 error.log 保留
```

**注意：** 如果一个文件夹被整体忽略（如 `node_modules/`），那么即使设置 `!node_modules/lib/something.js` 也不会生效，因为 Git 根本不会进入这个目录。

---

## 常见忽略模式

### 操作系统文件

```gitignore
# macOS
.DS_Store              ← macOS 生成的文件
.AppleDouble           ← macOS 元数据
.LSOverride            ← macOS 文件缓存
._*                    ← macOS 临时文件

# Windows
Thumbs.db              ← Windows 缩略图缓存
ehthumbs.db            ← Windows 视频缩略图
Desktop.ini            ← Windows 桌面配置
$*                     ← Windows 临时文件（以 $ 开头的文件）
```

### IDE 和编辑器配置

```gitignore
# VS Code
.vscode/
!.vscode/settings.json
!.vscode/tasks.json
!.vscode/launch.json

# IntelliJ IDEA
.idea/
*.iml
out/

# Eclipse
.classpath
.project
.settings/

# Vim
*.swp
*.swo
*~

# Emacs
*~
.\#*

# Sublime Text
*.sublime-*

# Atom
.atom/
```

### 日志文件

```gitignore
# 日志文件（按类型）
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*
lerna-debug.log*

# 日志目录
logs/
*.log
```

### 依赖目录

```gitignore
# Node.js
node_modules/
npm-debug.log*
yarn-error.log*

# Python
venv/
env/
ENV/
.venv
__pycache__/
*.py[cod]
*$py.class
.Python
build/
develop-eggs/
dist/
downloads/
eggs/
.eggs/
lib/
lib64/
parts/
sdist/
var/
wheels/
*.egg-info/
.installed.cfg
*.egg

# Java/Maven
target/
pom.xml.tag
pom.xml.releaseBackup
pom.xml.versionsBackup
pom.xml.next

# Java/Gradle
build/
.gradle/

# Ruby
.bundle/
vendor/bundle/
```

### 构建产物

```gitignore
# 前端构建产物
dist/
build/
output/
.next/
.nuxt/
out/

# 后端构建产物
target/
*.class
*.jar
*.war
*.ear

# Go
*.exe
*.exe~
*.dll
*.so
*.dylib
*.test
*.out
vendor/

# Rust
target/
Cargo.lock
```

### 环境配置文件

```gitignore
# 环境变量文件（敏感！）
.env
.env.local
.env.development
.env.production
.env.*.local

# 注意：.env.example 是示例文件，可以提交到仓库
.env.example
```

### 敏感文件

```gitignore
# 凭证和密钥
credentials.json
*.pem
*.key
id_rsa*
id_ecdsa*
*.p12
*.pfx
secrets.json
secrets.yaml

# 数据库连接配置
*.sqlite
*.db
config/database.yml

# 注意：不要提交真实的 .env 文件到仓库
# .env 包含真实的数据库密码、API 密钥等
```

### 其他常见忽略

```gitignore
# 测试覆盖率报告
coverage/
.nyc_output/
lcov.info

# 临时文件和调试文件
*.tmp
*.temp
*.cache
*.swp
*.swo

# 压缩包
*.zip
*.tar
*.tar.gz
*.rar

# 文档（如果不想版本控制）
# *.md （保留 README.md 等）
# !README.md
```

---

## 全局 .gitignore

### 为什么需要全局 .gitignore

有些文件是所有项目都会忽略的，比如操作系统文件和 IDE 配置。不需要在每个项目的 .gitignore 中重复配置。

### 配置方法

**跨平台注意：** 全局 .gitignore 配置在所有平台上语法相同，但配置文件路径不同。

```bash
# 1. 创建全局忽略文件
touch ~/.gitignore_global

# 2. 添加全局忽略规则（跨平台通用）
cat >> ~/.gitignore_global << 'EOF'
# OS files
.DS_Store
Thumbs.db

# IDE
.vscode/
.idea/
*.swp
*~

# Logs
*.log
EOF

# 3. 配置 Git 使用全局忽略文件

# Windows (Git Bash/msys2)
git config --global core.excludesFile ~/.gitignore_global

# macOS / Linux
git config --global core.excludesFile ~/.gitignore_global

# 4. 验证配置
git config --global core.excludesFile
# 输出应该是：/c/Users/Administrator/.gitignore_global
```

**跨平台注意：** 在 Windows Git Bash 中，`~` 会自动转换为 `/c/Users/用户名`，配置后路径可能显示为 `/c/Users/Administrator/.gitignore_global`，这是正常的。

### Windows 专属全局忽略

```gitignore
# Windows 特定规则（添加到 ~/.gitignore_global）
Thumbs.db
ehthumbs.db
Desktop.ini
# 注意：Unix shell 临时文件标记（Windows 用户可忽略）
~$
*~
```

---

## 已跟踪文件如何取消跟踪

### 场景问题

```bash
# 你发现有个文件已经被跟踪了
$ git ls-files | grep credentials
credentials.json   ← 已经被跟踪

# 你把它加入 .gitignore
echo credentials.json >> .gitignore

# 但它还在跟踪中！
$ git status
# .gitignore 变了，但没有 untracked 文件出现
```

### 解决方法

```bash
# 步骤 1：从 Git 跟踪中移除（但保留本地文件）
git rm --cached credentials.json

# --cached 的含义：
#   git rm credentials.json     → 删除文件并停止跟踪（本地文件也没了）
#   git rm --cached xxx        → 只从 Git 跟踪中移除，保留本地文件

# 步骤 2：提交这个更改
git commit -m "chore: stop tracking credentials.json"

# 步骤 3：以后这个文件就会根据 .gitignore 被忽略了
```

### 如果已经提交到远程

```bash
# 如果已经 push 到远程，需要额外步骤
# 1. 先在本地移除
git rm --cached credentials.json

# 2. 提交
git commit -m "chore: stop tracking credentials.json"

# 3. 强制推送到远程（从远程历史中移除）
# ⚠️ 注意：这会重写远程历史，如果有人已经拉取了你的提交，会有问题
git push origin main --force

# 更好的做法是通知协作者后再执行
```

**注意：** `git rm --cached` 只移除 Git 跟踪，不删除本地文件。本地文件还在，只是 Git 不再关注它。

### 批量移除已跟踪文件

```bash
# 如果你一次性忽略了整个目录（如 node_modules）
# 但它已经被跟踪了

# 批量移除（保留本地文件）
git rm -r --cached node_modules/

# 或者移除所有匹配 .gitignore 规则的文件
# 先查看有哪些文件会被移除（dry run）
git clean -n -d

# -n = 预览，不实际删除
# -d = 包含目录
```

### 恢复已移除的跟踪文件（如果误操作）

```bash
# 如果你误执行了 git rm --cached
# 可以通过以下方式恢复
git checkout HEAD -- credentials.json
```

---

## .gitignore 和 .gitkeep 的区别

### 表格对比

| 特性 | .gitignore | .gitkeep |
|------|------------|---------|
| 作用 | 告诉 Git 忽略哪些文件 | 强制 Git 跟踪空目录 |
| 位置 | 项目根目录 | 需要保留的空目录 |
| 内容 | 列出忽略模式 | 空文件即可（内容无所谓） |
| Git 行为 | 不跟踪匹配的文件 | 让 Git 跟踪（管理）该目录 |

### 为什么需要 .gitkeep

```bash
# 场景：你想保留某个空目录
$ mkdir logs
$ echo "日志文件会在这里" > logs/.gitkeep
# 但 logs/ 目录下没有任何其他文件

# 没有 .gitkeep 的情况
$ git status
# logs/ 不会出现在 untracked 中，因为 Git 不跟踪空目录

# 有 .gitkeep 的情况
$ echo "" > logs/.gitkeep
$ git status
# logs/.gitkeep 出现了，logs 目录被跟踪了
```

**注意：** Git 不跟踪空目录。如果你想保留目录结构（比如 `logs/` 目录供将来放日志文件），在目录中放一个 `.gitkeep` 文件即可。

### .gitkeep 的替代方案

如果不想用 `.gitkeep`（因为它名字不直观），可以用其他方式：

```bash
# 方式 1：使用 .gitkeep（最通用）
touch logs/.gitkeep

# 方式 2：在 .gitkeep 中加内容说明用途
echo "# 这个目录用于存储日志文件" > logs/.gitkeep

# 方式 3：使用 README.md（更常见）
echo "# Logs Directory\n\nAll application log files are stored here." > logs/README.md

# 方式 4：在 .gitignore 中反向添加（不推荐）
# logs/.gitignore
# !.gitkeep
# 这种方式不好，因为逻辑复杂
```

### 实际应用

```bash
# 实际项目中常见的目录保留模式
mkdir -p logs cache uploads temp

# 创建 .gitkeep 或 README
touch logs/.gitkeep cache/.gitkeep uploads/.gitkeep temp/.gitkeep

# 添加到 .gitignore（忽略目录内容，但保留目录结构）
logs/
cache/
uploads/
temp/

# gitignore 中的例外规则
# logs/.gitkeep
# !logs/.gitkeep
```

**注意：** 如果在 .gitignore 中写了 `logs/`（忽略 logs 目录下的所有内容），那么 `!logs/.gitkeep` 不会生效，因为 Git 根本不进入被忽略的目录。

---

## GitHub 官方 .gitignore 模板

### 使用方法

```bash
# 方法 1：通过 GitHub CLI 创建项目时指定模板
gh repo create my-project --gitignore Unity

# 支持的模板列表（部分）
# Unity、Node、Python、Java、Go、Rust、Ruby、Docker、Terraform 等

# 方法 2：手动下载模板
# 访问 https://github.com/github/gitignore
# 下载对应语言的 .gitignore 文件
```

### 常用模板速查

```bash
# 常见语言的 gitignore 模板内容

# Unity (.gitignore)
# https://github.com/github/gitignore/blob/main/Unity.gitignore
/[Ll]ibrary/
/[Tt]emp/
/[Oo]bj/
/[Bb]uild/
/[Bb]uilds/
/[Ll]ogs/
/[Uu]ser[Ss]ettings/
*.pidb
*.log

# Node.js (.gitignore)
# https://github.com/github/gitignore/blob/main/Node.gitignore
node_modules/
.npm
.eslintcache
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*
dist/
```

### 推荐做法

```bash
# 1. 创建新项目时先获取对应模板
# 访问 https://github.com/github/gitignore 获取模板

# 2. 常用模板组合示例

# 前端项目（Node + Vue/React）
node_modules/
dist/
.npm
.eslintcache
*.log

# 后端 Python 项目
venv/
__pycache__/
*.py[cod]
*.log
.env

# 后端 Go 项目
*.exe
vendor/
.go
```

---

## 本节总结

```
.gitignore 语法：
  *.log          → 忽略所有 .log 文件
  node_modules/ → 忽略 node_modules 目录
  /file.txt     → 只忽略根目录的 file.txt
  **/*.js       → 任意目录下的 .js 文件
  !important.log → 打破忽略，恢复 important.log

常见忽略：
  OS：.DS_Store、Thumbs.db
  IDE：.idea/、.vscode/、*.swp
  日志：*.log、logs/
  依赖：node_modules/、venv/、target/
  构建：dist/、build/、target/
  敏感：.env、*.pem、credentials.json

全局 .gitignore：
  git config --global core.excludesFile ~/.gitignore_global

已跟踪文件取消跟踪：
  git rm --cached filename

.gitkeep 作用：
  让 Git 跟踪空目录（因为 Git 不跟踪空目录）
```