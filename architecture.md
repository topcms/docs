# MinDoc 重构架构设计文档

> 原项目：MinDoc/TopDoc（Beego v2 + ORM + SSR 模板渲染）
> 目标架构：go-kratos（后端双服务）+ Vue 3（前端 Web）+ Tauri v2（桌面端）

---

## 一、整体架构概览

用户服务（User Service）作为独立微服务与主业务服务（MindDoc Service）并列部署，
两者均基于 go-kratos，通过 **gRPC 内部通信**，对外统一由 **Nginx 网关**路由。

```
┌───────────────────────────────────────────────────────────────────────┐
│                           客户端层                                      │
│   ┌────────────────────────┐        ┌───────────────────────────────┐ │
│   │   Web 浏览器（SPA）     │        │   桌面客户端（Tauri v2）       │ │
│   │   Vue 3 + Element Plus │        │   Vue 3（同一套代码）           │ │
│   └───────────┬────────────┘        └──────────────┬────────────────┘ │
└───────────────│──────────────────────────────────────│─────────────────┘
                │ HTTPS                                │ HTTP（本地）
┌───────────────▼──────────────────────────────────────▼─────────────────┐
│                       Nginx API 网关（:80/:443）                         │
│  /api/v1/user/*  → user-service                                         │
│  /api/v1/*       → mindoc-service                                        │
│  /*              → Vue 3 静态文件                                         │
└──────────────┬──────────────────────────────────────┬───────────────────┘
               │ HTTP :8080 / gRPC :9000               │ HTTP :8081 / gRPC :9001
┌──────────────▼──────────────────┐   ┌────────────────▼───────────────────┐
│       用户服务（User Service）    │   │    主业务服务（MindDoc Service）     │
│                                  │   │                                     │
│  ┌──────────┐  ┌───────────────┐ │   │  ┌──────────┐  ┌────────────────┐ │
│  │ HTTP srv │  │  gRPC srv     │ │   │  │ HTTP srv │  │  gRPC srv      │ │
│  │ :8080    │  │  :9000（对外） │ │   │  │ :8081    │  │  :9001（对外）  │ │
│  └────┬─────┘  └───────┬───────┘ │   │  └─────┬────┘  └───────┬────────┘ │
│       │                │         │   │        │               │           │
│  ┌────▼────────────────▼─────┐   │   │  ┌─────▼───────────────▼──────┐   │
│  │  service / biz / data      │   │   │  │  service / biz / data       │   │
│  │  登录/注册/Token/用户管理   │   │   │  │  文档/书籍/博客/评论/搜索    │   │
│  └───────────────────────────┘   │   │  └──────────────┬─────────────┘   │
│  DB: md_members / md_member_     │   │  DB: 其余所有表  │                  │
│       tokens / md_options         │   │                  │                  │
└──────────────────────┬───────────┘   │          gRPC 调用用户服务           │
                        │ gRPC :9000   └─────────────────┬───────────────────┘
                        └──────────────────────◄──────────┘
                          内部通信：GetUserInfo / ValidateToken
```

---

## 二、技术选型明细

### 后端

| 组件 | 选型 | 替换说明 |
|------|------|----------|
| Web 框架 | go-kratos v2 | 替换 Beego v2 |
| ORM | GORM v2 | 替换 Beego ORM |
| 认证 | golang-jwt/jwt v5 | 替换 Session + Cookie |
| 配置 | kratos/config（YAML） | 替换 conf/app.conf（ini） |
| 日志 | kratos/log + zap | 替换 beego/logs |
| 依赖注入 | google/wire | 新增 |
| 缓存 | go-redis/redis v9 | 原 cache/ 模块升级 |
| 数据库 | MySQL 5.7+ / SQLite | 保持不变 |

### 前端（Web + 桌面共用）

| 组件 | 选型 | 说明 |
|------|------|------|
| 框架 | Vue 3 + TypeScript | |
| 构建 | Vite 5 | |
| UI 组件 | Element Plus | 适合后台管理型应用 |
| 状态管理 | Pinia | 替换 Vuex |
| 路由 | Vue Router 4 | 含路由守卫（替换 filter.go） |
| HTTP 客户端 | axios | 含 JWT 拦截器 |
| Markdown 编辑器 | vditor | 替换原 editor.md |
| 国际化 | vue-i18n | 替换 beego/i18n |

### 桌面端

| 组件 | 选型 | 说明 |
|------|------|------|
| 桌面框架 | Tauri v2 | 系统 WebView，包体 ~10MB |
| 前端复用 | 同 frontend/ 代码 | 100% 复用 |
| 本地服务 | go-kratos sidecar | 内嵌后端二进制 |
| 原生能力 | Tauri 插件（fs/dialog/notification） | 文件保存、通知等 |

---

## 三、目录结构说明

### 项目根目录

```
mindoc/
├── backend/        # go-kratos 后端（见 §3.1）
├── frontend/       # Vue 3 前端，Web + 桌面共用（见 §3.2）
├── desktop/        # Tauri v2 桌面壳（见 §3.3）
├── scripts/        # 构建、迁移脚本
├── deploy/         # Docker / Nginx 部署配置
└── docs/           # 本文档所在目录
```

### 3.1 backend/ 结构

```
backend/
├── api/                          # Protobuf IDL + 生成代码（两个服务共享）
│   ├── user/v1/                  # 用户服务接口（对外 HTTP + 内部 gRPC）
│   ├── book/v1/                  # 项目/书籍服务
│   ├── document/v1/              # 文档服务
│   ├── manager/v1/               # 管理员服务
│   ├── blog/v1/                  # 博客服务
│   ├── comment/v1/               # 评论服务
│   ├── search/v1/                # 搜索服务
│   └── common/v1/                # 公共结构（分页、错误码）
│
├── app/
│   │
│   ├── user/                     # ★ 用户服务（独立部署）
│   │   ├── cmd/user/
│   │   │   ├── main.go           # 用户服务入口
│   │   │   └── wire.go           # Wire DI
│   │   └── internal/
│   │       ├── conf/             # 用户服务配置（DB/JWT/Mail/LDAP/DingTalk）
│   │       ├── server/           # HTTP :8080 + gRPC :9000
│   │       ├── middleware/       # JWT 校验（仅用于对外 HTTP 接口）
│   │       ├── service/          # 接口实现（Login/Register/Profile...）
│   │       ├── biz/              # 业务逻辑（认证/密码/Token/LDAP/钉钉）
│   │       └── data/             # 数据访问（仅操作用户相关表）
│   │                             #   md_members / md_member_tokens / md_options
│   │
│   └── mindoc/                   # ★ 主业务服务（独立部署）
│       ├── cmd/mindoc/
│       │   ├── main.go           # 主服务入口
│       │   └── wire.go           # Wire DI
│       └── internal/
│           ├── conf/             # 主服务配置（DB/UserService gRPC 地址）
│           ├── server/           # HTTP :8081 + gRPC :9001
│           ├── middleware/       # JWT 本地校验（共享 secret，无需调用用户服务）
│           ├── client/           # ★ gRPC 客户端（调用用户服务）
│           │                     #   user_client.go：GetUserInfo / BatchGetUsers
│           ├── service/          # 接口实现（book/document/blog/comment...）
│           ├── biz/              # 业务逻辑
│           └── data/             # 数据访问（操作除用户表以外的所有表）
│               └── cache/        # Redis 缓存
│
├── pkg/                          # 公共库（两个服务均可引用）
│   ├── auth/                     # JWT 工具（两个服务共用同一 secret 校验）
│   ├── mail/                     # 邮件（用户服务使用）
│   ├── converter/                # 文档导出（主服务使用）
│   ├── graphics/                 # 图片处理
│   ├── captcha/                  # 验证码（用户服务使用）
│   ├── dingtalk/                 # 钉钉登录（用户服务使用）
│   ├── ldap/                     # LDAP（用户服务使用）
│   ├── pagination/               # 分页工具（两个服务均用）
│   └── storage/                  # 文件存储（主服务使用）
│
└── configs/
    ├── user/
    │   ├── config.yaml           # 用户服务配置
    │   └── config.local.yaml
    └── mindoc/
        ├── config.yaml           # 主服务配置
        └── config.local.yaml
```

### 3.2 frontend/ 结构

```
frontend/
├── src/
│   ├── api/                      # HTTP 请求封装（对应各 proto service）
│   │   ├── request.ts            # axios 实例，含 JWT 拦截器和双端 baseURL
│   │   ├── account.ts
│   │   ├── book.ts
│   │   ├── document.ts
│   │   ├── manager.ts
│   │   ├── blog.ts
│   │   ├── comment.ts
│   │   ├── search.ts
│   │   ├── setting.ts
│   │   ├── label.ts
│   │   └── upload.ts
│   │
│   ├── router/                   # 路由（替代 routers/router.go + filter.go）
│   │   ├── index.ts
│   │   ├── guards.ts             # 路由守卫（鉴权逻辑）
│   │   └── routes/               # 按模块拆分路由配置
│   │       ├── account.ts
│   │       ├── book.ts
│   │       ├── document.ts
│   │       ├── manager.ts
│   │       ├── blog.ts
│   │       └── setting.ts
│   │
│   ├── stores/                   # Pinia 状态
│   │   ├── user.ts               # 用户信息 + JWT token
│   │   ├── book.ts               # 当前书籍状态
│   │   ├── document.ts           # 文档编辑状态
│   │   └── app.ts                # 全局配置（站点名、主题、语言）
│   │
│   ├── views/                    # 页面组件（替代 views/*.tpl）
│   │   ├── account/              # Login / Register / FindPassword
│   │   ├── home/                 # 首页
│   │   ├── book/                 # 项目管理
│   │   ├── document/             # 文档阅读/编辑
│   │   ├── manager/              # 后台管理
│   │   ├── blog/manage/          # 博客管理
│   │   ├── setting/              # 个人设置
│   │   ├── search/               # 搜索
│   │   ├── label/                # 标签
│   │   ├── itemsets/             # 知识库集合
│   │   └── error/                # 错误页（404/500）
│   │
│   ├── components/               # 公共组件
│   │   ├── editor/               # MarkdownEditor.vue（vditor 封装）
│   │   ├── document/             # DocTree.vue（替代 CreateDocumentTreeForHtml）
│   │   ├── common/               # Captcha / Pagination / FileUpload / QrCode
│   │   └── layout/               # DefaultLayout / DocLayout / AdminLayout
│   │
│   ├── composables/              # Vue 3 组合式函数
│   │   ├── useAuth.ts
│   │   ├── useDocument.ts
│   │   ├── useUpload.ts          # 上传（Web/桌面自动切换）
│   │   └── usePlatform.ts        # 判断 Web / Tauri 环境
│   │
│   ├── locales/                  # 国际化（替代 conf/*.ini）
│   │   ├── zh-CN.json
│   │   └── en-US.json
│   │
│   └── utils/
│       ├── request.ts
│       ├── token.ts              # JWT 存取
│       └── platform.ts           # Tauri 环境检测
│
└── public/
    └── favicon.ico
```

### 3.3 desktop/ 结构

```
desktop/
├── tauri.conf.json               # Tauri 配置（窗口、图标、权限、sidecar）
├── Cargo.toml
└── src/
    ├── main.rs                   # Tauri 入口，启动 go-kratos sidecar
    ├── lib.rs
    └── commands/                 # Tauri IPC 命令（桌面专属能力）
        ├── file.rs               # 文件系统（导出 PDF 保存到任意路径）
        ├── server.rs             # 启动/停止本地 go-kratos 服务
        └── notification.rs       # 系统通知
```

---

## 四、核心设计决策

### 4.1 认证体系：Session → JWT

**原机制：**
- `BaseController.Prepare()` 从 Session 读取登录用户
- SecureCookie 实现"记住我"功能
- `routers/filter.go` 对需鉴权路由插入 Filter

**新机制：**
- 登录接口返回 `access_token`（短效）+ `refresh_token`（长效）
- 前端 axios 拦截器自动附加 `Authorization: Bearer <token>`
- go-kratos middleware 校验 JWT，将用户信息注入 `context.Context`
- "记住我"改为延长 `refresh_token` 有效期

```go
// 从 context 获取当前用户（替代 c.Member）
func CurrentMember(ctx context.Context) *biz.Member {
    return ctx.Value(middleware.MemberKey{}).(*biz.Member)
}
```

### 4.2 文档树：HTML 字符串 → JSON 结构

**原机制：**
```go
// 后端拼 HTML 字符串，前端直接渲染
tree, _ := models.NewDocument().CreateDocumentTreeForHtml(bookId, selected)
c.Data["Result"] = template.HTML(tree)
```

**新机制：**
```json
// 后端返回 JSON 树，前端用 Element Plus el-tree 渲染
{
  "tree": [
    { "id": 1, "label": "第一章", "children": [
      { "id": 2, "label": "1.1 介绍", "children": [] }
    ]}
  ]
}
```

### 4.3 桌面端双模式

| 模式 | 场景 | 实现 |
|------|------|------|
| 远程服务模式 | 团队协作，数据存服务器 | Tauri WebView → 访问远程 kratos API |
| 本地服务模式 | 个人离线使用 | Tauri 启动 go-kratos sidecar → 访问 localhost |

```typescript
// frontend/src/utils/platform.ts
const isTauri = '__TAURI_INTERNALS__' in window

export const apiBaseURL = isTauri
  ? 'http://127.0.0.1:8080'
  : import.meta.env.VITE_API_BASE_URL
```

### 4.4 可直接复用的原有代码

以下模块无需重写，只需调整 import 路径后直接复用：

| 原目录 | 新位置 | 复用度 |
|--------|--------|--------|
| `utils/dingtalk/` | `backend/pkg/dingtalk/` | 100% |
| `utils/pagination/` | `backend/pkg/pagination/` | 100% |
| `mail/` | `backend/pkg/mail/` | 100% |
| `converter/` | `backend/pkg/converter/` | 100% |
| `graphics/` | `backend/pkg/graphics/` | 100% |
| `utils/cryptil/` | `backend/pkg/auth/password.go` | 95% |
| `utils/filetil/` | `backend/pkg/storage/` | 90% |
| `cache/` | `backend/app/mindoc/internal/data/cache/` | 80% |

---

## 五、API 接口规范

### 统一响应格式

```json
{
  "code": 0,
  "message": "ok",
  "data": { ... }
}
```

错误响应（复用原 `JsonResult` 错误码体系）：

```json
{
  "code": 6001,
  "message": "用户名格式不正确",
  "data": null
}
```

### 用户服务接口（由 user-service 响应）

```
POST   /api/v1/user/login             登录，返回 JWT
POST   /api/v1/user/register          注册
POST   /api/v1/user/logout            退出
POST   /api/v1/user/find-password     找回密码（发邮件）
POST   /api/v1/user/reset-password    重置密码（验 token）
GET    /api/v1/user/captcha           获取验证码（Base64 图片）
POST   /api/v1/user/dingtalk          钉钉登录
POST   /api/v1/user/refresh-token     刷新 JWT
GET    /api/v1/user/profile           获取当前用户信息
PUT    /api/v1/user/profile           更新个人资料
PUT    /api/v1/user/password          修改密码
PUT    /api/v1/user/avatar            上传头像
```

用户服务同时暴露 **gRPC 内部接口**（仅供 mindoc-service 调用，不对外）：

```protobuf
service UserInternalService {
  rpc GetUserInfo(GetUserInfoRequest) returns (UserInfo);        // 按 ID 查用户
  rpc BatchGetUsers(BatchGetUsersRequest) returns (BatchGetUsersReply); // 批量查
  rpc ValidateToken(ValidateTokenRequest) returns (TokenClaims); // 可选：集中校验
}
```

### 文档接口

```
GET    /api/v1/docs/:key              文档项目首页（含文档树）
GET    /api/v1/docs/:key/:id          阅读单篇文档
GET    /api/v1/docs/:key/tree         获取文档树 JSON
POST   /api/v1/docs/:key/create       创建文档
PUT    /api/v1/docs/:key/:id          更新文档
DELETE /api/v1/docs/:key/:id          删除文档
GET    /api/v1/docs/:key/:id/history  文档历史版本列表
POST   /api/v1/docs/:key/:id/restore  恢复历史版本
POST   /api/v1/docs/:key/search       书内搜索
POST   /api/v1/docs/:key/export       导出（PDF/Word）
POST   /api/v1/upload                 上传附件/图片
```

---

## 六、服务间通信设计

### 6.1 JWT 校验策略：本地校验（推荐）

两个服务**共享同一个 JWT secret**，mindoc-service 在本地直接解析 JWT，
**无需**每次请求都调用用户服务校验，避免额外的网络开销。

```
用户登录流程：
  前端 → user-service /api/v1/user/login
       → 验证账号密码 → 生成 JWT（含 user_id, role, username）
       → 返回 access_token + refresh_token

后续业务请求：
  前端 → Nginx → mindoc-service /api/v1/docs/...
       → middleware 本地解析 JWT（pkg/auth，共享 secret）
       → 注入 user_id 到 context
       → biz 层用 user_id 做权限判断（无需调用用户服务）
```

### 6.2 何时调用用户服务 gRPC

mindoc-service 仅在**需要展示用户详情**时通过 gRPC 调用用户服务：

| 场景 | 调用接口 |
|------|---------|
| 书籍成员列表（展示头像/昵称） | `GetUserInfo` / `BatchGetUsers` |
| 评论区展示作者信息 | `GetUserInfo` |
| 管理后台用户列表 | 管理后台直接对接用户服务 HTTP API |

```go
// backend/app/mindoc/internal/client/user_client.go
type UserClient struct {
    cc userpb.UserInternalServiceClient
}

func (c *UserClient) BatchGetUsers(ctx context.Context, ids []int64) ([]*userpb.UserInfo, error) {
    reply, err := c.cc.BatchGetUsers(ctx, &userpb.BatchGetUsersRequest{Ids: ids})
    // ...
}
```

### 6.3 服务发现

初期直接使用**静态配置**（主服务配置文件写入用户服务 gRPC 地址），
后期按需引入 etcd/consul：

```yaml
# backend/configs/mindoc/config.yaml
dependencies:
  user_service:
    grpc_addr: "user-service:9000"   # Docker 网络内服务名
    timeout: 2s
```

---

## 七、部署架构

### Web 部署（Docker Compose）

```
Nginx（:80/:443）
  ├── /api/v1/user/*  →  user-service（:8080）
  ├── /api/v1/*        →  mindoc-service（:8081）
  └── /*               →  Vue 3 静态文件

内部网络（docker network）：
  mindoc-service  →  gRPC  →  user-service（:9000）
```

Docker Compose 服务组成：

```yaml
services:
  nginx:         # 反向代理 + 静态文件
  user-service:  # 用户服务（HTTP :8080, gRPC :9000）
  mindoc-service: # 主业务服务（HTTP :8081, gRPC :9001）
  mysql:         # 数据库（共用，不同表）
  redis:         # 缓存（共用，不同 key 前缀）
```

### 桌面部署（Tauri sidecar）

桌面模式将两个 go-kratos 二进制均作为 sidecar 内嵌：

```
Tauri 安装包（.msi / .dmg / .AppImage）
  ├── 内嵌 Vue 3 静态资源（WebView 加载）
  ├── 内嵌 user-service 二进制（sidecar，监听 :8080/:9000）
  └── 内嵌 mindoc-service 二进制（sidecar，监听 :8081/:9001）
```
