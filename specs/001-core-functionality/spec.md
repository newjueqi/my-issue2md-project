# Spec #001: 核心功能 - GitHub 内容转 Markdown

## 📋 用户故事

### CLI 版本（MVP）

> 作为一个开发者，我希望能够将 GitHub Issue/PR/Discussion 快速转换为本地 Markdown 文件，以便离线归档和检索重要的技术讨论。

**典型工作流**:
```bash
# 输出到文件
issue2md https://github.com/owner/repo/issues/123 my-doc.md

# 输出到 stdout（便于管道操作）
issue2md https://github.com/owner/repo/pull/456 | grep "TODO"

# 访问私有仓库
export GITHUB_TOKEN=ghp_xxx
issue2md https://github.com/owner/private-repo/issues/789
```

### Web 版本（未来）

> 作为一个非技术用户，我希望通过浏览器访问一个网站，粘贴 GitHub URL 后直接下载 Markdown 文件，无需安装任何工具。

---

## ✅ 功能性需求

### 1. URL 识别与解析

**需求**: 工具必须能够自动识别并解析三种 GitHub 资源类型。

| URL 模式 | 资源类型 | 示例 |
|----------|----------|------|
| `*/issues/*` | Issue | `https://github.com/golang/go/issues/123` |
| `*/pull/*` | Pull Request | `https://github.com/golang/go/pull/456` |
| `*/discussions/*` | Discussion | `https://github.com/community/discussions/789` |

**实现细节**:
- 使用正则表达式或 URL 解析库
- 识别失败时返回明确的错误信息
- 支持 HTTP 和 HTTPS 协议

### 2. 命令行接口

**语法**:
```bash
issue2md [flags] <url> [output_file]
```

**参数**:
- `<url>` (必需): GitHub 资源的完整 URL
- `[output_file]` (可选): 输出文件路径
  - 省略时输出到 stdout
  - 提供时输出到指定文件
  - 文件名基于资源标题生成（特殊字符转义）

**Flags**:

| Flag | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `-enable-reactions` | bool | `false` | 包含评论的 emoji reactions |
| `-enable-user-links` | bool | `false` | 将 `@username` 转换为 GitHub 链接 |

### 3. 认证机制

**环境变量**:
```bash
export GITHUB_TOKEN=ghp_xxxxxxxxxxxxxxxxxxxx
```

**规则**:
- **不提供** `--token` 参数（防止 Shell 历史泄露）
- 公开仓库无需认证
- 私有仓库需要设置 `GITHUB_TOKEN`
- Token 可提高 API 限额（5000 次/小时 vs 60 次/小时）

### 4. 内容提取

#### 4.1 元数据（必须包含）

| 字段 | Issue | PR | Discussion |
|------|-------|-----|------------|
| 标题 | ✅ | ✅ | ✅ |
| 作者 | ✅ | ✅ | ✅ |
| 创建时间 | ✅ | ✅ | ✅ |
| 状态 | ✅ | ✅ | ✅ |
| 标签 | ✅ | ✅ | ❌ |
| 分支信息 | ❌ | ✅ | ❌ |
| 关联 Issue | ❌ | ✅ | ❌ |

**时间格式**: `2024-01-15 14:30:00`（本地时间）

#### 4.2 正文内容

- ✅ 标题和原始正文
- ✅ 所有评论和回复
- ✅ PR 的代码审查评论

#### 4.3 代码差异（仅 PR）

- 使用代码块内嵌到 Markdown
- 包含语言标识（如 ` ```go `）

**示例**:
````markdown
```diff
--- a/main.go
+++ b/main.go
@@ -1,3 +1,4 @@
 package main

+func newFeature() {}
```
````

### 5. 输出格式规范

#### 5.1 Markdown 结构

```markdown
# [Type #123] Resource Title

**Metadata:**
- **Author:** @username
- **Created:** 2024-01-15 14:30:00
- **Status:** Open/Closed/Merged
- **Labels:** bug, enhancement (如果适用)
- **Branch:** main -> fix-bug (仅 PR)
- **Related Issues:** #456 (仅 PR)

---

## Body

The original body content...

---

## Comments

### @user2 - 2024-01-15 15:00:00

Comment content...

### @user3 - 2024-01-15 16:00:00

Another comment...
```

#### 5.2 排序规则

- **评论排序**: 按时间正序（从早到晚）
- **PR Review Comments**: 与普通评论一起按时间线平铺
  - 不按 Review 分组
  - 目标：展示完整的对话时间线

#### 5.3 特殊标记

**Discussion Answer**:
```markdown
### @author - 2024-01-15 17:00:00 ✅

> This comment is marked as the answer.

Answer content...
```

#### 5.4 可选内容（通过 Flags 控制）

**Reactions** (`-enable-reactions`):
```markdown
### @user - 2024-01-15 15:00:00
Reactions: 👍 (3), ❤️ (1), 🎉 (2)

Comment content...
```

**用户链接** (`-enable-user-links`):
```markdown
### [(@user)](https://github.com/user) - 2024-01-15 15:00:00

Comment content mentioning [(@another)](https://github.com/another)...
```

### 6. 媒体与链接处理

| 内容类型 | 处理方式 |
|----------|----------|
| 图片 | 保持原始在线链接 |
| 代码块 | 添加语言标识（` ```go `） |
| 普通链接 | 保持不变 |
| @提及 | 默认纯文本，`-enable-user-links` 时转链接 |

### 7. 错误处理

**策略**: 快速失败（Fast Fail）

| 错误场景 | 错误信息示例 | 退出码 |
|----------|--------------|--------|
| 无效 URL | `Error: invalid GitHub URL format` | 1 |
| 资源不存在 | `Error: resource not found (may be private or deleted). Set GITHUB_TOKEN to access private repositories.` | 1 |
| 网络错误 | `Error: failed to fetch data: connection timeout` | 1 |
| API 限流 | `Error: GitHub API rate limit exceeded. Set GITHUB_TOKEN for higher limit.` | 1 |

---

## 🏗️ 非功能性需求

### 1. 架构解耦

**目标**: 核心逻辑与 GitHub API 实现解耦，便于：
- 单元测试（使用 fake 实现）
- 未来支持其他平台（GitLab、Gitea）
- Web 版本复用核心逻辑

**模块结构**:
```
internal/
├── core/           # 核心业务逻辑
│   ├── converter.go
│   └── models.go
├── github/         # GitHub API 实现
│   ├── client.go
│   └── parser.go
└── formatter/      # Markdown 格式化
    └── markdown.go
```

**接口定义**:
```go
// 核心 Converter 依赖抽象接口
type ResourceFetcher interface {
    FetchIssue(ctx, owner, repo, number) (*Issue, error)
    FetchPullRequest(ctx, owner, repo, number) (*PR, error)
    FetchDiscussion(ctx, owner, repo, number) (*Discussion, error)
}
```

### 2. 性能要求

| 指标 | 要求 |
|------|------|
| 单个 Issue 导出时间 | < 3 秒（典型 100 条评论） |
| 内存占用 | < 100 MB（大型 Discussion） |
| 并发支持 | 未来支持批量处理时考虑 |

### 3. 兼容性

| 平台 | 支持版本 |
|------|----------|
| Go | >= 1.24 |
| OS | Linux, macOS, Windows |
| Shell | Bash, Zsh, PowerShell |

### 4. 可维护性

- ✅ 遵循 Go 标准库优先原则
- ✅ 表格驱动测试
- ✅ 集成测试使用真实 GitHub API
- ✅ 错误包装使用 `fmt.Errorf("context: %w", err)`

---

## 🧪 验收标准

### 单元测试用例

#### URL 解析器

| 测试用例 | 输入 | 期望输出 |
|----------|------|----------|
| 有效 Issue URL | `https://github.com/owner/repo/issues/123` | `{Type: Issue, Owner: owner, Repo: repo, Number: 123}` |
| 有效 PR URL | `https://github.com/owner/repo/pull/456` | `{Type: PR, Owner: owner, Repo: repo, Number: 456}` |
| 有效 Discussion URL | `https://github.com/owner/repo/discussions/789` | `{Type: Discussion, Owner: owner, Repo: repo, Number: 789}` |
| 无效 URL | `https://example.com/page` | `error: invalid GitHub URL` |
| 缺少编号 | `https://github.com/owner/repo/issues` | `error: missing resource number` |

#### 文件名生成器

| 测试用例 | 输入标题 | 期望输出 |
|----------|----------|----------|
| 正常标题 | `Fix bug in API` | `fix-bug-in-api.md` |
| 特殊字符 | `Feature: Add /support` | `feature-add--support.md` |
| 空标题 | `""` | `issue-123.md` |
| 超长标题 | 300 字符 | 截断至 255 字符 + `.md` |

#### 时间格式化

| 测试用例 | 输入 (UTC) | 期望输出 (本地时间) |
|----------|------------|---------------------|
| 标准时间 | `2024-01-15T06:30:00Z` | `2024-01-15 14:30:00` (UTC+8) |

### 集成测试用例

#### 端到端测试

| 测试用例 | 步骤 | 验证点 |
|----------|------|--------|
| 导出公开 Issue | 运行 `issue2md <公开Issue URL>` | ✅ 输出包含标题、正文、评论<br>✅ 元数据完整<br>✅ 时间格式正确 |
| 导出 PR | 运行 `issue2md <PR URL>` | ✅ 包含代码差异<br>✅ 包含分支信息<br>✅ Review Comments 按时间排序 |
| 导出 Discussion | 运行 `issue2md <Discussion URL>` | ✅ Answer 有特殊标记<br>✅ 评论完整 |
| 输出到文件 | 运行 `issue2md <URL> output.md` | ✅ 文件创建成功<br>✅ 内容正确 |
| 私有仓库 | 设置 Token，运行 `issue2md <私有URL>` | ✅ 成功导出 |
| 无 Token 访问私有 | 不设置 Token，运行 `issue2md <私有URL>` | ❌ 返回错误提示 |
| `-enable-reactions` | 运行 `issue2md -enable-reactions <URL>` | ✅ 包含 Reactions 信息 |
| `-enable-user-links` | 运行 `issue2md -enable-user-links <URL>` | ✅ @用户名转换为链接 |

#### 错误场景测试

| 测试用例 | 输入 | 期望行为 |
|----------|------|----------|
| 资源不存在 | `https://github.com/owner/repo/issues/99999999` | ❌ 退出码非零，错误信息清晰 |
| 网络断开 | 任意 URL（无网络） | ❌ 超时或连接错误 |
| 无效 URL | `https://not-github.com/issue/1` | ❌ 明确提示 URL 格式错误 |

---

## 📄 输出格式示例

### 示例 1: Issue

```markdown
# [Issue #123] Fix: Memory leak in HTTP handler

**Metadata:**
- **Author:** @johndoe
- **Created:** 2024-01-15 14:30:00
- **Status:** Closed
- **Labels:** bug, high-priority

---

## Body

I've noticed a memory leak when using the HTTP handler with long-lived connections. The goroutines are not being cleaned up properly.

**Steps to reproduce:**
1. Start the server
2. Send 1000 requests
3. Check goroutine count

**Expected:** Goroutine count returns to baseline
**Actual:** Goroutine count keeps increasing

---

## Comments

### @janedoe - 2024-01-15 15:00:00

I can reproduce this issue. It looks like the `Close()` method is not being called on the response writer.

### @johndoe - 2024-01-15 15:30:00

Thanks @janedoe, I'll investigate the `Close()` method. I suspect it's related to the middleware chain.

### @devops - 2024-01-16 09:00:00

This is blocking our deployment. Any update?

### @johndoe - 2024-01-16 10:00:00

Fixed in PR #124. The issue was that we were deferring `Close()` in the wrong place.
```

### 示例 2: Pull Request

```markdown
# [PR #124] Fix memory leak in HTTP handler

**Metadata:**
- **Author:** @johndoe
- **Created:** 2024-01-16 11:00:00
- **Status:** Merged
- **Branch:** fix/memory-leak -> main
- **Related Issues:** #123

---

## Body

Fixes #123

**Changes:**
- Move `Close()` call to the correct location
- Add defer in the middleware chain
- Add unit test to prevent regression

---

## Comments

### @janedoe - 2024-01-16 11:30:00

Great fix! The code looks much cleaner now. Just one question: should we also add a timeout?

### @reviewer - 2024-01-16 12:00:00

LGTM with one suggestion.

```diff
- func handler(w http.ResponseWriter, r *http.Request) {
+ func handler(w http.ResponseWriter, r *http.Request) error {
      defer w.Close()
```

Returning an error would make it easier to add middleware later.

### @johndoe - 2024-01-16 13:00:00

Good point @reviewer! Updated.

### @bot - 2024-01-16 14:00:00

CI passed. All checks green.

### @maintainer - 2024-01-16 15:00:00

Merged! Thanks for the quick fix.
```

### 示例 3: Discussion

```markdown
# [Discussion #789] Best practices for error handling in Go 1.24?

**Metadata:**
- **Author:** @gopher123
- **Created:** 2024-01-10 09:00:00
- **Status:** Open

---

## Body

With the new error handling proposals in Go 1.24, what are the recommended patterns for handling errors in large codebases?

Should we:
1. Stick with traditional `if err != nil`
2. Use the new `try` function
3. Adopt a hybrid approach

---

## Comments

### @senior-dev - 2024-01-10 10:00:00 ✅

> This comment is marked as the answer.

After working with the beta, I recommend the hybrid approach:

- Use `try` for simple operations (file I/O, HTTP calls)
- Use traditional `if err != nil` for business logic where you need context

Here's an example:
```go
func processData() error {
    data := try(os.ReadFile("data.json"))
    return processBusinessLogic(data) // Explicit check for domain errors
}
```

The key is to use `try` where the error path is obvious and traditional checks where you need to add context or make decisions.

### @junior-dev - 2024-01-10 11:00:00

Thanks for the detailed answer! One question: what about performance? Is there any overhead with the new `try` function?

### @senior-dev - 2024-01-10 11:30:00

Great question! According to the Go team's benchmarks, `try` has zero runtime overhead because it's inlined by the compiler. It's essentially syntactic sugar.

### @guru - 2024-01-10 14:00:00

I'd also add that you should be careful about using `try` in hot paths. In performance-critical code, stick with manual error handling to avoid any potential stack traces.

```

### 示例 4: 带 `-enable-reactions` 和 `-enable-user-links`

```markdown
# [Issue #500] Add dark mode support

**Metadata:**
- **Author:** [(@ui-designer)](https://github.com/ui-designer)
- **Created:** 2024-01-20 10:00:00
- **Status:** Open
- **Labels:** enhancement, design

---

## Body

It would be great to have dark mode support for the web UI.

---

## Comments

### [(@frontend-dev)](https://github.com/frontend-dev) - 2024-01-20 11:00:00
Reactions: 👍 (12), ❤️ (5), 🎉 (3)

Great idea! I can work on this. Should we use CSS variables or a separate theme file?

### [(@ui-designer)](https://github.com/ui-designer) - 2024-01-20 11:30:00
Reactions: 👍 (8)

CSS variables would be better for maintainability. We can use `prefers-color-scheme` for automatic switching.

### [(@accessibility-expert)](https://github.com/accessibility-expert) - 2024-01-20 14:00:00
Reactions: 👀 (2)

Please ensure the dark mode meets WCAG AAA contrast requirements. I can help review the colors.
```

---

## 📝 附录

### A. GitHub API 端点

| 资源类型 | API 端点 |
|----------|----------|
| Issue | `GET /repos/{owner}/{repo}/issues/{issue_number}` |
| Issue Comments | `GET /repos/{owner}/{repo}/issues/{issue_number}/comments` |
| Pull Request | `GET /repos/{owner}/{repo}/pulls/{pull_number}` |
| PR Commits | `GET /repos/{owner}/{repo}/pulls/{pull_number}/commits` |
| PR Files | `GET /repos/{owner}/{repo}/pulls/{pull_number}/files` |
| PR Review Comments | `GET /repos/{owner}/{repo}/pulls/{pull_number}/comments` |
| Discussion | `GET /repos/{owner}/{repo}/discussions/{discussion_number}` |
| Discussion Comments | `GET /repos/{owner}/{repo}/discussions/{discussion_number}/comments` |

### B. 退出码定义

| 退出码 | 含义 |
|--------|------|
| 0 | 成功 |
| 1 | 一般错误（无效参数、网络错误、API 错误等） |
| 2 | 用户中断（Ctrl+C） |

### C. 环境变量完整列表

| 变量名 | 类型 | 必需 | 默认值 | 说明 |
|--------|------|------|--------|------|
| `GITHUB_TOKEN` | string | 否 | - | GitHub Personal Access Token |
| `HTTP_PROXY` | string | 否 | - | HTTP 代理地址 |
| `HTTPS_PROXY` | string | 否 | - | HTTPS 代理地址 |
| `NO_PROXY` | string | 否 | - | 不使用代理的域名列表 |
