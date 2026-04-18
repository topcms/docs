好，最后两块：**RBAC 权限系统 + 多环境(dev/staging/prod)**。
补完这两块，整套就真的是公司级微服务体系了。

---

# 一、RBAC 权限系统（不是简单角色）

先讲清楚目标：

不是

* “是否登录” ✔（已经做了）

而是

* **这个用户能不能访问这个接口？**

---

## RBAC 模型设计

```text
User → Role → Permission → API
```

表设计（最小可用版）：

```sql
users(id, name)

roles(id, name)
user_roles(user_id, role_id)

permissions(id, method, path)
role_permissions(role_id, permission_id)
```

例：

| role   | permission        |
| ------ | ----------------- |
| admin  | GET /v1/user/list |
| admin  | POST /v1/user/ban |
| normal | GET /v1/user/info |

---

## 1️⃣ auth-service 增加权限查询接口

```proto
service Auth {
  rpc CheckPermission(CheckPermissionRequest) returns (CheckPermissionReply);
}

message CheckPermissionRequest {
  int64 user_id = 1;
  string method = 2;
  string path = 3;
}
```

---

## 2️⃣ biz 层权限逻辑

核心思路：
启动时把权限 **缓存到内存 + Redis**。

```go
func (uc *AuthUsecase) CheckPermission(uid int64, method, path string) bool {
	perms := uc.cache.GetUserPermissions(uid)

	key := method + ":" + path
	return perms[key]
}
```

不要每次查数据库，性能会炸。

---

## 3️⃣ Gateway 加 RBAC Middleware（关键）

JWT 验证之后 → 再做权限检查。

```go
func RBACAuth(authClient authv1.AuthClient) middleware.Middleware {
	return func(handler middleware.Handler) middleware.Handler {
		return func(ctx context.Context, req any) (any, error) {

			uid := ctx.Value("uid").(int64)
			httpReq, _ := http.RequestFromServerContext(ctx)

			resp, _ := authClient.CheckPermission(ctx, &authv1.CheckPermissionRequest{
				UserId: uid,
				Method: httpReq.Method,
				Path:   httpReq.URL.Path,
			})

			if !resp.Allow {
				return nil, errors.Forbidden("NO_PERMISSION", "no permission")
			}

			return handler(ctx, req)
		}
	}
}
```

现在：

* 登录 → JWT
* 有权限 → RBAC
* 没权限 → 403

权限系统完成。

---

# 二、权限缓存刷新机制（必须有）

当管理员改权限：

```text
更新数据库 → 发布消息 → 清空缓存
```

用 MQ（Kafka/RabbitMQ）：

```go
publish("permission_changed", userID)
```

auth-service 订阅后刷新缓存。

---

# 三、多环境配置（dev / staging / prod）

这是大坑，必须从一开始设计好。

---

## 环境划分

| 环境      | 用途      |
| ------- | ------- |
| dev     | 本地开发    |
| staging | 测试 / 灰度 |
| prod    | 正式环境    |

---

## 1️⃣ 配置中心分环境路径

Consul KV 结构：

```text
config/
 ├── dev/
 │   └── user-service
 ├── staging/
 │   └── user-service
 └── prod/
     └── user-service
```

---

## 2️⃣ 服务按环境读取配置

启动时传 ENV：

```bash
ENV=dev ./server
```

加载配置：

```go
env := os.Getenv("ENV")

source, _ := consulconfig.New(
	client,
	consulconfig.WithPath("config/"+env+"/user-service"),
)
```

同一套代码跑三个环境。

---

# 三、K8s 多环境部署

命名空间隔离：

```text
k8s namespaces
├── dev
├── staging
└── prod
```

部署时指定 namespace：

```yaml
metadata:
  namespace: staging
```

---

# 四、CI/CD 按环境发布

改进 pipeline：

```yaml
on:
  push:
    branches:
      - dev
      - staging
      - main
```

部署逻辑：

```bash
dev branch → 部署 dev 环境
staging branch → 部署 staging
main → 部署 prod
```

完全自动化。

---

# 五、数据库多环境隔离

必须不同实例：

```text
mysql-dev
mysql-staging
mysql-prod
```

不要共用，否则早晚出事故。

---

# 六、域名规划（真实公司）

```text
dev-api.xxx.com
staging-api.xxx.com
api.xxx.com
```

gateway 按域名区分环境。

---

# 七、完整研发流程（最终形态）

```text
开发 → push dev → 自动部署dev
测试 → 合并staging → 自动部署staging
发布 → 合并main → 灰度 → 全量
```

---

# 八、至此整套体系闭环

你现在拥有：

### 微服务底座

* 服务注册发现
* API Gateway
* 配置中心
* 链路追踪
* Prometheus监控

### 安全体系

* JWT 登录
* RBAC 权限

### DevOps

* CI/CD 自动部署
* 灰度发布
* 多环境隔离

这已经是完整企业级微服务架构闭环。

如果后面你要做：

* 多租户 SaaS
* 分布式事务
* 消息队列架构

那就是更高阶阶段了。
