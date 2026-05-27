# 14. Commit Message 规范

## 本节目标

学完这节后，你会：
- 理解为什么 commit message 要规范
- 掌握 Conventional Commits 格式
- 写出专业的 commit message
- 理解 CI 中 commitlint 的作用

---

## 为什么要规范 commit message

**没有规范的 commit message：**

```
commit 1: update
commit 2: fix bug
commit 3: asdfasdf
commit 4: merged from feature
```

**有规范的 commit message：**

```
a1b2c3d feat(auth): add JWT token authentication
e4f5g6h fix(payment): resolve null pointer when amount is null
i7j8k9l refactor(order): extract discount calculation to utility
```

**规范的好处：**
- `git log --oneline` 一眼看懂项目演进
- 自动生成 CHANGELOG
- Code Review 时快速理解改动意图
- `git log --grep="fix:"` 快速筛选所有修复

---

## Conventional Commits 格式

业界最流行的规范是 **Conventional Commits**，格式如下：

```
<type>(<scope>): <subject>

[optional body]

[optional footer]
```

### 第一行格式规则

- 最多 **72 字符**
- 用**现在时**："add" 不是 "added"，"fix" 不是 "fixed"
- 句尾**不加句号**
- 标题即结论，不需要完整句子

---

## Type 完整列表

| Type | 何时使用 | 示例 |
|------|---------|------|
| `feat` | 新功能（用户可见） | `feat(payment): add credit card support` |
| `fix` | 修复 bug | `fix(auth): resolve JWT token expiry edge case` |
| `refactor` | 重构（不改行为） | `refactor(api): extract validation layer` |
| `perf` | 性能优化 | `perf(db): add index on orders.user_id` |
| `test` | 测试相关 | `test(cart): add unit tests for discount` |
| `docs` | 文档更新 | `docs(readme): update installation steps` |
| `style` | 代码格式（不影响功能） | `style(format): fix tab/space issue` |
| `build` | 构建系统或依赖变更 | `build(webpack): upgrade to v5` |
| `ci` | CI 配置变更 | `ci(github): add matrix build strategy` |
| `chore` | 其他杂项 | `chore(deps): bump lodash version` |
| `revert` | 回滚提交 | `revert: revert abc1234` |

---

## Scope 规范

Scope 是**受影响模块/包/服务名**，让 commit 定位精确到具体位置：

```
✅ feat(user): add password reset flow
✅ feat(auth): add password reset flow
❌ feat: add password reset flow          （太泛）

✅ feat(order-service): add bulk export API    （微服务场景）
✅ feat(shared-lib): add HttpClient wrapper    （公共库场景）
```

---

## Commit Message 示例

### 基础格式

```bash
git commit -m "feat(auth): add JWT-based authentication"
git commit -m "fix(payment): resolve null pointer when amount is null"
git commit -m "docs(api): update user endpoint signature"
```

### 带详细说明

```bash
git commit -m "feat(payment): support split payment for orders over 1000

- Add PaymentStrategy interface
- Implement AlipayStrategy and WechatStrategy
- Refactor PaymentController to use strategy pattern
- Add integration tests for each payment gateway

Closes #234"
```

### BREAKING CHANGE

破坏性变更必须在 footer 标注：

```bash
git commit -m "refactor(api): change /user response structure

BREAKING CHANGE: /user endpoint now returns {data: User} instead of User.
All clients must update their parsers before upgrading.

Closes #199"
```

---

## Commit Message 速查表

| 场景 | Type | 示例 |
|------|------|------|
| 新增用户登录功能 | `feat` | `feat(auth): add login with magic link` |
| 修复登录失败 bug | `fix` | `fix(auth): resolve login timeout when network is slow` |
| 重构登录模块 | `refactor` | `refactor(auth): extract token service to separate class` |
| 优化登录性能 | `perf` | `perf(auth): cache user permissions to reduce DB queries` |
| 新增登录测试 | `test` | `test(auth): add unit tests for token refresh logic` |
| 更新登录文档 | `docs` | `docs(auth): update authentication flow diagram` |
| 格式化代码 | `style` | `style(auth): format code with prettier` |
| 升级依赖 | `build` | `build(deps): upgrade react to 18.2` |
| 修改 CI 配置 | `ci` | `ci: add auth service to e2e pipeline` |
| 更新 gitignore | `chore` | `chore: ignore .env files` |
| 回退错误提交 | `revert` | `revert: revert abc1234 due to regression` |

---

## 常见错误

```bash
# ❌ 错误示例
git commit -m "update"                    # 太模糊
git commit -m "fix bug"                   # 不知道修了什么
git commit -m "asdfasdf"                  # ...
git commit -m "feat: Added user login."   # 用了过去式
git commit -m "feat(auth): add login."    # 句尾加了句号

# ✅ 正确示例
git commit -m "feat(auth): add user login with magic link"
git commit -m "fix(auth): resolve login timeout when network is slow"
```

---

## Commitlint CI 自动检查

企业项目通常在 CI 中集成 `commitlint`，不符合规范的提交直接被拒绝：

```yaml
# .github/workflows/commitlint.yml
name: Commitlint
on: [push, pull_request]
jobs:
  commitlint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: wagoid/commitlint-github-action@v5
```

```javascript
// .commitlintrc.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [2, 'always', [
      'feat', 'fix', 'refactor', 'perf', 'test',
      'docs', 'style', 'build', 'ci', 'chore', 'revert'
    ]],
    'subject-full-stop': [2, 'never', '.'],    # [2, 'never', '.'] = 禁止句号结尾
    'subject-max-length': [2, 'always', 72]   # [2, 'always', 72] = 限制 72 字符
  }
};
```

---

## 本节总结

```
格式：<type>(<scope>): <subject>

Type：
  feat    新功能
  fix     修复 bug
  refactor 重构
  perf    性能优化
  test    测试
  docs    文档
  style   格式
  build   依赖/构建
  ci      CI 配置
  chore   杂项
  revert  回退

规则：
  - 最多 72 字符
  - 现在时（add 不是 added）
  - 不加句号
  - scope 精确到模块
```