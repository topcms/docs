# MinDoc 重构计划

> 本文档描述从 Beego v2 单体 SSR 应用迁移到  
> go-kratos（后端）+ Vue 3（前端 Web）+ Tauri v2（桌面端）的分阶段实施计划。

---

## 一、总体时间线

```
Phase 1  基础建设与认证模块          ████████                   4 周
Phase 2  核心文档功能                          ████████████       6 周
Phase 3  书籍项目管理与管理后台                            ████   4 周
Phase 4  附属功能与桌面端                                      ██ 4 周
─────────────────────────────────────────────────────────────────
总计约 18 周（~4.5 个月）
```

---

## 二、Phase 1：基础建设 + 用户服务（第 1-4 周）

### 目标
- **同时**搭建用户服务（`app/user/`）和主服务（`app/mindoc/`）的项目骨架
- 完成前端 Vue 3 项目初始化
- 实现完整的登录/注册/JWT 认证链路（用户服务独立对外）
- 完成两服务间 gRPC 通信框架

### 后端任务

| 周次 | 任务 | 对应原文件 | 产出 |
|------|------|-----------|------|
| 第 1 周 | **用户服务**初始化（kratos new、Wire、DB 连接） | `main.go`、`conf/` | `app/user/cmd/user/main.go` |
| 第 1 周 | **主服务**初始化（kratos new、Wire、DB 连接） | `main.go` | `app/mindoc/cmd/mindoc/main.go` |
| 第 1 周 | 公共包初始化：pkg/auth（JWT + 密码 hash） | `utils/cryptil/` | `pkg/auth/` |
| 第 2 周 | 迁移 Member 数据层 → **用户服务** data 层 | `models/Member.go` | `app/user/internal/data/member.go` |
| 第 2 周 | 迁移 MemberToken 数据层 → **用户服务** | `models/MemberToken.go` | `app/user/internal/data/member_token.go` |
| 第 2 周 | 迁移 Options 数据层 → **用户服务**（登录配置读取） | `models/Options.go`（部分） | `app/user/internal/data/options.go` |
| 第 2 周 | 迁移登录/注册/找回密码业务逻辑 → **用户服务** biz | `controllers/AccountController.go` | `app/user/internal/biz/user.go`、`token.go` |
| 第 3 周 | 定义 `api/user/v1/user.proto`（含 HTTP + 内部 gRPC 接口） | 新建 | `api/user/v1/` |
| 第 3 周 | 实现 UserService（Login/Register/Logout/FindPassword/ResetPassword/Captcha） | `controllers/AccountController.go` | `app/user/internal/service/user.go` |
| 第 3 周 | 实现 UserInternalService（GetUserInfo/BatchGetUsers，gRPC only） | 新建 | `app/user/internal/service/user_internal.go` |
| 第 3 周 | 用户服务 JWT middleware（对外 HTTP 接口鉴权） | `routers/filter.go` | `app/user/internal/middleware/auth.go` |
| 第 3 周 | **主服务** JWT middleware（本地校验，共享 secret） | `routers/filter.go` | `app/mindoc/internal/middleware/auth.go` |
| 第 3 周 | **主服务** gRPC client（调用用户服务） | 新建 | `app/mindoc/internal/client/user_client.go` |
| 第 4 周 | 迁移 pkg/mail | `mail/` | `pkg/mail/`（直接复用） |
| 第 4 周 | 迁移 pkg/dingtalk + 钉钉登录接口 | `utils/dingtalk/`、`AccountController.DingTalkLogin` | `pkg/dingtalk/`、`app/user/internal/biz/dingtalk.go` |
| 第 4 周 | 迁移 pkg/ldap + LDAP 登录 | LDAP 相关代码 | `pkg/ldap/`、`app/user/internal/biz/ldap.go` |
| 第 4 周 | 验证码接口（Redis 存 captcha，Base64 返回） | `AccountController.Captcha` | `app/user/internal/biz/captcha.go` |

### 前端任务

| 周次 | 任务 | 对应原模板 | 产出 |
|------|------|-----------|------|
| 第 1 周 | Vue 3 + Vite + Element Plus 项目初始化 | - | `frontend/` 骨架 |
| 第 1 周 | axios 封装（JWT 拦截器、双端 baseURL） | - | `src/api/request.ts` |
| 第 1 周 | Pinia user store（token 存取） | - | `src/stores/user.ts` |
| 第 2 周 | 登录页面 | `account/login.tpl` | `src/views/account/Login.vue` |
| 第 2 周 | 注册页面 | `account/register.tpl` | `src/views/account/Register.vue` |
| 第 2 周 | 找回密码页面 | `account/find_password_*.tpl` | `src/views/account/FindPassword.vue` |
| 第 2 周 | 验证码组件（Base64 展示 + 刷新） | - | `src/components/common/Captcha.vue` |
| 第 3 周 | 路由配置 + 路由守卫（替代 filter.go） | `routers/router.go` | `src/router/` |
| 第 3 周 | 基础布局组件（导航、侧边栏） | `views/widgets/` | `src/components/layout/` |
| 第 4 周 | 全局配置获取（站点名、是否开启注册） | `BaseController.Prepare()` | `src/stores/app.ts` |
| 第 4 周 | 国际化初始化（vue-i18n） | `conf/*.ini` | `src/locales/` |

### Phase 1 验收标准
- [ ] `POST /api/v1/user/login` 登录并获取 JWT（由用户服务响应）
- [ ] JWT 失效后 mindoc-service 返回 401，前端跳转登录页
- [ ] 注册、找回密码、钉钉登录流程走通
- [ ] 验证码可正常展示与校验
- [ ] mindoc-service 通过 gRPC 可成功调用用户服务 BatchGetUsers

---

## 三、Phase 2：核心文档功能（第 5-10 周）

### 目标
- 完成文档 CRUD、文档树、历史版本、全文搜索
- 前端实现文档阅读页 + Markdown 编辑器
- 这是 MinDoc 的核心功能，工作量最大

### 后端任务

| 周次 | 任务 | 对应原文件 | 产出 |
|------|------|-----------|------|
| 第 5 周 | 迁移 Document 数据层 | `models/DocumentModel.go` | `internal/data/document.go` |
| 第 5 周 | 迁移 DocumentHistory 数据层 | `models/DocumentHistory.go` | `internal/data/document_history.go` |
| 第 5 周 | 迁移 DocumentTree 工具 | `models/DocumentTree.go` | `internal/data/document_tree.go` |
| 第 5 周 | 迁移 Attachment 数据层 | `models/AttachmentModel.go` | `internal/data/attachment.go` |
| 第 6 周 | 文档业务层（CRUD/树构建/历史/搜索） | `controllers/DocumentController.go`（业务部分） | `internal/biz/document.go` |
| 第 6 周 | 文档树 JSON 化（替代 CreateDocumentTreeForHtml） | `models/DocumentTree.go` | `internal/biz/document.go` |
| 第 7 周 | 定义 document.proto，生成代码 | 新建 | `api/document/v1/` |
| 第 7 周 | 实现 DocumentService（CRUD、树、历史） | `controllers/DocumentController.go` | `internal/service/document.go` |
| 第 8 周 | 文件上传接口（图片/附件） | `DocumentController.Upload` | `internal/service/document.go` |
| 第 8 周 | 文档导出接口（PDF/Word）+ converter 复用 | `DocumentController.Export`、`converter/` | `internal/biz/export.go`、`pkg/converter/` |
| 第 9 周 | 全文搜索实现（SQLite FTS / MySQL FULLTEXT） | `controllers/SearchController.go` | `internal/biz/search.go`、`internal/service/search.go` |
| 第 9 周 | 定义 search.proto，生成代码 | 新建 | `api/search/v1/` |
| 第 10 周 | 书内搜索（/docs/:key/search） | `DocumentController.Search` | `internal/service/document.go` |
| 第 10 周 | QrCode 接口（二维码生成） | `DocumentController.QrCode` | `internal/service/document.go` |

### 前端任务

| 周次 | 任务 | 对应原模板 | 产出 |
|------|------|-----------|------|
| 第 5 周 | vditor Markdown 编辑器封装 | `static/js/editor.md/` | `src/components/editor/MarkdownEditor.vue` |
| 第 5 周 | Markdown 只读预览组件 | - | `src/components/editor/MarkdownPreview.vue` |
| 第 6 周 | 文档树组件（el-tree，替代 HTML 字符串） | `CreateDocumentTreeForHtml` | `src/components/document/DocTree.vue` |
| 第 6 周 | 文档阅读页（左树右内容布局） | `document/*_read.tpl` | `src/views/document/Read.vue` |
| 第 7 周 | 文档阅读页多主题支持（default/dark 等） | 多个 `*_read.tpl` | CSS 变量动态切换 |
| 第 7 周 | 文档编辑页 | `document/edit.tpl` | `src/views/document/Edit.vue` |
| 第 8 周 | 文件上传组件（图片/附件，含拖拽） | - | `src/components/common/FileUpload.vue` |
| 第 8 周 | 文档历史版本对比 | `DocumentController.Compare` | `src/views/document/History.vue` |
| 第 9 周 | 全局搜索页面 | `SearchController.Index` | `src/views/search/Index.vue` |
| 第 10 周 | 文档导出功能（轮询下载进度） | `DocumentController.Export` | `src/views/document/Export.vue` |

### Phase 2 验收标准
- [ ] 可以创建、编辑、删除文档，Markdown 实时预览正常
- [ ] 文档树可拖拽排序，层级展示正确
- [ ] 文档历史版本可查看、恢复
- [ ] 图片/附件上传正常
- [ ] 全文搜索返回结果正确
- [ ] 文档导出 PDF 可下载

---

## 四、Phase 3：书籍项目管理与管理后台（第 11-14 周）

### 目标
- 完成书籍/项目的全部管理功能（BookController）
- 完成后台管理模块（ManagerController）
- 完成个人设置模块（SettingController）

### 后端任务

| 周次 | 任务 | 对应原文件 | 产出 |
|------|------|-----------|------|
| 第 11 周 | 迁移 Book 数据层 | `models/BookModel.go`、`models/BookResult.go` | `internal/data/book.go` |
| 第 11 周 | 迁移 Relationship 数据层（书籍成员关系） | `models/Relationship.go` | `internal/data/relationship.go` |
| 第 11 周 | 迁移 Team 相关数据层 | `models/Team.go`、`models/TeamMember.go`、`models/TeamRelationship.go` | `internal/data/team.go` |
| 第 11 周 | 书籍业务层（创建/设置/发布/转让/权限） | `controllers/BookController.go` | `internal/biz/book.go` |
| 第 12 周 | 定义 book.proto，生成代码 | 新建 | `api/book/v1/` |
| 第 12 周 | 实现 BookService | `controllers/BookController.go`、`controllers/BookMemberController.go` | `internal/service/book.go` |
| 第 12 周 | 迁移 Options 数据层（站点配置） | `models/Options.go` | `internal/data/options.go` |
| 第 13 周 | 定义 manager.proto，生成代码 | 新建 | `api/manager/v1/` |
| 第 13 周 | 实现 ManagerService（用户/书籍/附件/标签/团队/Itemsets 管理） | `controllers/ManagerController.go` | `internal/service/manager.go` |
| 第 13 周 | 迁移 Itemsets 数据层 | `models/Itemsets.go` | `internal/data/itemsets.go` |
| 第 13 周 | 迁移 Label 数据层 | `models/LabelModel.go` | `internal/data/label.go` |
| 第 14 周 | 实现 SettingService（个人设置、密码修改、头像上传） | `controllers/SettingController.go` | `internal/service/setting.go` |
| 第 14 周 | 实现 TemplateService | `controllers/TemplateController.go`、`models/Template.go` | `internal/service/template.go` |

### 前端任务

| 周次 | 任务 | 对应原模板 | 产出 |
|------|------|-----------|------|
| 第 11 周 | 首页（项目列表） | `HomeController.Index` | `src/views/home/Index.vue` |
| 第 11 周 | 书籍列表、创建 | `book/index.tpl`、`book/create.tpl` | `src/views/book/Index.vue`、`Create.vue` |
| 第 12 周 | 书籍设置（基本信息、封面上传、成员管理、团队） | `book/setting.tpl`、`book/users.tpl` | `src/views/book/Setting.vue`、`Users.vue`、`Team.vue` |
| 第 12 周 | 书籍 Dashboard（统计信息） | `book/dashboard.tpl` | `src/views/book/Dashboard.vue` |
| 第 13 周 | 管理后台布局 + 菜单 | `manager/*.tpl` | `src/components/layout/AdminLayout.vue` |
| 第 13 周 | 用户管理（列表/创建/编辑/禁用） | `manager/users.tpl`、`manager/edit_users.tpl` | `src/views/manager/Users.vue` |
| 第 13 周 | 书籍管理、附件管理 | `manager/books.tpl`、`manager/attach_list.tpl` | `src/views/manager/Books.vue`、`Attach.vue` |
| 第 14 周 | 站点设置、团队管理、知识库集合 | `manager/setting.tpl`、`manager/team.tpl` | `src/views/manager/Setting.vue`、`Team.vue`、`Itemsets.vue` |
| 第 14 周 | 个人设置页面（资料/密码/头像） | `setting/index.tpl`、`setting/password.tpl` | `src/views/setting/` |

### Phase 3 验收标准
- [ ] 书籍创建/设置/发布/删除流程完整
- [ ] 书籍成员管理（添加、修改角色、移除）
- [ ] 管理员可管理全站用户、书籍、附件
- [ ] 站点全局设置可正常修改
- [ ] 个人设置（昵称、密码、头像）可正常保存

---

## 五、Phase 4：附属功能与桌面端（第 15-18 周）

### 目标
- 完成博客、评论、标签、知识库集合等附属功能
- 桌面端 Tauri 壳搭建与联调

### 后端任务

| 周次 | 任务 | 对应原文件 | 产出 |
|------|------|-----------|------|
| 第 15 周 | 迁移 Blog 数据层、业务层 | `models/Blog.go`、`controllers/BlogController.go` | `internal/data/blog.go`、`internal/biz/blog.go` |
| 第 15 周 | 定义 blog.proto，实现 BlogService | 新建 | `api/blog/v1/`、`internal/service/blog.go` |
| 第 15 周 | 迁移 Comment 数据层、业务层 | `models/comment.go`、`controllers/comment.go` | `internal/data/comment.go`、`internal/biz/comment.go` |
| 第 15 周 | 定义 comment.proto，实现 CommentService | 新建 | `api/comment/v1/`、`internal/service/comment.go` |
| 第 16 周 | 标签（Label）服务完整实现 | `controllers/LabelController.go` | `internal/service/label.go` |
| 第 16 周 | 知识库集合（Itemsets）服务完整实现 | `controllers/ItemsetsController.go` | `internal/service/itemsets.go` |
| 第 16 周 | Logs 数据层（操作日志） | `models/Logs.go` | `internal/data/logs.go` |
| 第 16 周 | 整体接口回归测试 | - | API 测试用例 |

### 前端任务

| 周次 | 任务 | 对应原模板 | 产出 |
|------|------|-----------|------|
| 第 15 周 | 博客列表、博客详情 | `blog/list.tpl`、`blog/index.tpl` | `src/views/blog/List.vue`、`Index.vue` |
| 第 15 周 | 博客管理（创建/编辑/删除） | `blog/manage/*.tpl` | `src/views/blog/manage/` |
| 第 15 周 | 评论组件（列表/发布/回复） | `comment/index.tpl` | `src/components/common/Comment.vue` |
| 第 16 周 | 标签页面、知识库集合页面 | `label/index.tpl`、`items/index.tpl` | `src/views/label/`、`src/views/itemsets/` |
| 第 16 周 | 错误页面（404/500） | `errors/error.tpl` | `src/views/error/` |
| 第 16 周 | 前端整体回归测试 | - | E2E 测试 |

### 桌面端任务

| 周次 | 任务 | 说明 | 产出 |
|------|------|------|------|
| 第 17 周 | Tauri v2 项目初始化（在 frontend/ 基础上） | `npx tauri init` | `desktop/src-tauri/` |
| 第 17 周 | 配置 sidecar（将 go-kratos 二进制内嵌） | `tauri.conf.json` | 本地服务启动 |
| 第 17 周 | 平台检测工具（isTauri 判断） | `src/utils/platform.ts` | 双端 baseURL 切换 |
| 第 17 周 | 桌面端文件选择（导出保存） | `desktop/src/commands/file.rs` | `useUpload.ts` 桌面分支 |
| 第 18 周 | 系统通知（文档发布/导出完成） | `desktop/src/commands/notification.rs` | 桌面通知 |
| 第 18 周 | 系统托盘（最小化到托盘） | `tauri.conf.json` | 托盘图标与菜单 |
| 第 18 周 | 桌面端打包测试（Windows/macOS） | `tauri build` | 安装包 `.msi`/`.dmg` |

### Phase 4 验收标准
- [ ] 博客发布、评论、标签、知识库集合功能完整
- [ ] 桌面端（Windows）可正常安装启动
- [ ] 桌面端本地模式：内嵌 go-kratos 可自动启动
- [ ] 桌面端文件保存到任意本地路径
- [ ] 前后端整体回归测试通过

---

## 六、用户服务抽离的额外注意事项

### 数据库共享策略
- 两个服务**共用同一个 MySQL 实例**，通过表划分职责
- `md_options` 表两服务均可读，写入权限归用户服务（管理员设置接口）
- Redis 使用不同 key 前缀：`user:` 和 `mindoc:`，避免冲突

### JWT Secret 共享
- 两个服务的配置文件中 `auth.jwt_secret` 必须保持一致
- 生产环境通过**配置中心**（或 K8s Secret）统一下发，禁止硬编码

### 桌面端 sidecar 变更
- 原来一个 sidecar → 现在**两个 sidecar**
- Tauri 启动顺序：先启动 user-service，再启动 mindoc-service（等待 gRPC 就绪）

### 前端接口路径调整
- 将 `src/api/account.ts` 拆分为 `src/api/user.ts`（用户服务）
- `user.ts` 的 `baseURL` 指向用户服务（同 Nginx 统一入口，前缀 `/api/v1/user/`）

---

## 七、各模块复用度与风险评估

| 模块 | 可复用度 | 重写工作量 | 主要风险 |
|------|---------|----------|---------|
| 认证（AccountController） | 逻辑 80%，代码 20% | 中 | Session→JWT 切换需全面测试 |
| 文档 CRUD（DocumentController） | 逻辑 90%，代码 20% | 高 | 文档树 JSON 化、编辑器迁移 |
| 书籍管理（BookController） | 逻辑 85%，代码 20% | 中高 | 权限校验逻辑复杂 |
| 管理后台（ManagerController） | 逻辑 90%，代码 20% | 中 | 页面多但逻辑相对简单 |
| 博客（BlogController） | 逻辑 90%，代码 20% | 低中 | 独立性强 |
| 评论（CommentController） | 逻辑 90%，代码 20% | 低 | 相对简单 |
| 数据层（models/） | ORM 标签 100% 重写 | 高 | 表名前缀、关联查询适配 |
| 工具包（utils/mail/converter） | 95% 直接复用 | 低 | import 路径修改 |
| 前端（views/*.tpl） | 0% 复用，完全重写 | 最高 | 编辑器、文档树交互 |
| 桌面端 | 新建 | 低中 | Tauri sidecar 配置 |

---

## 八、并行开发建议

Phase 2-3 阶段，建议**前后端并行开发**，约定好接口协议后各自推进：

```
后端开发者           前端开发者
────────────         ────────────
定义 proto ──────►  根据 proto 生成 TypeScript 类型
实现 Service         实现页面组件（Mock 数据先联调）
联调接口 ◄─────────  联调接口
```

使用 **Apifox / Postman** 管理接口文档，后端先写 proto 注释，自动生成 API 文档。

---

## 九、代码迁移顺序（data 层）

data 层迁移建议严格按以下顺序进行（从无依赖到有依赖）：

```
第 1 批（无外键依赖）：
  Options → Member → MemberToken → Team → Label → Template → Logs

第 2 批（依赖 Member）：
  Book → TeamMember → Blog

第 3 批（依赖 Book + Member）：
  Relationship → TeamRelationship → Document → Attachment → Itemsets

第 4 批（依赖 Document）：
  DocumentHistory → DocumentTree → Comment
```

---

## 十、里程碑检查点

| 里程碑 | 时间节点 | 可验证内容 |
|--------|---------|-----------|
| M1：认证可用 | 第 4 周末 | 用户服务独立运行，登录/注册/JWT 完整链路，两服务 gRPC 通信正常 |
| M2：文档核心可用 | 第 10 周末 | 文档创建/编辑/阅读/搜索/导出 |
| M3：管理功能可用 | 第 14 周末 | 书籍管理、后台管理、个人设置 |
| M4：全功能 Web 版上线 | 第 16 周末 | 全量功能回归测试通过 |
| M5：桌面端发布 | 第 18 周末 | Windows/macOS 安装包发布 |
