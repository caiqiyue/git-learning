# 10. SSH 连接 GitHub

## 本节目标

学完这节后，你会：
- 理解 SSH 是什么，为什么要用它
- 生成 SSH 密钥对
- 把公钥配置到 GitHub
- 用 SSH 方式克隆和推送代码

---

## 为什么需要 SSH

当你执行 `git push` 时，需要身份验证。两种方式：

| 方式 | 每次操作 | 配置难度 | 安全性 |
|------|---------|---------|--------|
| HTTPS | 需要输入用户名 + Token | 每次都要输入 | ⚠️ Token 泄露风险 |
| SSH | 一次配置，之后不用输入 | 配置一次即可 | ✅ 更安全 |

**SSH 的优势：**
- 配置一次，之后 `git push`/`git pull` 不需要输入密码
- 比 HTTPS 更安全
- 企业开发标配

---

## 生成 SSH 密钥对

### Step 1：检查是否已有密钥

```bash
# 查看是否已有 SSH 密钥
ls ~/.ssh

# 或者
cat ~/.ssh/id_rsa.pub
cat ~/.ssh/id_ed25519.pub
```

如果已经有 `.pub` 文件，可以跳过生成步骤。

### Step 2：生成新密钥

**推荐使用 Ed25519 算法（现代、安全、快速）**

```bash
ssh-keygen -t ed25519 -C "你的邮箱@example.com"
```

**或者使用 RSA 算法（兼容旧系统）**

```bash
ssh-keygen -t rsa -b 4096 -C "你的邮箱@example.com"
```

### Step 3：输入保存位置和密码

```bash
Generating public/private ed25519 key.
Enter file in which to save the key (/c/Users/你的用户名/.ssh/id_ed25519):  # 直接回车默认
Enter passphrase (empty for no passphrase):  # 输入密码（可以留空直接回车）
Enter same passphrase again:  # 确认密码
```

**🌍 注意：**
- Windows Git Bash：密钥默认保存在 `~/.ssh/`
- macOS/Linux：同样是 `~/.ssh/`

---

## 查看公钥

```bash
cat ~/.ssh/id_ed25519.pub
# 输出：
# ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... 你的邮箱@example.com
```

或者：

```bash
clip < ~/.ssh/id_ed25519.pub    # Windows 复制到剪贴板
pbcopy < ~/.ssh/id_ed25519.pub  # macOS 复制到剪贴板
```

**⚠️ 是 `.pub` 文件，不是私钥文件！**

---

## 配置到 GitHub

### Step 1：登录 GitHub

打开 https://github.com

### Step 2：进入 SSH 设置

```
Settings → SSH and GPG keys → New SSH key
```

### Step 3：填写信息

```
Title: （随便写，比如 "My Laptop" 或 "Windows Work PC"）
Key type: Authentication Key
Key: （粘贴公钥内容，就是 ~/.ssh/id_ed25519.pub 的全部内容）
```

### Step 4：验证连接

```bash
ssh -T git@github.com
```

如果看到：

```
Hi username! You've successfully authenticated, but GitHub does not provide shell access.
```

说明连接成功！

---

## 配置 SSH 替代 HTTPS

### 方式一：创建 config 文件（推荐）

```bash
# 创建 config 文件
touch ~/.ssh/config

# 编辑 config 文件，添加：
# Host github.com
#     HostName github.com
#     User git
#     IdentityFile ~/.ssh/id_ed25519
```

### 方式二：修改远程仓库 URL

```bash
# 查看当前远程 URL
git remote -v
# 输出：
# origin  https://github.com/username/repo.git (fetch)
# origin  https://github.com/username/repo.git (push)

# 修改为 SSH 方式
git remote set-url origin git@github.com:username/repo.git

# 验证修改
git remote -v
# 输出：
# origin  git@github.com:username/repo.git (fetch)
# origin  git@github.com:username/repo.git (push)
```

---

## 克隆使用 SSH

```bash
git clone git@github.com:username/repo-name.git
# 不再需要输入用户名和密码
```

---

## 常见问题

**Q：提示 "Could not resolve hostname" 怎么解决？**
A：Windows 上可能需要以管理员身份打开 Git Bash，或者重启终端。

**Q：密码和 passphrase 有什么区别？**
A：密码是访问私钥的密码（如果你设置了的话）。passphrase 是在生成 SSH 密钥时设置的额外保护，可以留空。

**Q：可以在多台电脑上用同一个 SSH 密钥吗？**
A：可以，但不是最佳实践。建议每个设备生成单独的 SSH 密钥，然后在 GitHub 上添加多个公钥。

**Q：公钥和私钥有什么区别？**
A：公钥（`.pub` 文件）可以公开，放在 GitHub 上；私钥要保密，存在你的电脑里。SSH 认证用私钥加密，GitHub 用公钥解密验证。

**Q：GitHub 提示 "Key is already in use"**
A：说明这个公钥已经被其他账户或仓库使用了。每个公钥只能绑定到一个 GitHub 账户。

---

## 本节总结

```
生成 SSH 密钥：
  ssh-keygen -t ed25519 -C "邮箱"

查看公钥：
  cat ~/.ssh/id_ed25519.pub

配置到 GitHub：
  1. 复制公钥内容
  2. GitHub → Settings → SSH and GPG keys → New SSH key
  3. 粘贴公钥

验证连接：
  ssh -T git@github.com

修改远程 URL 为 SSH：
  git remote set-url origin git@github.com:username/repo.git
```