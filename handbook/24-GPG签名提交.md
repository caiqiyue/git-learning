# 24. GPG 签名提交

## 本节目标

学完这节后，你会：
- 理解 GPG 签名提交的作用和重要性
- 生成并管理 GPG 密钥对
- 配置 Git 使用 GPG 签名
- 在 GitHub/GitLab 上绑定签名密钥
- 验证提交签名状态
- 解决常见的签名问题

---

## 什么是 GPG 签名提交

### 传统提交的问题

没有签名的提交存在以下风险：

```
提交者可以伪装成任何人
commit 1234567 Author: Elon Musk <elon@tesla.com>  ← 谁都能写
commit 8901234 Author: 你的同事 <attacker@evil.com> ← 伪造身份
```

**攻击场景：**
- 攻击者假冒开发者提交恶意代码
- 内部人员伪装成其他同事提交有问题的代码
- 在开源项目中，假扮知名开发者的身份

### GPG 签名解决什么问题

GPG 签名通过**非对称加密**证明提交者的真实身份：

```
我的私钥（只有我知道）────sign────→  提交获得签名
我的公钥（在 GitHub/GitLab）────verify──→  任何人都能验证
```

**签名后的效果：**

```
commit a1b2c3d Verified [GPG] feat(auth): add login
     └── GitHub 显示 "Verified" 标签，证明是我提交的
```

---

## GPG 密钥基础

### 工作原理简述

GPG 使用**密钥对**工作：

| 密钥 | 作用 | 保管 |
|------|------|------|
| 私钥（Private Key） | 对数据签名 | **必须保密**，只有我能访问 |
| 公钥（Public Key） | 验证签名 | 可以公开，放在 GitHub/GitLab |

---

### 生成 GPG 密钥

**注意：生成前先确认 gpg 已安装**

```bash
# 检查 gpg 是否安装
gpg --version
# 输出：gpg (GnuPG) 2.4.x
```

```bash
# 生成密钥（交互式）
gpg --full-generate-key

# 我选择的配置：
# 选择类型：RSA and RSA
# 密钥长度：4096（企业推荐，安全度更高）
# 过期时间：0（永不过期，或设置 1-2 年）
# 真实姓名： 你的姓名
# 邮箱： your@email.com
# 密码： 设置一个强密码（不要和登录密码相同）
```

**跨平台差异：**

| 平台 | gpg 安装方式 |
|------|-------------|
| macOS | `brew install gpg` + `brew install pinentry-mac` |
| Ubuntu/Debian | `sudo apt install gnupg` |
| Windows | Gpg4win（https://www.gpg4win.org/） |
| Arch Linux | `sudo pacman -S gnupg` |

### 查看密钥

```bash
# 查看私钥（列出 Key ID）
gpg --list-secret-keys --keyid-format=long

# 输出示例：
# sec   rsa4096/ABC123DEF456 2026-05-27 [SC]
# uid     caiqiyue (my gpg key) <caiqiyue@example.com>
# 这里的 ABC123DEF456 就是我的 Key ID
```

```bash
# 查看公钥
gpg --list-public-keys --keyid-format=long
```

### 导出公钥

```bash
# 导出公钥（用于添加到 GitHub）
gpg --armor --export ABC123DEF456

# 输出类似于：
# -----BEGIN PGP PUBLIC KEY BLOCK-----
# mQINBF...
# -----END PGP PUBLIC KEY BLOCK-----
```

**注意：只导出公钥，私钥绝对不能分享给任何人！**

### 备份密钥（企业推荐）

```bash
# 备份私钥（加密的，很重要！）
gpg --armor --export-secret-keys ABC123DEF456 > private-key.asc

# 备份公钥
gpg --armor --export ABC123DEF456 > public-key.asc

# 将备份文件存放在安全的位置（加密硬盘、USB 等）
```

---

## 配置 Git 使用 GPG

### 1. 设置全局签名密钥

```bash
# 配置 Git 使用我的 GPG 密钥（Key ID 写你自己的）
git config --global user.signingkey ABC123DEF456

# 开启默认签名提交
git config --global commit.gpgsign true
```

### 2. 配置 gpg 程序路径

**跨平台差异：**

```bash
# Linux/macOS 通常不需要配置
# Windows 可能需要指定 gpg 路径
git config --global gpg.program "C:/Program Files/GnuPG/bin/gpg.exe"
```

### 3. 验证配置

```bash
# 查看所有 GPG 相关配置
git config --list | grep -i gpg

# 输出：
# user.signingkey=ABC123DEF456
# commit.gpgsign=true
```

---

## 签署提交

### 基本签署命令

```bash
# 签署提交（-S 是 --gpg-sign 的缩写）
git commit -S -m "feat(auth): add login feature"

# 执行后需要输入私钥密码
# ┌─────────────────────────────────────┐
# │ 请输入密码：************************ │
# └─────────────────────────────────────┘
```

**注意：首次使用会要求输入私钥密码，之后如果启用了 gpg-agent，短时间内不需要重复输入**

### 签署标签

```bash
# 签署标签
git tag -s v1.0.0 -m "Release version 1.0.0"

# 验证标签签名
git tag -v v1.0.0
```

### 签署合并提交

```bash
# 签署合并提交
git merge --no-ff develop --gpg-sign
```

---

## GitHub/GitLab 配置签名密钥

### GitHub 配置

**Step 1：复制我的公钥**

```bash
gpg --armor --export ABC123DEF456
# 全选复制输出的内容
```

**Step 2：添加到 GitHub**

1. 访问 GitHub → Settings → SSH and GPG keys
2. 点击 "New GPG key"
3. 粘贴公钥内容
4. 点击 "Add GPG key"

**Step 3：开启签名提交**

1. 访问 GitHub → 你的仓库 → Settings → Branches
2. 在 "Branch protection rules" 中可以要求签名提交
3. 或者在仓库 Settings → Archives 中勾选 "Sign commits"

### GitLab 配置

1. 访问 GitLab → User Settings → GPG Keys
2. 粘贴公钥内容
3. 点击 "Add key"

**注意：GitLab 要求签名者的公钥必须先添加才能验证通过**

---

## 验证签名

### 查看提交的签名状态

```bash
# 查看单个提交的 GPG 签名
git show --format=raw -s abc1234

# 输出包含：
# commit abc1234
# gpg: Signature made ...
# gpg: Good signature from "caiqiyue <caiqiyue@example.com>"
```

### 查看分支所有提交的签名状态

```bash
# 查看最近 10 条提交的签名状态
git log --format="%H %G? %an: %s" -10

# %G? 的含义：
# G = Good（有效签名）
# B = Bad（无效签名）
# U = Good but untrusted（有效但不信任）
# N = No signature（无签名）
```

### 使用 gpg 直接验证

```bash
# 验证某个提交
git verify-commit abc1234

# 输出：
# gpg: Signature made 2026-05-27 10:30:00
# gpg: Good signature from "caiqiyue <caiqiyue@example.com>"
```

### GitHub 界面验证

签名验证通过后，GitHub 会在提交旁边显示：

```
commit abc1234 Verified [GPG] feat(auth): add login
                     ↑
              绿色 "Verified" 标签
```

**未签名或签名无效的提交：**

```
commit abc1234 Unverified feat(auth): add login
                     ↑
              灰色 "Unverified" 标签
```

---

## 信任模型和 Key Signing

### Web of Trust（信任网络）

GPG 使用"信任网络"模型：

```
我 ──sign──→ 同事A 的公钥（我信任同事A的身份）
同事A ──sign──→ 同事B 的公钥（通过同事A，我可以间接信任同事B）
```

### 在 GitHub 中建立信任

GitHub/GitLab 充当"可信第三方"，不需要传统的 Web of Trust：

```
GitHub 验证签名者的身份（通过 GPG Keys 配置）
         ↓
用户提交带签名的 commit
         ↓
GitHub 用提交者的公钥验证签名
         ↓
验证通过 → 显示 "Verified"
```

### 多设备使用

如果你有多个电脑：

```bash
# 电脑 A：生成密钥对
gpg --full-generate-key
# Key ID: ABC123DEF456

# 电脑 A：导出私钥
gpg --armor --export-secret-keys ABC123DEF456 > my-private-key.asc

# 电脑 B：导入私钥
gpg --import my-private-key.asc

# 电脑 B：信任导入的密钥（需要本地验证）
gpg --edit-key ABC123DEF456
# 在 gpg 交互界面输入：trust
# 选择 5（I trust ultimately）
```

---

## 常见问题解决

### GPG 密钥丢失怎么办

**情况 1：还没推送**

```bash
# 如果私钥丢了，但有备份
gpg --import backup.asc

# 如果私钥彻底丢了
# 只能重新生成新密钥
gpg --full-generate-key

# 更新 GitHub/GitLab 上的公钥为新密钥
```

**情况 2：已经推送，且需要撤销旧密钥的信任**

1. 在 GitHub → Settings → GPG Keys 中删除旧公钥
2. 通知团队成员不要再使用旧公钥验证

### 签名验证失败排查

**问题 1：GitHub 显示 "Unverified"**

排查步骤：

```bash
# Step 1：确认 GitHub 添加的公钥和本地使用的一致
gpg --list-keys --keyid-format=long

# Step 2：确认 commit 时使用了正确的密钥签名
git log --format="%G?" -1
# 输出 N = No signature，说明提交时没签名

# Step 3：确认 Git 全局配置了 signing key
git config --global user.signingkey
```

**问题 2：gpg 找不到密钥**

```bash
# 查看所有密钥
gpg --list-keys

# 如果密钥列表为空，说明公钥没正确导入
# 重新导入：
gpg --import public-key.asc
```

**问题 3：gpg-agent 缓存过期每次都要输入密码**

**跨平台差异：**

```bash
# macOS：使用 pinentry-mac
# 编辑 ~/.gnupg/gpg-agent.conf
enable-ssh-support
pinentry-program /usr/local/bin/pinentry-mac

# Linux：配置 gpg-agent
echo "pinentry-program /usr/bin/pinentry-gnome3" >> ~/.gnupg/gpg-agent.conf

# Windows：Gpg4win 自带配置
# 重启 gpg-agent：
gpgconf --reload gpg-agent
```

**问题 4：Windows 上签名失败**

```bash
# Windows 专用配置
git config --global gpg.program "C:/Program Files (x86)/GnuPG/bin/gpg.exe"

# 或者使用 WSL
git config --global gpg.program wslpath "$(which gpg)"
```

**问题 5：commit 忘记加 -S 怎么办**

```bash
# 如果已经 commit 了但没签名
# 只能 amend（但要注意，只用于本地未推送的提交）
git commit --amend --no-edit -S

# 如果已经 push 到远程，不要 amend
# 这时需要在远程手动删除错误的提交
```

### 性能问题

签名操作本身很快，但如果每次输入密码很慢：

```bash
# macOS 配置钥匙串
git config --global credential.helper osxkeychain

# Linux 配置 gnome-keyring 或 gpg-agent
```

---

## 企业级应用场景

### 场景 1：开源项目签名要求

很多知名开源项目要求所有提交签名：

```bash
# Kubernetes 项目提交规范
# 要求：Every commit must be signed

# 如果没签名，CI 会失败：
# Error: Required Signed-off-by missing
# Error: GPG signature verification failed
```

### 场景 2：企业内部代码审计

企业可以配置：

1. **强制签名策略**：所有分支的合并都需要签名提交
2. **签名验证 CI**：CI 中检查每条提交链的签名完整性
3. **审计日志**：记录每个签名的验证状态

```bash
# GitLab 配置强制签名
# Settings → Repository → Protected branches
# 勾选 "Require signed commits"
```

### 场景 3：供应链安全

签名可以防止供应链攻击：

```
攻击者试图植入恶意代码
        ↓
攻击者没有我的私钥，无法签名
        ↓
其他开发者在 pull 代码时发现签名验证失败
        ↓
发现异常，拒绝合并
```

### 企业 GPG 基础设施建议

| 需求 | 推荐方案 |
|------|----------|
| 密钥管理 | 使用硬件密钥（如 YubiKey）存储私钥 |
| 密钥轮换 | 每年生成新密钥，旧密钥保留用于验证历史提交 |
| 团队密钥托管 | 使用 KMS（Key Management Service）统一管理 |
| 审计 | 配置 CI 检查所有提交链的签名完整性 |

---

## 本节总结

```
GPG 签名提交流程：

1. 生成密钥：
   gpg --full-generate-key

2. 导出公钥并添加到 GitHub/GitLab：
   gpg --armor --export <key-id>

3. 配置 Git：
   git config --global user.signingkey <key-id>
   git config --global commit.gpgsign true

4. 签署提交：
   git commit -S -m "feat: add feature"

5. 验证签名：
   git log --format="%G?" -1
   GitHub 显示 "Verified" 标签

⚠️ 注意事项：
   - 私钥绝对不能泄露
   - 跨平台注意 gpg 安装和配置差异
   - 已 push 的提交不要 amend
   - 建议备份私钥
```