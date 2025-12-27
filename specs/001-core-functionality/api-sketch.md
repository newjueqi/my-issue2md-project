# API Sketch - 核心包接口设计

本文档描述核心包的主要接口签名，作为后续实现的参考。

## 设计原则

遵循项目宪法：
- ✅ **简单性**: 优先使用标准库，避免过度抽象
- ✅ **明确性**: 所有错误显式处理，使用 `fmt.Errorf("context: %w", err)`
- ✅ **无全局变量**: 所有依赖通过函数参数或结构体成员显式注入

---

## 1. `internal/parser` - URL 解析器

**职责**: 解析 GitHub URL，识别资源类型和参数

### 1.1 核心数据结构

```go
// ResourceType 表示 GitHub 资源类型
type ResourceType int

const (
    ResourceTypeUnknown ResourceType = iota
    ResourceTypeIssue
    ResourceTypePullRequest
    ResourceTypeDiscussion
)

// ParsedURL 表示解析后的 URL
type ParsedURL struct {
    Type     ResourceType
    Owner    string
    Repo     string
    Number   int
    Original string // 原始 URL
}
```

### 1.2 主要接口

```go
// ParseURL 解析 GitHub URL 并返回资源信息
func ParseURL(url string) (*ParsedURL, error)

// String 返回资源的字符串表示（用于日志）
func (p *ParsedURL) String() string

// IsPrivate 判断资源是否可能属于私有仓库
func (p *ParsedURL) IsPrivate() bool
```

### 1.3 错误定义

```go
var (
    ErrInvalidURLFormat   = errors.New("invalid GitHub URL format")
    ErrUnsupportedType    = errors.New("unsupported resource type")
    ErrMissingNumber      = errors.New("missing resource number")
)
```

---

## 2. `internal/github` - GitHub API 客户端

**职责**: 与 GitHub API 交互，获取 Issue/PR/Discussion 数据

### 2.1 核心数据结构

```go
// Client 表示 GitHub API 客户端
type Client struct {
    token      string // Personal Access Token
    httpClient *http.Client
    baseURL    string // 默认: https://api.github.com
}

// Comment 表示统一的评论结构
type Comment struct {
    ID        int64
    Author    string
    Body      string
    CreatedAt time.Time
    UpdatedAt time.Time
    Reactions map[string]int // emoji -> count (可选)
}

// Issue 表示一个 Issue
type Issue struct {
    Number    int
    Title     string
    Author    string
    Body      string
    State     string // "open" | "closed"
    Labels    []string
    CreatedAt time.Time
    UpdatedAt time.Time
    Comments  []Comment
}

// PullRequest 表示一个 Pull Request
type PullRequest struct {
    Number        int
    Title         string
    Author        string
    Body          string
    State         string // "open" | "closed" | "merged"
    HeadBranch    string
    BaseBranch    string
    Labels        []string
    CreatedAt     time.Time
    UpdatedAt     time.Time
    MergedAt      *time.Time
    Comments      []Comment // 包含 Review Comments
    RelatedIssues []int     // 关联的 Issue 编号
    Diff          string    // 代码差异（可选）
}

// Discussion 表示一个 Discussion
type Discussion struct {
    Number    int
    Title     string
    Author    string
    Body      string
    State     string // "open" | "closed"
    CreatedAt time.Time
    UpdatedAt time.Time
    Comments  []DiscussionComment
}

// DiscussionComment 表示 Discussion 的评论
type DiscussionComment struct {
    ID        int64
    Author    string
    Body      string
    CreatedAt time.Time
    UpdatedAt time.Time
    IsAnswer  bool // 是否被标记为答案
    Replies   []Comment
}
```

### 2.2 主要接口

```go
// NewClient 创建一个新的 GitHub API 客户端
// token 可以为空（访问公开资源），但推荐设置以提高限额
func NewClient(token string) *Client

// FetchIssue 获取 Issue 数据
func (c *Client) FetchIssue(ctx context.Context, owner, repo string, number int) (*Issue, error)

// FetchPullRequest 获取 PR 数据
// includeDiff 控制是否获取代码差异
func (c *Client) FetchPullRequest(ctx context.Context, owner, repo string, number int, includeDiff bool) (*PullRequest, error)

// FetchDiscussion 获取 Discussion 数据
func (c *Client) FetchDiscussion(ctx context.Context, owner, repo string, number int) (*Discussion, error)

// FetchComments 获取评论（独立方法，便于测试）
func (c *Client) FetchComments(ctx context.Context, owner, repo string, number int, resourceType ResourceType) ([]Comment, error)
```

### 2.3 错误定义

```go
var (
    ErrResourceNotFound    = errors.New("resource not found")
    ErrAuthenticationFailed = errors.New("authentication failed")
    ErrRateLimitExceeded    = errors.New("rate limit exceeded")
    ErrNetworkError         = errors.New("network error")
)

// IsAuthError 判断是否为认证错误
func IsAuthError(err error) bool

// IsRateLimitError 判断是否为限流错误
func IsRateLimitError(err error) bool
```

---

## 3. `internal/converter` - Markdown 转换器

**职责**: 将 GitHub 数据转换为 Markdown 格式

### 3.1 核心数据结构

```go
// Converter 配置
type Config struct {
    EnableReactions  bool  // 是否包含 reactions
    EnableUserLinks  bool  // 是否转换用户链接
    LocalTimezone    string // 本地时区，如 "Asia/Shanghai"
}

// Converter 表示 Markdown 转换器
type Converter struct {
    config  Config
    timezone *time.Location
}
```

### 3.2 主要接口

```go
// NewConverter 创建一个新的转换器
func NewConverter(cfg Config) (*Converter, error)

// ConvertIssue 将 Issue 转换为 Markdown
func (c *Converter) ConvertIssue(issue *github.Issue) (string, error)

// ConvertPullRequest 将 PR 转换为 Markdown
func (c *Converter) ConvertPullRequest(pr *github.PullRequest) (string, error)

// ConvertDiscussion 将 Discussion 转换为 Markdown
func (c *Converter) ConvertDiscussion(discussion *github.Discussion) (string, error)

// ConvertComment 将评论转换为 Markdown 格式
func (c *Converter) ConvertComment(comment github.Comment) string

// FormatTime 格式化时间为本地时间
func (c *Converter) FormatTime(t time.Time) string

// GenerateFilename 根据标题生成文件名
func (c *Converter) GenerateFilename(title string, resourceType github.ResourceType, number int) string
```

### 3.3 辅助方法

```go
// escapeMarkdown 转义 Markdown 特殊字符
func escapeMarkdown(text string) string

// formatReactions 格式化 reactions 为字符串
func (c *Converter) formatReactions(reactions map[string]int) string

// formatUserLink 格式化用户链接或纯文本
func (c *Converter) formatUserLink(username string) string

// sanitizeFilename 清理文件名中的非法字符
func sanitizeFilename(name string) string
```

---

## 4. `internal/config` - 配置管理

**职责**: 管理环境变量和命令行参数

### 4.1 核心数据结构

```go
// Flags 表示命令行参数
type Flags struct {
    EnableReactions bool
    EnableUserLinks bool
}

// Config 表示完整的配置
type Config struct {
    Flags        Flags
    GitHubToken  string
    OutputFile   string // 空字符串表示 stdout
    URL          string
    ParsedURL    *parser.ParsedURL
}
```

### 4.2 主要接口

```go
// LoadConfig 从环境变量和命令行参数加载配置
func LoadConfig(args []string) (*Config, error)

// Validate 验证配置的有效性
func (c *Config) Validate() error

// GetGitHubToken 从环境变量获取 GitHub Token
func GetGitHubToken() string
```

### 4.3 错误定义

```go
var (
    ErrMissingURL      = errors.New("URL is required")
    ErrInvalidFlag     = errors.New("invalid flag")
    ErrTooManyArgs     = errors.New("too many arguments")
)
```

---

## 5. `internal/cli` - 命令行接口

**职责**: 处理命令行输入，协调各模块完成转换

### 5.1 核心数据结构

```go
// Runner 表示 CLI 运行器
type Runner struct {
    config     *config.Config
    githubClient *github.Client
    converter  *converter.Converter
}

// Result 表示执行结果
type Result struct {
    Success   bool
    Output    string
    Error     error
    ExitCode  int
}
```

### 5.2 主要接口

```go
// NewRunner 创建一个新的 Runner
func NewRunner(cfg *config.Config) (*Runner, error)

// Run 执行转换流程
func (r *Runner) Run(ctx context.Context) *Result

// writeOutput 将结果写入文件或 stdout
func (r *Runner) writeOutput(markdown string) error

// handleError 处理错误并返回用户友好的错误信息
func (r *Runner) handleError(err error) string
```

### 5.3 执行流程

```go
func (r *Runner) Run(ctx context.Context) *Result {
    // 1. 解析 URL
    parsed, err := parser.ParseURL(r.config.URL)
    if err != nil {
        return &Result{Success: false, Error: err, ExitCode: 1}
    }

    // 2. 根据 Type 获取数据
    var data interface{}
    switch parsed.Type {
    case parser.ResourceTypeIssue:
        data, err = r.githubClient.FetchIssue(ctx, parsed.Owner, parsed.Repo, parsed.Number)
    case parser.ResourceTypePullRequest:
        data, err = r.githubClient.FetchPullRequest(ctx, parsed.Owner, parsed.Repo, parsed.Number, true)
    case parser.ResourceTypeDiscussion:
        data, err = r.githubClient.FetchDiscussion(ctx, parsed.Owner, parsed.Repo, parsed.Number)
    default:
        return &Result{Success: false, Error: parser.ErrUnsupportedType, ExitCode: 1}
    }

    if err != nil {
        return r.handleError(err)
    }

    // 3. 转换为 Markdown
    var markdown string
    switch v := data.(type) {
    case *github.Issue:
        markdown, err = r.converter.ConvertIssue(v)
    case *github.PullRequest:
        markdown, err = r.converter.ConvertPullRequest(v)
    case *github.Discussion:
        markdown, err = r.converter.ConvertDiscussion(v)
    }

    if err != nil {
        return &Result{Success: false, Error: err, ExitCode: 1}
    }

    // 4. 输出
    if err := r.writeOutput(markdown); err != nil {
        return &Result{Success: false, Error: err, ExitCode: 1}
    }

    return &Result{Success: true, ExitCode: 0}
}
```

---

## 6. 包依赖关系

```
cmd/issue2md/
    └── internal/cli
        ├── internal/config
        ├── internal/parser
        ├── internal/github
        └── internal/converter
            └── internal/github
```

**依赖规则**:
- `internal/github` 是最底层的包，不依赖任何内部包
- `internal/converter` 依赖 `internal/github`（使用其数据结构）
- `internal/parser` 是独立的包，不依赖任何内部包
- `internal/cli` 协调所有包，处于顶层
- `internal/config` 只被 `cli` 使用

---

## 7. 测试策略

### 7.1 单元测试

```go
// internal/parser/url_test.go
func TestParseURL(t *testing.T) {
    tests := []struct {
        name    string
        url     string
        want    *ParsedURL
        wantErr error
    }{
        // 表格驱动测试...
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := ParseURL(tt.url)
            if err != tt.wantErr {
                t.Errorf("ParseURL() error = %v, wantErr %v", err, tt.wantErr)
            }
            // ...
        })
    }
}
```

### 7.2 集成测试

```go
// internal/github/client_integration_test.go
func TestClient_FetchIssue_Integration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }

    client := NewClient(os.Getenv("GITHUB_TOKEN"))
    // 使用真实的 GitHub API 进行测试
}
```

### 7.3 Fake 实现（用于单元测试）

```go
// internal/github/fake_client_test.go
type FakeClient struct {
    Issues map[string]*Issue
}

func (f *FakeClient) FetchIssue(ctx context.Context, owner, repo string, number int) (*Issue, error) {
    key := fmt.Sprintf("%s/%s#%d", owner, repo, number)
    issue, ok := f.Issues[key]
    if !ok {
        return nil, ErrResourceNotFound
    }
    return issue, nil
}
```

---

## 8. 未来扩展考虑

### 8.1 Web 版本支持

```
cmd/issue2mdweb/
    └── internal/handler
        ├── internal/converter
        ├── internal/github
        └── internal/parser
```

**注意**: Web 版本直接复用 `internal/converter` 和 `internal/github`，无需重复实现。

### 8.2 其他 Git 平台支持

```
internal/
    ├── github/
    ├── gitlab/     # 未来实现
    └── gitea/      # 未来实现

// 定义统一的接口
type ResourceFetcher interface {
    FetchIssue(ctx, owner, repo, number) (interface{}, error)
    FetchPullRequest(ctx, owner, repo, number) (interface{}, error)
}
```

**注意**: 当前版本专注于 GitHub，不过度设计抽象层。

---

## 9. 错误处理示例

所有错误必须使用 `fmt.Errorf` 包装上下文：

```go
func (c *Client) FetchIssue(ctx context.Context, owner, repo string, number int) (*Issue, error) {
    url := fmt.Sprintf("%s/repos/%s/%s/issues/%d", c.baseURL, owner, repo, number)

    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, fmt.Errorf("failed to create request: %w", err)
    }

    // 添加认证头
    if c.token != "" {
        req.Header.Set("Authorization", fmt.Sprintf("Bearer %s", c.token))
    }

    resp, err := c.httpClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("failed to fetch issue: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode == http.StatusNotFound {
        return nil, fmt.Errorf("issue %d not found in %s/%s: %w", number, owner, repo, ErrResourceNotFound)
    }

    // 解析响应...
}
```

---

## 10. 性能考虑

### 10.1 并发获取评论

对于大型 Discussion，可以考虑并发获取评论：

```go
func (c *Client) FetchDiscussionComments(ctx context.Context, owner, repo string, number int) ([]Comment, error) {
    // 使用 errgroup 进行并发控制
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(5) // 最多 5 个并发请求

    var comments []Comment
    // ...
    return comments, g.Wait()
}
```

**注意**: MVP 版本可以简化为串行获取，根据实际需求再优化。

### 10.2 HTTP 客户端配置

```go
func NewClient(token string) *Client {
    return &Client{
        token: token,
        httpClient: &http.Client{
            Timeout: 30 * time.Second,
            Transport: &http.Transport{
                MaxIdleConns:        10,
                IdleConnTimeout:     30 * time.Second,
                DisableCompression:  true,
            },
        },
    }
}
```

---

*最后更新: 2024-01-15*
