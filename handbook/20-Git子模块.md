# 20. Git 子模块（Submodule）

## 本节目标

学完这节后，你会：
- 理解 Git 子模块的概念和使用场景
- 添加、克隆、更新子模块
- 在子模块中进行开发工作
- 修改子模块 URL
- 删除子模块
- 处理子模块的常见问题
- 理解 subtree 和 submodule 的区别

---

## 什么是子模块

子模块（submodule）是 Git 中将一个仓库嵌套在另一个仓库内的机制。被嵌套的仓库称为"子仓库"，包含它的仓库称为"父仓库"。

### 基本概念

```
父仓库（my-project/
  ├── src/
  │   └── main.js
  ├── .gitmodules          # 记录子模块配置的文件
  └── submodule/
        └── libs/         # 这是一个独立的 Git 仓库
```

子模块本质上是一个指向特定 commit 的引用，而不是复制代码。这意味着：
- 子模块仓库是独立的，有自己的 `.git` 目录
- 父仓库只记录子模块的 commit hash 引用
- 更新子模块需要显式操作

### 什么时候需要子模块

**适合使用子模块的场景：**

| 场景 | 说明 | 举例 |
|------|------|------|
| 依赖第三方库 | 需要使用外部仓库但希望保持控制权 | 引用一个开源库并定制化 |
| 共享组件 | 多个项目共享同一个组件仓库 | 公司内部 UI 组件库 |
| 微仓库架构 | 将巨型仓库拆分为独立模块 | 前端、后台、文档分开管理 |

**不适合使用子模块的场景：**

| 场景 | 替代方案 | 原因 |
|------|---------|------|
| 简单的依赖管理 | npm/yarn/pip 等包管理工具 | 子模块管理复杂 |
| 只是想用别人的代码 | 直接 fork | 子模块更新麻烦 |
| 团队协作频繁改动 | monorepo | 子模块的 commit 引用频繁失效 |

**注意：** 在决定使用子模块之前，请先评估是否真的需要。如果只是依赖一个库，使用包管理工具通常是更好的选择。

---

## 添加子模块

### 基本语法

```bash
# 在指定目录添加子模块
git submodule add <仓库URL> <本地路径>

# 示例：添加一个公开的库作为子模块
git submodule add https://github.com/lodash/lodash.git libs/lodash
```

### 添加私有仓库作为子模块

```bash
# 使用 SSH 地址
git submodule add git@github.com:company/shared-components.git src/components

# 或者使用 HTTPS（需要配置凭证）
git submodule add https://github.com/company/shared-components.git src/components
```

### 添加特定版本/分支

```bash
# 添加特定标签版本
cd src/components
git checkout v1.2.0
cd ../..
git add src/components
git commit -m "chore: update components to v1.2.0"

# 或者添加时指定分支（默认跟踪该分支）
git submodule add -b main https://github.com/company/ui-lib.git src/ui
# -b 指定分支，但子模块仍会锁定在特定 commit
```

添加子模块后，Git 会创建一个 `.gitmodules` 文件：

```ini
[submodule "src/components"]
    path = src/components
    url = https://github.com/company/shared-components.git
    branch = main
```

**注意：** `.gitmodules` 文件需要提交到仓库，它记录了所有子模块的配置信息。

---

## 克隆包含子模块的仓库

### 克隆后默认不获取子模块内容

当我们克隆一个包含子模块的仓库时，子模块目录是空的：

```bash
# 克隆仓库
git clone https://github.com/company/my-project.git

# 查看子模块目录
ls src/components/
# 空目录！

# 查看状态
git status
# src/components/ (nothing displayed, no commits)
```

### 初始化并更新子模块

```bash
# 方法一：初始化子模块（只需要第一次）
git submodule init

# 更新子模块到记录的 commit
git submodule update

# 方法二：一步到位（常用）
git submodule update --init
```

### 初始化并跟踪分支

```bash
git submodule update --init --remote
# --remote 会获取子模块仓库的最新提交（而不是记录的那个）
```

### 克隆时自动获取子模块

```bash
# 在 clone 命令中指定
git clone --recurse-submodules https://github.com/company/my-project.git

# 如果只需要特定子模块
git clone --recurse-submodules --jobs 4 \
  https://github.com/company/my-project.git
```

### 跨平台注意

Windows 用户在命令行中遇到长命令时，需要注意换行：
```bash
git clone --recurse-submodules https://github.com/company/my-project.git
# 不要手动换行，直接写成一行
```

---

## 更新子模块

子模块的更新有两个维度：

### 将子模块更新到父仓库记录的版本

```bash
# 回到父仓库记录的子模块 commit
git submodule update

# 只更新特定子模块
git submodule update --init src/components
```

### 将子模块更新到远程的最新版本

```bash
# 进入子模块目录
cd src/components

# 获取远程最新代码
git fetch origin

# 查看远程分支
git branch -r

# 切换到远程分支最新提交
git checkout main
git pull origin main

# 返回父仓库
cd ../..

# 提交子模块的更新引用
git add src/components
git commit -m "chore: update components to latest main"
```

### 使用 `--remote` 自动更新

```bash
# 将所有子模块更新到远程最新（不建议用于生产，可能导致不一致）
git submodule update --remote

# 只更新特定子模块
git submodule update --remote src/components
```

**注意：** 使用 `--remote` 更新子模块时要谨慎，因为它可能引入与你本地测试的不同步的代码。企业项目中通常建议锁定子模块版本。

---

## 在子模块中工作

### 进入子模块目录工作

```bash
# 进入子模块目录
cd src/components

# 创建功能分支
git checkout -b feature/new-button

# 开发、提交
git commit -m "feat: add new button style"

# 推送子模块的修改到远程
git push origin feature/new-button

# 返回父仓库
cd ../..
```

### 在子模块中切换分支

```bash
cd src/components
git checkout -b develop
# 在 develop 分支开发
```

### 父仓库查看子模块状态

```bash
# 查看子模块状态
git submodule status
# 输出：abc1234... src/components (develop)
# - abc1234 是当前子模块指向的 commit hash

# 更详细的信息
git submodule status -- warnings
```

### 子模块与父仓库的提交关联

**重要：** 当子模块有新的提交时，必须同时更新父仓库的提交来记录这个新的 commit 引用。

```bash
# 子模块提交后
cd src/components
git commit -m "fix: resolve button click issue"
git push origin develop
cd ../..

# 父仓库必须提交这个引用变更
git add src/components
git commit -m "chore: update components submodule"
git push
```

**注意：** 如果只推送子模块而不更新父仓库，其他成员克隆后会得到不一致的状态。

---

## 修改子模块 URL

### 场景

公司迁移了仓库，或者子模块从 SSH 切换到 HTTPS，需要更新 `.gitmodules` 中的 URL。

### 修改步骤

```bash
# 1. 编辑 .gitmodules 文件
git submodule set-url -- FILL_IN src/components \
  https://github.com/company/new-components.git

# 或者手动编辑 .gitmodules
# 原来的内容：
# [submodule "src/components"]
#     url = https://github.com/company/old-components.git

# 修改为：
# [submodule "src/components"]
#     url = https://github.com/company/new-components.git

# 2. 提交更改
git commit -m "chore: update components submodule URL"

# 3. 更新本地配置（不影响其他成员）
git submodule sync
```

### 跨平台注意

`git submodule set-url` 是 Git 2.8+ 的功能。如果你使用的是旧版 Git，需要手动编辑 `.gitmodules` 和 `.git/config`。

手动编辑方式：

```bash
# 编辑 .git/config 中的 URL
git config submodule.src/components.url https://github.com/company/new-components.git

# 然后提交 .gitmodules 的更改
```

---

## 删除子模块

删除子模块需要几个步骤，不能简单地删除文件。

### 完整删除步骤

```bash
# 1. 取消注册子模块
git submodule deinit src/components
# 这会清除 .git/config 中的子模块配置

# 2. 删除子模块文件
rm -rf src/components
# 跨平台 注意：Windows 用 rmdir /s /q 或使用 PowerShell

# 3. 从暂存区删除
git rm --cached src/components

# 4. 删除 .git/modules 目录（子模块的 git 数据）
rm -rf .git/modules/src/components

# 5. 提交更改
git commit -m "chore: remove components submodule"

# 6. 删除 .gitmodules 中的配置（如果没有其他子模块）
# 如果还有其他子模块，需要编辑 .gitmodules 移除对应条目
```

### 简化方法（Git 2.17+）

```bash
# 移除子模块（Git 2.17+）
git rm src/components
git commit -m "chore: remove components submodule"
```

但这只移除引用，子模块的 git 数据可能仍在 `.git/modules` 中。

**注意：** 删除子模块是危险操作。在执行之前，请确保所有成员都知道这个变更，并且已经提交了使用子模块的相关工作。

---

## 子模块的常见问题与解决

### 问题 1：子模块目录是空白的

**原因：** 克隆后没有初始化子模块。

**解决：**
```bash
git submodule update --init
```

### 问题 2：子模块指向错误的 commit

**原因：** 可能有人在子模块中提交但没有更新父仓库的引用。

**解决：**
```bash
# 更新到子模块分支的最新提交
cd src/components
git checkout main
git pull origin main
cd ../..
git add src/components
git commit -m "chore: sync components to latest"
```

### 问题 3：推送被拒绝

**原因：** 尝试推送时子模块有未同步的远程更新。

**解决：**
```bash
git pull --recurse-submodules
# 或者手动更新子模块
```

### 问题 4：合并冲突

当两个分支都修改了子模块引用时会产生合并冲突。

**解决：**
```bash
# 查看冲突
git status

# 解决冲突 - 选择要保留的子模块版本
git add src/components
git commit -m "Merge: resolve submodule conflict, keeping latest version"
```

### 问题 5：submodule update 每次都回到旧的 commit

**原因：** 父仓库记录的 commit 没有更新。

**解决：**
```bash
# 在主仓库更新子模块引用
cd src/components
git fetch origin
git checkout main
cd ../..
git add src/components
git commit -m "chore: update submodule ref"
```

### 问题 6：Windows 上子模块路径过长

**解决：** Git 有路径长度限制，可以设置：
```bash
git config --global core.longpaths true
```

---

## Subtree vs Submodule

这是两个常被混淆的概念。

### 对比表

| 特性 | Submodule | Subtree |
|------|----------|---------|
| 存储方式 | 独立仓库，父仓库只存引用 | 完整复制到父仓库 |
| `.git` 目录 | 子模块有独立的 `.git` | 只有一个 `.git` |
| 更新方式 | 显式更新引用 | 合并远程更新 |
| 权限控制 | 可以独立控制权限 | 继承父仓库权限 |
| 推送修改 | 需要在子模块中推送 | 在父仓库中直接推送 |
| 学习曲线 | 较复杂 | 较简单 |
| 适用场景 | 共享组件/库 | 依赖第三方代码 |

### Subtree 基本用法

```bash
# 添加 subtree（替代 submodule）
git subtree add --prefix=libs/lodash https://github.com/lodash/lodash.git main

# 更新 subtree
git subtree pull --prefix=libs/lodash https://github.com/lodash/lodash.git main

# 推送修改到远程
git subtree push --prefix=libs/lodash https://github.com/lodash/lodash.git main
```

### 选择建议

| 场景 | 推荐 |
|------|------|
| 需要维护独立的组件仓库 | Submodule |
| 只是单向引入第三方库 | Subtree 或直接用包管理 |
| 团队共享的配置 | Submodule |
| 简单嵌入外部代码 | Subtree |

**注意：** 如果不确定用哪个，优先考虑包管理工具（npm/yarn/pip）。只有在包管理无法满足需求（如需要修改源码并回传）时，才考虑 subtree 或 submodule。

---

## 常见问题

**Q：子模块可以嵌套吗？**
A：可以，子模块可以包含自己的子模块。但层数越多越难管理，通常不建议超过两层。

**Q：子模块可以切换到不同的分支吗？**
A：可以，但切换后需要提交父仓库的更改。子模块在任意时刻都指向某个特定的 commit。

**Q：如何在 CI/CD 中处理子模块？**
A：在 CI 配置中添加子模块初始化步骤：
```yaml
# GitHub Actions 示例
- uses: actions/checkout@v3
  with:
    submodules: true
```

**Q：子模块可以被直接 Fork 吗？**
A：可以，但 Fork 的仓库和原仓库没有关联。需要手动维护同步。

**Q：删除子模块后还能恢复吗？**
A：可以通过重新添加子模块并更新到原来的 commit 来恢复。

---

## 本节总结

```
添加子模块：
  git submodule add <url> <path>

克隆包含子模块的仓库：
  git clone --recurse-submodules <url>
  git submodule update --init

更新子模块：
  git submodule update              # 更新到父仓库记录的版本
  git submodule update --remote   # 更新到远程最新

在子模块中工作：
  cd <submodule-path>
  git checkout -b feature/xxx
  # 开发、提交、推送
  cd ..
  git add <submodule-path>
  git commit -m "chore: update submodule"

修改子模块 URL：
  git submodule set-url <path> <new-url>
  git submodule sync

删除子模块：
  git submodule deinit <path>
  rm -rf <path>
  git rm --cached <path>
  git commit -m "chore: remove submodule"

Submodule vs Subtree：
  - Submodule：独立仓库，父仓库只记录引用
  - Subtree：完整复制，更简单但无独立历史
```

**企业实践：**
1. 评估是否真的需要子模块，优先考虑包管理工具
2. 在文档中记录所有子模块的用途和维护方式
3. 指定专人负责维护子模块更新
4. 在 CI/CD 中配置子模块支持
5. 尽量保持子模块引用稳定，避免频繁更新