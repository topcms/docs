# 用户服务设计文档（User Service）

> 用户服务是从主业务服务中抽离的第一个独立微服务，  
> 负责所有与"人"相关的能力：认证、授权、账号管理、第三方登录。

---

## 一、服务职责划分

### 用户服务负责

| 功能 | 来源（原代码） |
|------|--------------|
| 登录（账号密码 / LDAP） | `AccountController.Login` + `models/Member.go` |
| 注册 | `AccountController.Register` |
| 退出 | `AccountController.Logout` |
| 找回/重置密码（邮件） | `AccountController.FindPassword` + `AccountController.ValidEmail` |
| 验证码生成 | `AccountController.Captcha` |
| 钉钉登录 / 钉钉扫码登录 | `AccountController.DingTalkLogin` / `QRLogin` |
| JWT 签发 / 刷新 / 吊销 | 新增（替换 Session） |
| 个人资料查询/修改 | `SettingController.Index` / `SettingController.Upload`（头像） |
| 修改密码 | `SettingController.Password` |
| 用户管理（管理员） | `ManagerController.Users` / `EditMember` / `CreateMember` 等 |
| 用户信息查询（内部 gRPC） | 新增（供 mindoc-service 调用） |

### 用户服务**不**负责

- 书籍/文档/博客/评论等业务权限判断（由 mindoc-service 基于 JWT 中的 user_id 和 role 自行判断）
- 团队管理（Team 属于业务概念，由 mindoc-service 管理）
- 书籍成员关系（Relationship，由 mindoc-service 管理）

---

## 二、目录结构

```
backend/app/user/
├── cmd/user/
│   ├── main.go               # 入口：启动 HTTP + gRPC server
│   └── wire.go               # Wire DI provider 集合
│
└── internal/
    ├── conf/                  # 配置结构体
    │   └── conf.proto         # 生成 conf.pb.go
    │
    ├── server/
    │   ├── http.go            # HTTP server（:8080）注册对外接口
    │   ├── grpc.go            # gRPC server（:9000）注册内部接口
    │   └── server.go          # Wire provider
    │
    ├── middleware/
    │   ├── auth.go            # JWT 校验（对外 HTTP 接口的鉴权）
    │   └── whitelist.go       # 白名单（login/register/captcha 不鉴权）
    │
    ├── service/
    │   ├── user.go            # 对外 HTTP：实现 UserService proto
    │   └── user_internal.go   # 对内 gRPC：实现 UserInternalService proto
    │
    ├── biz/
    │   ├── user.go            # 用户聚合（Member）+ Usecase
    │   ├── token.go           # JWT 签发/刷新/吊销逻辑
    │   ├── captcha.go         # 验证码业务
    │   ├── ldap.go            # LDAP 认证
    │   ├── dingtalk.go        # 钉钉登录业务
    │   └── biz.go             # Wire provider
    │
    └── data/
        ├── data.go            # DB 初始化 + Wire provider
        ├── member.go          # md_members 表 CRUD
        ├── member_token.go    # md_member_tokens 表（密码重置 token）
        └── options.go         # md_options 表（站点配置，供用户服务读取）
```

---

## 三、Protobuf 接口定义

### 3.1 对外 HTTP 接口（api/user/v1/user.proto）

```protobuf
syntax = "proto3";
package user.v1;

import "google/api/annotations.proto";

option go_package = "github.com/mindoc-org/mindoc/api/user/v1;v1";

// ─── 对外 HTTP 服务 ───────────────────────────────────────────────
service UserService {

  // 登录（账号密码 / LDAP）
  rpc Login(LoginRequest) returns (LoginReply) {
    option (google.api.http) = { post: "/api/v1/user/login"; body: "*" };
  }

  // 注册
  rpc Register(RegisterRequest) returns (RegisterReply) {
    option (google.api.http) = { post: "/api/v1/user/register"; body: "*" };
  }

  // 退出（使 refresh_token 失效）
  rpc Logout(LogoutRequest) returns (LogoutReply) {
    option (google.api.http) = { post: "/api/v1/user/logout"; body: "*" };
  }

  // 刷新 access_token
  rpc RefreshToken(RefreshTokenRequest) returns (LoginReply) {
    option (google.api.http) = { post: "/api/v1/user/refresh-token"; body: "*" };
  }

  // 获取验证码（Base64 图片）
  rpc GetCaptcha(GetCaptchaRequest) returns (GetCaptchaReply) {
    option (google.api.http) = { get: "/api/v1/user/captcha" };
  }

  // 找回密码（发邮件）
  rpc FindPassword(FindPasswordRequest) returns (FindPasswordReply) {
    option (google.api.http) = { post: "/api/v1/user/find-password"; body: "*" };
  }

  // 重置密码（验证 token）
  rpc ResetPassword(ResetPasswordRequest) returns (ResetPasswordReply) {
    option (google.api.http) = { post: "/api/v1/user/reset-password"; body: "*" };
  }

  // 钉钉登录（App 内扫码）
  rpc DingTalkLogin(DingTalkLoginRequest) returns (LoginReply) {
    option (google.api.http) = { post: "/api/v1/user/dingtalk/login"; body: "*" };
  }

  // 钉钉扫码登录回调
  rpc DingTalkQRCallback(DingTalkQRRequest) returns (LoginReply) {
    option (google.api.http) = { post: "/api/v1/user/dingtalk/qr"; body: "*" };
  }

  // ── 以下需要 JWT 鉴权 ──

  // 获取当前用户信息
  rpc GetProfile(GetProfileRequest) returns (UserInfo) {
    option (google.api.http) = { get: "/api/v1/user/profile" };
  }

  // 更新个人资料（昵称/邮箱等）
  rpc UpdateProfile(UpdateProfileRequest) returns (UserInfo) {
    option (google.api.http) = { put: "/api/v1/user/profile"; body: "*" };
  }

  // 修改密码
  rpc ChangePassword(ChangePasswordRequest) returns (ChangePasswordReply) {
    option (google.api.http) = { put: "/api/v1/user/password"; body: "*" };
  }

  // 上传头像
  rpc UploadAvatar(UploadAvatarRequest) returns (UserInfo) {
    option (google.api.http) = { post: "/api/v1/user/avatar"; body: "*" };
  }

  // ── 管理员接口（需 admin role） ──

  rpc ListMembers(ListMembersRequest) returns (ListMembersReply) {
    option (google.api.http) = { get: "/api/v1/user/manager/members" };
  }
  rpc GetMember(GetMemberRequest) returns (UserInfo) {
    option (google.api.http) = { get: "/api/v1/user/manager/members/{member_id}" };
  }
  rpc CreateMember(CreateMemberRequest) returns (UserInfo) {
    option (google.api.http) = { post: "/api/v1/user/manager/members"; body: "*" };
  }
  rpc UpdateMember(UpdateMemberRequest) returns (UserInfo) {
    option (google.api.http) = { put: "/api/v1/user/manager/members/{member_id}"; body: "*" };
  }
  rpc DeleteMember(DeleteMemberRequest) returns (DeleteMemberReply) {
    option (google.api.http) = { delete: "/api/v1/user/manager/members/{member_id}" };
  }
  rpc UpdateMemberStatus(UpdateMemberStatusRequest) returns (UserInfo) {
    option (google.api.http) = { put: "/api/v1/user/manager/members/{member_id}/status"; body: "*" };
  }
  rpc ChangeMemberRole(ChangeMemberRoleRequest) returns (UserInfo) {
    option (google.api.http) = { put: "/api/v1/user/manager/members/{member_id}/role"; body: "*" };
  }
}

// ─── 内部 gRPC 服务（不注册 HTTP，仅供 mindoc-service 调用）────────
service UserInternalService {
  rpc GetUserInfo(GetUserInfoRequest) returns (UserInfo);
  rpc BatchGetUsers(BatchGetUsersRequest) returns (BatchGetUsersReply);
}

// ─── 消息定义 ─────────────────────────────────────────────────────
message LoginRequest {
  string account     = 1;
  string password    = 2;
  string captcha_id  = 3;   // 验证码 ID（开启时必填）
  string captcha_code = 4;  // 验证码
  bool   remember_me = 5;   // 记住我（延长 refresh_token 有效期）
}

message LoginReply {
  string access_token  = 1;
  string refresh_token = 2;
  int64  expires_in    = 3;  // access_token 有效期（秒）
  UserInfo member      = 4;
}

message UserInfo {
  int64  member_id   = 1;
  string account     = 2;
  string real_name   = 3;
  string email       = 4;
  string phone       = 5;
  string avatar      = 6;
  int32  role        = 7;   // 0=普通用户 1=管理员 2=超级管理员
  int32  status      = 8;   // 0=正常 1=禁用
  string auth_method = 9;   // local / ldap
  string description = 10;
}

message GetCaptchaReply {
  string captcha_id = 1;
  string image      = 2;    // data:image/jpeg;base64,...
}

message GetUserInfoRequest  { int64 member_id = 1; }
message BatchGetUsersRequest { repeated int64 ids = 1; }
message BatchGetUsersReply   { repeated UserInfo users = 1; }
```

---

## 四、核心业务逻辑设计

### 4.1 JWT Token 设计

```go
// backend/app/user/internal/biz/token.go

// access_token Claims（有效期 2 小时）
type Claims struct {
    MemberId int64  `json:"mid"`
    Account  string `json:"acc"`
    Role     int32  `json:"role"`
    jwt.RegisteredClaims
}

// refresh_token 存入 Redis
// Key:   "user:refresh:{token_hash}"
// Value: member_id
// TTL:   7天（普通）/ 30天（记住我）

// 吊销：Logout 时删除 Redis 中的 refresh_token
// 主动失效：管理员禁用用户时，可通过 user_id 批量删除其所有 refresh_token
```

### 4.2 验证码流程

```
GET /api/v1/user/captcha
  → 生成随机 4 位文本 + 图片
  → captcha_id 随机生成，文本存 Redis（TTL 5分钟）
  → 返回 {captcha_id, image: "data:image/jpeg;base64,..."}

POST /api/v1/user/login（含验证码）
  → 从 Redis 查 captcha_id 对应的文本
  → 比对通过后删除 Redis key（防重放）
```

### 4.3 找回密码流程

```
POST /api/v1/user/find-password（发邮件）
  → 检查邮件频率（同一邮箱 1 小时内最多 N 次）→ md_member_tokens 表
  → 生成 token（32位随机），写入 md_member_tokens
  → 异步发邮件（pkg/mail），邮件含重置链接
  → 返回成功

POST /api/v1/user/reset-password（重置）
  → 验证 token 有效性（未过期、未使用）
  → 更新密码 hash（pkg/auth/password）
  → 标记 token 已使用（valid_time = now）
  → 返回成功，前端跳转登录页
```

### 4.4 LDAP 登录

```go
// backend/app/user/internal/biz/ldap.go
// 原 models/Member.go 中的 LDAP 逻辑迁移到此

func (uc *UserUsecase) LDAPLogin(ctx context.Context, account, password string) (*Member, error) {
    // 1. 用 pkg/ldap 验证账号密码
    // 2. 如果本地 md_members 没有该账号，自动创建（auth_method=ldap）
    // 3. 返回 member，由调用方签发 JWT
}
```

---

## 五、用户服务数据库表

用户服务**只读写以下三张表**，其余表由 mindoc-service 负责：

| 表名 | 说明 |
|------|------|
| `md_members` | 用户信息（账号/密码/角色/状态） |
| `md_member_tokens` | 密码重置 token |
| `md_options` | 站点全局配置（用户服务读取登录相关配置） |

```
注意：md_options 表同时被两个服务使用：
  - user-service：读取 ENABLE_REGISTER / ENABLED_CAPTCHA / SITE_NAME 等
  - mindoc-service：读取 ENABLE_ANONYMOUS / ENABLE_DOCUMENT_HISTORY 等
  两者只读，写入由管理后台接口（归属用户服务的管理员接口）负责
```

---

## 六、与 mindoc-service 的职责边界

```
                    ┌────────────────────────────────────────┐
                    │         职责边界示意                    │
                    │                                        │
  用户服务           │           主业务服务                   │
  ──────────        │           ──────────                   │
  登录/注册          │           文档 CRUD                    │
  JWT 签发           │           书籍管理                     │
  用户 CRUD          │           博客/评论                    │
  LDAP/钉钉          │           搜索/标签                    │
  验证码             │           附件上传                     │
  找回密码           │           文档导出                     │
  个人设置           │           团队管理                     │
  用户管理（管理员） │           书籍成员关系                  │
                    │           知识库集合                    │
                    └────────────────────────────────────────┘

  共享：
    JWT secret（通过配置共享，两服务各自校验，无需调用对方）
    MySQL 数据库（不同表，部分表只读共享如 md_options）
    Redis（不同 key 前缀：user: 和 mindoc:）
    pkg/ 公共包（auth/pagination/storage 等）
```

### mindoc-service 中需要用户信息时

```go
// ✅ 大多数情况：直接用 JWT 中的 user_id/role，无需调用用户服务
func (s *DocumentService) CreateDocument(ctx context.Context, req *pb.CreateDocumentRequest) (*pb.CreateDocumentReply, error) {
    claims := middleware.ClaimsFromContext(ctx)  // 从 JWT 取 user_id
    doc, err := s.biz.CreateDocument(ctx, claims.MemberId, req)
    // ...
}

// ✅ 少数情况：展示用户详情时，通过 gRPC 调用用户服务
func (s *BookService) GetBookMembers(ctx context.Context, req *pb.GetBookMembersRequest) (*pb.GetBookMembersReply, error) {
    relationships, _ := s.biz.GetBookMembers(ctx, req.BookKey)
    // 批量获取用户详情（头像、昵称）
    userIds := extractUserIds(relationships)
    users, _ := s.userClient.BatchGetUsers(ctx, userIds)
    // 组装结果...
}
```

---

## 七、用户服务配置文件

```yaml
# backend/configs/user/config.yaml

server:
  http:
    addr: 0.0.0.0:8080
    timeout: 5s
  grpc:
    addr: 0.0.0.0:9000      # 内部 gRPC，仅 Docker 内网可访问
    timeout: 5s

auth:
  jwt_secret: "shared-secret-with-mindoc"  # 与 mindoc-service 保持一致
  access_token_expire: 7200        # 2 小时
  refresh_token_expire: 604800     # 7 天
  refresh_token_remember: 2592000  # 记住我：30 天

data:
  database:
    driver: mysql
    source: "user:pass@tcp(mysql:3306)/mindoc?charset=utf8mb4&parseTime=True"
  redis:
    addr: redis:6379
    password: ""
    db: 0
    key_prefix: "user:"      # Redis key 前缀，避免与 mindoc-service 冲突

captcha:
  enabled: true
  length: 4
  expire: 300                # 验证码有效期（秒）

mail:
  enabled: false
  smtp_host: ""
  smtp_port: 465
  smtp_username: ""
  smtp_password: ""
  from_username: ""
  secure: true
  hourly_limit: 3            # 同一邮箱每小时最多发送次数
  expire_minutes: 60         # 重置链接有效分钟数

ldap:
  enabled: false
  host: ""
  port: 389
  base_dn: ""
  bind_dn: ""
  bind_password: ""
  user_filter: "(uid=%s)"

dingtalk:
  corp_id: ""
  app_key: ""
  app_secret: ""
  tmp_reader: ""
  qr_key: ""
  qr_secret: ""

app:
  enable_register: true       # 是否开放注册
  enable_captcha: false       # 是否强制验证码
  default_avatar: "/static/images/default_avatar.png"
  base_url: "http://localhost"
```

---

## 八、迁移路径

### Phase 1（与主服务同步开发）

1. 从 `backend/app/mindoc/` 中剥离 account 相关的 service / biz / data
2. 迁移至 `backend/app/user/`，参照 [migration-guide.md](migration-guide.md) 的 ORM 替换规范
3. 在 `backend/app/mindoc/internal/client/` 实现 gRPC 客户端
4. mindoc-service 的 JWT middleware 使用共享 secret 本地校验，**不调用**用户服务

### 对前端的影响（接口路径变更）

| 原接口路径 | 新接口路径 | 服务 |
|-----------|-----------|------|
| `/api/v1/account/login` | `/api/v1/user/login` | user-service |
| `/api/v1/account/register` | `/api/v1/user/register` | user-service |
| `/api/v1/account/captcha` | `/api/v1/user/captcha` | user-service |
| `/api/v1/settings/profile` | `/api/v1/user/profile` | user-service |
| `/api/v1/settings/password` | `/api/v1/user/password` | user-service |
| `/api/v1/manager/members/*` | `/api/v1/user/manager/members/*` | user-service |
| `/api/v1/docs/*` | `/api/v1/docs/*`（不变）| mindoc-service |
| `/api/v1/books/*` | `/api/v1/books/*`（不变）| mindoc-service |

前端只需在 `src/api/user.ts` 中统一配置用户服务的 `baseURL`，其他接口保持不变。
