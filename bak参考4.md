好，最后把 **CI/CD + 鉴权体系（JWT）+ 灰度发布** 一次补齐。
这三块是微服务真正上线必备的最后拼图。

---

# 一、统一登录鉴权体系（JWT）

微服务必须 **只做业务，不做登录**。
登录统一放在 **gateway + auth-service**。

---

## 架构

```text
客户端 → gateway → auth-service → 颁发JWT
                   ↓
            所有微服务验证JWT
```

---

## 1️⃣ 新建 auth-service

```text
auth-service/
├── internal/biz/jwt.go
├── internal/service/auth.go
└── api/auth/v1/auth.proto
```

---

## 2️⃣ JWT 生成

```go
// internal/biz/jwt.go
package biz

import (
	"time"
	"github.com/golang-jwt/jwt/v5"
)

var secret = []byte("very-secret-key")

type Claims struct {
	UserID int64 `json:"uid"`
	jwt.RegisteredClaims
}

func GenerateToken(uid int64) (string, error) {
	claims := Claims{
		UserID: uid,
		RegisteredClaims: jwt.RegisteredClaims{
			ExpiresAt: jwt.NewNumericDate(time.Now().Add(7*24*time.Hour)),
		},
	}
	return jwt.NewWithClaims(jwt.SigningMethodHS256, claims).SignedString(secret)
}
```

---

## 3️⃣ 登录接口

```go
func (s *AuthService) Login(ctx context.Context, req *pb.LoginRequest) (*pb.LoginReply, error) {
	token, _ := biz.GenerateToken(req.UserId)
	return &pb.LoginReply{Token: token}, nil
}
```

---

# 二、Gateway 全局鉴权 Middleware（关键）

所有请求先经过 gateway 验证 JWT。

```go
// gateway/internal/middleware/auth.go
func JWTAuth() middleware.Middleware {
	return func(handler middleware.Handler) middleware.Handler {
		return func(ctx context.Context, req any) (any, error) {
			token := extractToken(ctx)
			if token == "" {
				return nil, errors.Unauthorized("NO_TOKEN", "no token")
			}

			claims := &biz.Claims{}
			_, err := jwt.ParseWithClaims(token, claims,
				func(t *jwt.Token) (interface{}, error) {
					return []byte("very-secret-key"), nil
				})
			if err != nil {
				return nil, errors.Unauthorized("BAD_TOKEN", "bad token")
			}

			ctx = context.WithValue(ctx, "uid", claims.UserID)
			return handler(ctx, req)
		}
	}
}
```

挂到 HTTP Server：

```go
http.NewServer(
	http.Middleware(
		JWTAuth(),
	),
)
```

现在：

* 未登录 → 直接拦截
* 已登录 → 自动把 uid 注入上下文

---

## 3️⃣ 业务服务获取用户ID

```go
uid := ctx.Value("uid").(int64)
```

所有服务共享登录态。

---

# 三、CI/CD（GitHub Actions 示例）

每个服务 **独立流水线**。

放在每个仓库：

```yaml
# .github/workflows/deploy.yml
name: build-and-deploy

on:
  push:
    branches: [ "main" ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Build Docker
        run: docker build -t user-service .

      - name: Push Image
        run: |
          docker tag user-service registry/user-service:${{ github.sha }}
          docker push registry/user-service:${{ github.sha }}

      - name: Deploy
        run: |
          kubectl set image deployment/user-service \
          user-service=registry/user-service:${{ github.sha }}
```

流程：

```text
git push → 自动打镜像 → 自动部署 → 完全无人值守
```

---

# 四、版本号规范（必须）

所有服务统一：

```text
v1.2.3

主版本：破坏性改动
次版本：新增功能
修订号：修复bug
```

镜像标签：

```text
user-service:v1.4.2
```

禁止使用 latest。

---

# 五、灰度发布（真正企业必备）

K8s Deployment 改成 **Rolling Update**

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

发布流程：

```text
旧版本 3 pod
新版本 1 pod → 2 → 3
```

用户无感。

---

## Canary 灰度（进阶）

通过 gateway 按 header 分流：

```go
if req.Header.Get("X-Canary") == "true" {
    endpoint = "discovery:///user-service-canary"
}
```

实现：

* 内部员工先用新版本
* 正常用户用旧版本

非常实用。

---

# 六、完整发布流程（公司级）

开发完成：

```text
1. 提交代码
2. CI自动测试
3. 自动构建镜像
4. 自动部署K8s
5. 灰度发布
6. 全量发布
```

无需人工登录服务器。

---

# 七、最终你拥有的整套体系

现在这套包含：

### 微服务底座

* 服务注册发现（Consul）
* API 网关
* 配置中心
* 链路追踪
* Prometheus监控

### 安全

* JWT统一登录
* Gateway全局鉴权

### DevOps

* 自动CI/CD
* Docker镜像发布
* Kubernetes部署
* 灰度发布

这已经是完整企业级微服务架构。

---

如果你下一步要做 **权限系统（RBAC）** 或 **多环境(dev/staging/prod)**，那就是最后两块了。
