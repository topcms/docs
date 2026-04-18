# MinDoc 迁移指南

> 本文档描述从 Beego v2 + ORM 迁移到 go-kratos + GORM 的具体操作方法，  
> 供开发人员在各模块重构时参考。

---

## 一、依赖替换速查表

### 后端依赖变更

| 原依赖 | 新依赖 | 说明 |
|--------|--------|------|
| `github.com/beego/beego/v2/server/web` | `github.com/go-kratos/kratos/v2` | Web 框架 |
| `github.com/beego/beego/v2/client/orm` | `gorm.io/gorm` | ORM |
| `github.com/beego/beego/v2/core/logs` | `github.com/go-kratos/kratos/v2/log` + `go.uber.org/zap` | 日志 |
| Beego Session | `github.com/golang-jwt/jwt/v5` | 认证 |
| `github.com/beego/beego/v2/server/web/context` | `context.Context`（标准库） | 上下文 |
| `github.com/beego/i18n` | `github.com/nicksnyder/go-i18n/v2` | 国际化 |
| `gopkg.in/ini.v1`（app.conf） | `github.com/go-kratos/kratos/v2/config` | 配置 |

### 保留依赖（无需变更）

```
github.com/mattn/go-sqlite3          // SQLite 驱动，保留
github.com/go-sql-driver/mysql       // MySQL 驱动，保留
gopkg.in/ldap.v2                     // LDAP，保留
github.com/lifei6671/gocaptcha       // 验证码，保留
github.com/boombuler/barcode         // 二维码，保留
github.com/russross/blackfriday/v2   // Markdown 渲染，保留（后端场景）
github.com/PuerkitoBio/goquery       // HTML 解析，保留
```

---

## 二、ORM 迁移（Beego ORM → GORM）

### 2.1 Model 标签替换

```go
// ===== 原来（Beego ORM）=====
type Member struct {
    MemberId      int       `orm:"pk;auto;column(member_id)"`
    Account       string    `orm:"size(100);column(account)"`
    Password      string    `orm:"size(1000);column(password)"`
    CreateAt      int       `orm:"column(create_at)"`
    Email         string    `orm:"size(100);column(email)"`
    LastLoginTime time.Time `orm:"column(last_login_time);null;type(datetime)"`
}

// ===== 新的（GORM）=====
type Member struct {
    MemberId      int       `gorm:"primaryKey;autoIncrement;column:member_id"`
    Account       string    `gorm:"size:100;column:account"`
    Password      string    `gorm:"size:1000;column:password"`
    CreateAt      int       `gorm:"column:create_at"`
    Email         string    `gorm:"size:100;column:email"`
    LastLoginTime time.Time `gorm:"column:last_login_time"`
}

// 表名方法（替代 Beego ORM 的 TableName）
func (Member) TableName() string { return "md_members" }
```

### 2.2 常用查询迁移

```go
// ===== 查询单条记录 =====
// 原来
o := orm.NewOrm()
member := &Member{MemberId: id}
err := o.Read(member)

// 新的
var member Member
err := db.First(&member, id).Error

// ===== 条件查询 =====
// 原来
o.QueryTable("md_members").Filter("account", account).One(&member)

// 新的
db.Where("account = ?", account).First(&member)

// ===== 插入 =====
// 原来
o.Insert(&member)

// 新的
db.Create(&member)

// ===== 更新指定字段 =====
// 原来（原项目大量使用这种方式）
member.Update("last_login_time")   // 原 models/Member.go 中的方法

// 新的
db.Model(&member).Update("last_login_time", member.LastLoginTime)
// 或更新多个字段
db.Model(&member).Updates(map[string]interface{}{
    "last_login_time": member.LastLoginTime,
    "password":        member.Password,
})

// ===== 分页查询 =====
// 原来
o.QueryTable("md_books").Limit(pageSize).Offset(offset).All(&books)

// 新的
db.Limit(pageSize).Offset(offset).Find(&books)

// ===== 关联查询（多表）=====
// 原来
o.QueryTable("md_relationship").
    Filter("book_id", bookId).
    Filter("role_id__gte", 0).
    RelatedSel("Member").
    All(&relationships)

// 新的
db.Preload("Member").
    Where("book_id = ? AND role_id >= 0", bookId).
    Find(&relationships)
```

### 2.3 数据库初始化

```go
// backend/app/mindoc/internal/data/data.go

func NewDB(conf *conf.Data) (*gorm.DB, func(), error) {
    var db *gorm.DB
    var err error

    switch conf.Database.Driver {
    case "mysql":
        db, err = gorm.Open(mysql.Open(conf.Database.Source), &gorm.Config{})
    case "sqlite3":
        db, err = gorm.Open(sqlite.Open(conf.Database.Source), &gorm.Config{})
    }
    if err != nil {
        return nil, nil, err
    }

    // 自动迁移（替代原 models/Migrations.go）
    db.AutoMigrate(
        &Member{}, &Book{}, &Document{},
        &DocumentHistory{}, &Attachment{},
        &Blog{}, &Comment{}, &Team{},
        &TeamMember{}, &Relationship{},
        &Label{}, &Itemsets{}, &Options{},
        &Template{}, &MemberToken{}, &Logs{},
    )

    cleanup := func() {
        sqlDB, _ := db.DB()
        sqlDB.Close()
    }
    return db, cleanup, nil
}
```

---

## 三、Controller → Service 迁移

### 3.1 基本结构对比

```go
// ===== 原来（Beego Controller）=====
type AccountController struct {
    BaseController
}

func (c *AccountController) Login() {
    c.Prepare()
    if c.Ctx.Input.IsPost() {
        account := c.GetString("account")
        password := c.GetString("password")
        member, err := models.NewMember().Login(account, password)
        if err != nil {
            c.JsonResult(500, "用户名或密码错误")
        }
        c.SetMember(*member)
        c.JsonResult(0, "ok", c.referer())
    }
}

// ===== 新的（kratos Service）=====
type AccountService struct {
    pb.UnimplementedAccountServiceServer
    uc  *biz.AccountUsecase
    log *log.Helper
}

func (s *AccountService) Login(ctx context.Context, req *pb.LoginRequest) (*pb.LoginReply, error) {
    member, err := s.uc.Login(ctx, req.Account, req.Password)
    if err != nil {
        return nil, v1.ErrorUserNotFound("用户名或密码错误")
    }
    token, err := s.uc.GenerateToken(ctx, member)
    if err != nil {
        return nil, err
    }
    return &pb.LoginReply{
        AccessToken:  token.AccessToken,
        RefreshToken: token.RefreshToken,
        ExpiresIn:    token.ExpiresIn,
    }, nil
}
```

### 3.2 JsonResult 错误码迁移

原项目 `JsonResult(errCode, errMsg)` 中的错误码迁移为 kratos errors：

```go
// backend/api/common/v1/errors.proto
enum ErrorReason {
    // 账号相关：6000-6099
    CAPTCHA_WRONG          = 6001;  // 验证码错误
    ACCOUNT_OR_PWD_EMPTY   = 6002;  // 账号或密码为空
    WRONG_ACCOUNT_PASSWORD = 6003;  // 账号或密码错误
    EMAIL_INVALID_FORMAT   = 6004;  // 邮箱格式不正确
    ACCOUNT_EXISTED        = 6005;  // 账号已存在
    FAILED_REGISTER        = 6006;  // 注册失败

    // 权限相关：403
    ILLEGAL_REQUEST        = 4030;  // 非法请求
    PERMISSION_DENIED      = 4031;  // 无权限
}
```

### 3.3 BaseController.Prepare() 的迁移

`Prepare()` 中的公共逻辑迁移到 middleware 和 biz 层：

```go
// ===== 原来的 Prepare() 职责 =====
// 1. 从 Session 读用户信息         → JWT middleware 解析后注入 context
// 2. 从 Cookie 读用户信息          → JWT refresh_token 机制
// 3. 读取站点 Options              → biz 层提供 GetSiteOptions() 方法
// 4. 注入模板数据（SiteName 等）    → 前端 app store 维护
// 5. 设置语言                      → 前端 vue-i18n 处理

// ===== middleware/auth.go 替代 1、2 =====
func JWTMiddleware(secret string) middleware.Middleware {
    return func(handler middleware.Handler) middleware.Handler {
        return func(ctx context.Context, req interface{}) (interface{}, error) {
            // 从 Authorization header 解析 JWT
            // 注入 member 信息到 ctx
            return handler(ctx, req)
        }
    }
}
```

---

## 四、路由迁移（router.go → server/http.go）

### 4.1 路由注册对比

```go
// ===== 原来（Beego）=====
web.Router("/login", &controllers.AccountController{}, "*:Login")
web.Router("/docs/:key", &controllers.DocumentController{}, "*:Index")
web.Router("/api/:key/content/?:id", &controllers.DocumentController{}, "*:Content")

// ===== 新的（kratos HTTP）=====
// 通过 proto 注解自动生成，在 api/account/v1/account.proto 中：
// option (google.api.http) = {
//   post: "/api/v1/account/login"
//   body: "*"
// };

// server/http.go 中注册 service
func NewHTTPServer(c *conf.Server, ac *service.AccountService, dc *service.DocumentService, ...) *http.Server {
    srv := http.NewServer(
        http.Address(c.Http.Addr),
        http.Middleware(
            recovery.Recovery(),
            logging.Server(logger),
            middleware.JWTMiddleware(c.Auth.JwtSecret),
        ),
    )
    pb_account.RegisterAccountServiceHTTPServer(srv, ac)
    pb_document.RegisterDocumentServiceHTTPServer(srv, dc)
    return srv
}
```

### 4.2 路由对照表（原路由 → 新 API）

| 原路由 | HTTP 方法 | 新 API | Service 方法 |
|--------|-----------|--------|-------------|
| `/login` | POST | `/api/v1/account/login` | AccountService.Login |
| `/register` | POST | `/api/v1/account/register` | AccountService.Register |
| `/logout` | GET/POST | `/api/v1/account/logout` | AccountService.Logout |
| `/find_password` | POST | `/api/v1/account/find-password` | AccountService.FindPassword |
| `/valid_email` | POST | `/api/v1/account/reset-password` | AccountService.ResetPassword |
| `/captcha` | GET | `/api/v1/account/captcha` | AccountService.GetCaptcha |
| `/dingtalk_login` | POST | `/api/v1/account/dingtalk` | AccountService.DingTalkLogin |
| `/book` | GET | `/api/v1/books` | BookService.ListBooks |
| `/book/create` | POST | `/api/v1/books` | BookService.CreateBook |
| `/book/:key/setting` | GET/POST | `/api/v1/books/:key` | BookService.GetBook / UpdateBook |
| `/book/:key/release` | POST | `/api/v1/books/:key/release` | BookService.Release |
| `/docs/:key` | GET | `/api/v1/docs/:key` | DocumentService.GetBookIndex |
| `/docs/:key/:id` | GET | `/api/v1/docs/:key/:id` | DocumentService.ReadDocument |
| `/api/:key/create` | POST | `/api/v1/docs/:key/documents` | DocumentService.CreateDocument |
| `/api/:key/content/:id` | GET/POST | `/api/v1/docs/:key/documents/:id` | DocumentService.GetContent |
| `/api/:key/delete` | POST | `/api/v1/docs/:key/documents/:id` | DocumentService.DeleteDocument |
| `/history/get` | GET | `/api/v1/docs/:key/documents/:id/history` | DocumentService.GetHistory |
| `/history/restore` | POST | `/api/v1/docs/:key/documents/:id/restore` | DocumentService.RestoreHistory |
| `/api/upload` | POST | `/api/v1/upload` | DocumentService.Upload |
| `/export/:key` | GET | `/api/v1/books/:key/export` | DocumentService.Export |
| `/search` | GET | `/api/v1/search` | SearchService.Search |
| `/manager/users` | GET | `/api/v1/manager/members` | ManagerService.ListMembers |
| `/manager/member/create` | POST | `/api/v1/manager/members` | ManagerService.CreateMember |
| `/manager/setting` | GET/POST | `/api/v1/manager/settings` | ManagerService.GetSettings / UpdateSettings |
| `/blogs` | GET | `/api/v1/blogs` | BlogService.ListBlogs |
| `/blog-:id.html` | GET | `/api/v1/blogs/:id` | BlogService.GetBlog |
| `/comment/create` | POST | `/api/v1/comments` | CommentService.CreateComment |
| `/tag/:key` | GET | `/api/v1/labels/:key` | LabelService.GetLabel |
| `/items` | GET | `/api/v1/itemsets` | ItemsetsService.ListItemsets |
| `/setting` | GET/POST | `/api/v1/settings/profile` | SettingService.GetProfile / UpdateProfile |
| `/setting/password` | POST | `/api/v1/settings/password` | SettingService.ChangePassword |

---

## 五、认证体系迁移（Session → JWT）

### 5.1 登录流程对比

```
===== 原流程 =====
POST /login
  → AccountController.Login()
  → models.Member.Login(account, password)
  → c.SetSession(conf.LoginSessionName, member)  // 写 Session
  → 返回 {errcode: 0, data: redirect_url}
  → 前端跳转页面

===== 新流程 =====
POST /api/v1/account/login
  → AccountService.Login()
  → biz.AccountUsecase.Login(account, password)
  → 生成 JWT（access_token 2h + refresh_token 30d）
  → 返回 {code: 0, data: {access_token, refresh_token, expires_in, member}}
  → 前端存储 token（localStorage）
  → 后续请求 Header: Authorization: Bearer <access_token>
```

### 5.2 "记住我"功能迁移

```
原来：SetSecureCookie("login", encoded_member_info, 30天)
新的：refresh_token 有效期设为 30 天
     前端定时调用 /api/v1/account/refresh-token 换取新 access_token
```

### 5.3 middleware 白名单（替代 filter.go）

```go
// 不需要 JWT 的接口（对应原 filter.go 中不拦截的路径）
var authWhiteList = []string{
    "/api/v1/account/login",
    "/api/v1/account/register",
    "/api/v1/account/captcha",
    "/api/v1/account/find-password",
    "/api/v1/account/dingtalk",
    "/api/v1/docs/*/",         // 匿名访问（需配合 ENABLE_ANONYMOUS 配置）
}
```

---

## 六、配置文件迁移（app.conf → config.yaml）

```yaml
# backend/configs/config.yaml

server:
  http:
    addr: 0.0.0.0:8080
    timeout: 1s
  grpc:
    addr: 0.0.0.0:9000
    timeout: 1s

auth:
  jwt_secret: "your-secret-key"       # 原 app_key
  access_token_expire: 7200           # 2小时（秒）
  refresh_token_expire: 2592000       # 30天（秒）

data:
  database:
    driver: mysql                     # 原 db_adapter
    source: "root:123456@tcp(localhost:3306)/mindoc?charset=utf8mb4&parseTime=True"
  redis:
    addr: 127.0.0.1:6379
    password: ""
    db: 0
    read_timeout: 0.2s
    write_timeout: 0.2s

app:
  base_url: "http://localhost:8080"   # 原 baseurl
  upload_path: "./uploads"            # 原 upload_path
  upload_file_size: 10485760          # 10MB，原 upload_file_size
  enable_register: true               # 原 ENABLE_REGISTER（Options 表）
  enable_anonymous: false             # 原 ENABLE_ANONYMOUS（Options 表）
  default_lang: "zh-CN"              # 原 default_lang

mail:
  enabled: false                      # 原 EnableMail
  smtp_host: ""                       # 原 smtp_host
  smtp_port: 465
  smtp_username: ""
  smtp_password: ""
  from_username: ""
  secure: true
  mail_number: 3                      # 每小时最大发送次数

dingtalk:
  corp_id: ""                         # 原 dingtalk_corpid
  app_key: ""                         # 原 dingtalk_app_key
  app_secret: ""                      # 原 dingtalk_app_secret
  tmp_reader: ""                      # 原 dingtalk_tmp_reader
  qr_key: ""                          # 原 dingtalk_qr_key
  qr_secret: ""                       # 原 dingtalk_qr_secret

ldap:
  enabled: false
  host: ""
  port: 389
  base_dn: ""
  bind_dn: ""
  bind_password: ""
  user_filter: ""

highlight:
  style: "github"                     # 原 highlight_style
```

---

## 七、前端迁移要点

### 7.1 模板变量 → API 数据

| 原模板变量（Beego Data）| 新获取方式（Vue 3）|
|------------------------|-------------------|
| `{{.Member}}` | `userStore.member`（Pinia） |
| `{{.SiteName}}` | `appStore.siteName`（从 API 获取） |
| `{{.BaseUrl}}` | `import.meta.env.VITE_API_BASE_URL` |
| `{{.Model}}`（书籍信息）| `bookStore.current` |
| `{{.Result}}`（文档树 HTML）| `<el-tree :data="docTreeData">` |
| `{{.Content}}`（文档 HTML）| `<div v-html="document.content">` |
| `{{.Lang}}` | `i18n.locale` |
| `{{.xsrfdata}}` | 无需（JWT 无 CSRF 问题） |

### 7.2 验证码处理

```typescript
// 原来：<img src="/captcha"> 直接返回图片
// 新的：API 返回 Base64 图片数据

// GET /api/v1/account/captcha
// Response: { "captcha_id": "xxx", "image": "data:image/jpeg;base64,..." }

const { data } = await accountApi.getCaptcha()
captchaId.value = data.captcha_id
captchaImage.value = data.image  // 绑定到 <img :src="captchaImage">

// 登录时携带 captcha_id
await accountApi.login({
  account, password,
  captcha_id: captchaId.value,
  captcha_code: captchaInput.value
})
```

### 7.3 文件上传

```typescript
// 原来：POST /api/upload（multipart/form-data）
// 新的：接口地址变更，逻辑保持不变

// frontend/src/composables/useUpload.ts
import { isTauri } from '@/utils/platform'
import { open } from '@tauri-apps/plugin-dialog'   // 桌面端文件选择

export function useUpload() {
  async function selectFile() {
    if (isTauri) {
      // 桌面端：使用系统文件选择框
      return await open({ multiple: false })
    } else {
      // Web 端：触发 input[type=file]
      return await openFileDialog()
    }
  }
  return { selectFile }
}
```

---

## 八、数据库表名注意事项

原项目所有表均有 `md_` 前缀，GORM 迁移时需通过 `TableName()` 方法明确指定，
避免 GORM 自动推导表名（如 `Member` → `members`）与实际表名不符。

```go
// 所有 data 层 model 都需要实现 TableName()
func (*Member)       TableName() string { return "md_members" }
func (*Book)         TableName() string { return "md_books" }
func (*Document)     TableName() string { return "md_documents" }
func (*Attachment)   TableName() string { return "md_attachments" }
func (*Blog)         TableName() string { return "md_blogs" }
func (*Comment)      TableName() string { return "md_comments" }
func (*Team)         TableName() string { return "md_teams" }
func (*TeamMember)   TableName() string { return "md_team_members" }
func (*Relationship) TableName() string { return "md_relationships" }
func (*Label)        TableName() string { return "md_labels" }
func (*Itemsets)     TableName() string { return "md_itemsets" }
func (*MemberToken)  TableName() string { return "md_member_tokens" }
func (*Options)      TableName() string { return "md_options" }
func (*Template)     TableName() string { return "md_templates" }
func (*Logs)         TableName() string { return "md_logs" }
```
