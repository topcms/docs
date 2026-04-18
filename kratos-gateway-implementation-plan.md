# kratos-gateway 完整实现方案

> 本文档描述在现有 **`kratos-template`**、**`kratos-infra`** 基础上，将 **`kratos-gateway`** 从空壳落实为 **BFF/API 网关** 的完整方案，供评审后再决定是否编码落地。  
> 相关背景可参考仓库内 `参考1.md`～`参考3.md`、`bak参考3.md`～`bak参考5.md` 中的团队协作文与网关/鉴权/RBAC 思路。

---

## 一、文档目的与范围

| 项目 | 说明 |
|------|------|
| **目的** | 统一对外 HTTP 入口；经服务发现调用各域 gRPC；复用现有中间件与注册体系；与模板服务工程习惯一致。 |
| **范围** | 网关进程、配置、契约依赖方式、分阶段交付与验收；不包含具体业务表设计。 |
| **非目标** | 不替代 Nginx/Ingress 的 TLS 终结与静态资源；不重复实现一套与 `kratos-infra` 冲突的 JWT 解析逻辑。 |

---

## 二、背景与现状

### 2.1 当前仓库事实

- **`kratos-template`**：已具备 `wire`、`HTTP`+`gRPC`、**Consul 注册与发现**（`internal/registry/consul.go`）、`kratos-infra` 中间件（tracing → logging → 可选 JWT → recovery）、默认服务名 **`user.service`**（`cmd/server/main.go`）等，可作为「业务服务」标准形态。
- **`kratos-infra`**：已提供 `middleware/auth`（Bearer JWT）、`auth/jwt`（校验器）、`middleware/tracing`、`middleware/logging`、`middleware/recovery`、`registry/consul` 等，**网关应直接依赖，不复制代码**。
- **`kratos-gateway`**：目前仅有模块占位（如 `README`），**需按本方案从零搭工程骨架**。

### 2.2 与参考文档的对应关系

| 参考要点 | 本方案落点 |
|----------|------------|
| 多仓库 + 协议独立（参考1、参考3） | 网关 `go.mod` 以 **契约生成模块** 为依赖；中长期与 `api-contracts` 版本 tag 对齐。 |
| HTTP → 后端 gRPC（bak参考3） | 使用 Kratos 生成的 **`RegisterXxxHTTPServer(srv, client)`**，第二参数为 **gRPC Client**。 |
| 登录与 JWT 在网关侧校验（bak参考4） | 复用 `infraauth` + `infrajwt.NewTokenValidator`，与 template 的 `internal/server/http.go` 顺序一致。 |
| RBAC（bak参考5） | **可选二阶段**：JWT 通过后调用 auth 服务的 `CheckPermission`（需 proto 与实现就绪）。 |

---

## 三、目标架构

```
                    ┌─────────────────────┐
  浏览器 / App ───► │   kratos-gateway     │  HTTP（对外唯一业务入口，可选）
                    │   （仅 BFF，无业务库） │
                    └──────────┬──────────┘
                               │ gRPC + discovery:///
         ┌─────────────────────┼─────────────────────┐
         ▼                     ▼                     ▼
  user.service           order.service …     auth.service（可选）
         │                     │
         └──────────┬──────────┘
                    ▼
            Consul（服务注册与发现）
```

**说明**：

- 网关 **默认不提供业务 gRPC Server**（仅 HTTP）；若未来需要内部管理接口再增加。
- 各业务服务 **仍自行注册** Consul；网关 **仅作为 Consumer** 使用 `Discovery` 建连（与 template 中 `data` 层远程调用方式一致）。

---

## 四、设计原则

1. **与 `kratos-template` 同构**：`cmd/server`、`internal/server`、`internal/conf`、`configs`、`Makefile`、`wire`，降低团队心智负担。
2. **无业务持久化**：网关不写业务 MySQL；必要时仅 **Redis**（限流、黑名单等，属可选增强）。
3. **契约驱动**：路由与请求体由 **proto 生成** 的 HTTP 绑定决定，避免手写大量转发胶水代码。
4. **共享基础设施**：所有横切能力以 **`kratos-infra`** 为准；配置语义与 template 对齐（JWT、Consul、日志等）。

---

## 五、契约与模块依赖

### 5.1 推荐依赖方式（按成熟度递进）

| 阶段 | 做法 | 说明 |
|------|------|------|
| **A. 共仓过渡期** | `go.work` 或 `replace` 指向本仓库内 **`api/` 生成代码包**（与 template 同源生成） | 快，适合先把链路跑通；注意 **不要** `import` 整个 `kratos-template` 业务包。 |
| **B. 规范期** | 独立 **`api-contracts`**（或等价）仓库，打 tag，网关与业务服务 **同版本 bump** | 与 `参考3.md` 一致，利于多团队并行。 |

### 5.2 包引用边界

- **允许**：`github.com/xxx/api/user/v1`（仅生成代码：`*.pb.go`、`*_http.pb.go`、`*_grpc.pb.go`）。
- **禁止**：引用 `kratos-template/internal/...`（避免 DB、业务 wire、循环依赖）。

---

## 六、网关目录结构（建议）

```text
kratos-gateway/
├── cmd/server/
│   ├── main.go
│   └── wire.go
├── internal/
│   ├── server/
│   │   └── http.go          # HTTP Server + 中间件 + RegisterXxxHTTPServer
│   ├── client/              # 可选：按域封装 gRPC Client 创建
│   │   └── user.go
│   └── conf/
│       ├── conf.proto       # 裁剪：仅 Server / Auth / Registry / Log（无 Data）
│       └── buf.gen.yaml
├── configs/
│   ├── config.yaml
│   └── registry.yaml
├── build/docker/Dockerfile
├── Makefile
├── go.mod
└── README.md
```

与 template 的差异：**无 `internal/biz`、`internal/data`**，或仅保留极薄适配层（如聚合多个 client 的 facade，按需）。

---

## 七、核心实现要点

### 7.1 服务发现与 gRPC Client

- 使用与 template 相同的 **`kratos-infra/registry/consul`** 初始化 `Discovery`。
- 使用 `github.com/go-kratos/kratos/v2/transport/grpc`：

  - `grpc.DialInsecure(ctx, grpc.WithEndpoint("discovery:///user.service"), grpc.WithDiscovery(d), grpc.WithTimeout(...))`
  - 其中 **`user.service`** 必须与被调服务 `kratos.New(kratos.Name("..."))` 一致（与 template `main` 及 `configs` 中 `remote.user_service.service_name` 约定相同）。

### 7.2 HTTP 层绑定（Kratos 生成代码）

- 对每个已暴露 HTTP 的 gRPC 服务，在网关中：

  `userv1.RegisterUserServiceHTTPServer(httpServer, userGRPCClient)`

- **语义**：生成的 HTTP Handler 将 HTTP 请求转为 gRPC 调用；**第二参数类型**为 gRPC **Client** 接口，与业务服务里传入 **Service 实现** 相对，二者共用同一套生成代码。

### 7.3 `kratos.App` 选项

- `kratos.Name("gateway")`（或公司统一命名如 `api.gateway`），便于 Consul UI 区分实例。
- `kratos.Server(httpServer)`。
- `kratos.Registrar(registrar)`：**建议开启**，便于运维与健康检查脚本发现网关进程（与 bak参考3 一致）；若纯无状态且运维不依赖注册可再评估。

### 7.4 中间件顺序（与 template 对齐）

建议顺序：

1. `infratracing.Server()`
2. `infralogging.Server(logger)`
3. `infraauth.Server(validate)`（**可选**，`validate == nil` 时不启用）
4. `infrarecovery.Server()`

**网关特有需求**：在 JWT 之前或之中支持 **路径白名单**（如 `/health`、`/v1/auth/login`），避免未登录无法访问登录接口。实现方式三选一即可：

- 在网关增加 **薄中间件**：白名单路径直接 `next`，否则再走 `infraauth`；
- 或扩展 `kratos-infra` 的 auth 中间件支持配置（需改公共库，影响面大时放后）。

### 7.5 将 Trace / Request-ID 传到下游

- 入站已有 `X-Request-Id`（见 `kratos-infra/middleware/tracing`）。
- 出站 gRPC 调用时，在 **client 侧** 将 trace / request id 写入 **gRPC metadata**，便于业务服务与网关日志关联（具体 key 与公司 OTel 规范一致，如 W3C `traceparent` 或内部约定）。

---

## 八、配置设计（裁剪版 Bootstrap）

建议在网关的 `conf.proto` 中仅保留：

| 分组 | 字段 | 说明 |
|------|------|------|
| `Server` | `http` | 监听地址、超时 |
| `Auth` | `jwt` | 与 token 颁发方一致的 `secret` / `issuer` / `audience`；`audience` 可设为网关专用值（如 `kratos-gateway`） |
| `Registry` | `consul` | 与 template 同结构 |
| `Log` | 与 template 一致 | 便于统一采集 |

**不包含** `Data.database` / `Data.redis`（除非后续做限流等）。

可增加 **网关专用配置**（可选）：

- `gateway.public_paths`：字符串列表前缀匹配；
- `gateway.backends.user_service_name`：默认 `user.service`，便于多环境覆盖。

---

## 九、安全能力分阶段

| 阶段 | 内容 | 依赖 |
|------|------|------|
| **MVP** | HTTPS 由前置 Nginx/Ingress 处理；网关 HTTP + 可选 JWT | 无 |
| **增强** | 全路径 JWT + 白名单 | `auth.jwt` 配置 |
| **进阶** | RBAC：`CheckPermission(uid, method, path)` | auth 服务 proto、实现及缓存策略（见 bak参考5） |

---

## 十、可观测性与运维

- **日志**：沿用 `kratos-infra` + template 的 zap/json 配置思路。
- **指标**：Kratos 默认 metrics；Prometheus 抓取网关进程 HTTP 端口（与 template 文档中 Prometheus 小节一致）。
- **健康检查**：提供 `GET /health`（或 Kratos 标准健康接口），供 K8s probe 与负载均衡使用。

---

## 十一、与 `kratos-template` 的协作关系

| 维度 | template（业务服务） | gateway |
|------|----------------------|---------|
| 对外 HTTP | 可保留（联调、内部），生产可由网关收口 | **主对外入口** |
| gRPC | 提供实现 | **仅 Client** |
| Consul | Registrar + 可选 Discovery | **Discovery 必选**；Registrar 建议 |
| JWT | 可按域开启校验服务间或直连场景 | **面向终端流量时主校验点** |

---

## 十二、分阶段实施计划（建议）

### 阶段 A：链路跑通（MVP）

1. 初始化 `kratos-gateway` 工程：`go mod`、`wire`、`Makefile`、裁剪 `conf`。
2. 接入 Consul `Discovery`，为 **user** 域建立 gRPC Client 并完成 `RegisterUserServiceHTTPServer`。
3. 本地：Consul + 启动 `kratos-template` + 启动网关，验证 HTTP 路径与业务服务一致。

**验收**：经网关访问用户相关 HTTP 接口，行为与直连 template HTTP 一致（在相同后端版本下）。

### 阶段 B：工程化与安全

1. Dockerfile / K8s Deployment 与 template 仓库风格对齐。
2. 打开 JWT，配置 `audience`/`issuer`，实现 **登录等路径白名单**。
3. Client 侧 metadata 传递 trace/request id。

**验收**：未带 Token 访问受保护接口返回 401；白名单接口可匿名；日志可关联一次请求。

### 阶段 C：多域与扩展

1. 每新增一个后端服务，增加对应 **gRPC Client** + **RegisterXxxHTTPServer**（契约升级走 `api-contracts` 流程）。
2. 按需引入 Redis 限流或接入 API 网关策略（与本文档可拆独立 ADR）。

### 阶段 D：RBAC（可选）

1. 在契约中定义 `Auth.CheckPermission`。
2. 网关 JWT 后增加 RBAC 中间件，调用 auth 服务；注意 **超时、熔断与本地缓存**（见 bak参考5）。

---

## 十三、风险与对策

| 风险 | 对策 |
|------|------|
| 契约版本漂移 | 锁定 `api-contracts` tag；CI 校验网关与依赖服务引用的 api 版本。 |
| 网关成为单点 | 多副本 + K8s HPA；Consul 健康检查；前置负载均衡。 |
| 重复实现 JWT | 强制只使用 `kratos-infra` 校验器。 |
| 循环依赖 | 网关禁止 import 业务 `internal`。 |

---

## 十四、本地联调顺序（参考）

1. 启动 Consul（及可选 Jaeger/Prometheus）。
2. 启动 `kratos-template`（已注册 `user.service`）。
3. 启动 `kratos-gateway`。
4. 请求网关对外 HTTP 地址，验证路由与鉴权。

---

## 十五、文档修订记录

| 日期 | 修订说明 |
|------|----------|
| 2026-04-18 | 初稿：基于 `kratos-template` / `kratos-infra` 现状与参考文档整理完整落地方案。 |

---

## 十六、结论

本方案将 **`kratos-gateway`** 定位为 **无业务库的 Kratos HTTP BFF**：通过 **契约生成的 HTTP 注册函数** + **Consul 服务发现下的 gRPC Client** 对接现有及未来的业务服务，并 **复用 `kratos-infra` 的中间件与注册组件**。实施时严格区分 **契约模块** 与 **业务服务模块** 依赖边界，按阶段 A→B→C→（D）推进，可在控制风险的前提下与团队现有模板工程保持一致。

审阅通过后，可按 **第十二节** 阶段拆分创建任务与分支进行编码。
