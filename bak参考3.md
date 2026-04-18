好，下面把**网关 + 服务注册 + 配置中心 + 可本地跑通**这一整套补齐。
目标：新同事 clone 就能把整套微服务跑起来。

---

# 一、整体运行架构（落地版）

```
                 ┌──────────────┐
   HTTP请求 ───► │   gateway     │
                 └──────┬───────┘
                        │ gRPC
     ┌──────────┬───────┴──────────┬──────────┐
     │          │                  │          │
 user-service order-service   pay-service  ...
     │
     └── consul 服务注册中心
```

核心组件：

* 服务注册：Consul
* 网关：grpc-gateway + kratos
* 配置中心：Consul KV
* 服务通信：gRPC

---

# 二、先启动基础设施（docker-compose）

新建仓库 **infra**

```yaml
# infra/docker-compose.yml
version: "3.9"

services:
  consul:
    image: consul:1.15
    container_name: consul
    ports:
      - "8500:8500"
    command: consul agent -dev -client=0.0.0.0

  jaeger:
    image: jaegertracing/all-in-one:1.52
    ports:
      - "16686:16686"
      - "14268:14268"

  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
```

启动：

```bash
docker compose up -d
```

打开：

* Consul UI → [http://localhost:8500](http://localhost:8500)
* Jaeger → [http://localhost:16686](http://localhost:16686)

---

# 三、服务接入 Consul（所有服务通用）

## 1️⃣ 添加依赖

```bash
go get github.com/go-kratos/kratos/contrib/registry/consul/v2
```

---

## 2️⃣ 修改 main.go（关键）

```go
import (
	"github.com/hashicorp/consul/api"
	consul "github.com/go-kratos/kratos/contrib/registry/consul/v2"
)

func newApp(bc conf.Bootstrap) *kratos.App {
	cfg := api.DefaultConfig()
	cfg.Address = "127.0.0.1:8500"

	client, _ := api.NewClient(cfg)
	reg := consul.New(client)

	grpcSrv := server.NewGRPCServer(bc)
	httpSrv := server.NewHTTPServer(bc)

	return kratos.New(
		kratos.Name("user-service"),
		kratos.Server(grpcSrv, httpSrv),
		kratos.Registrar(reg),   // 注册服务
	)
}
```

启动服务后，Consul 会自动出现：

```
user-service
```

---

# 四、服务间调用（Service Discovery）

以后**禁止写死IP**。

创建 grpc client 工厂（放 kratos-common）。

```go
// kratos-common/grpcclient/client.go
package grpcclient

import (
	"github.com/go-kratos/kratos/v2/transport/grpc"
	"github.com/go-kratos/kratos/contrib/registry/consul/v2"
	"github.com/hashicorp/consul/api"
)

func NewConn(serviceName string) (*grpc.ClientConn, error) {
	cfg := api.DefaultConfig()
	cfg.Address = "127.0.0.1:8500"
	client, _ := api.NewClient(cfg)

	r := consul.New(client)

	return grpc.DialInsecure(
		context.Background(),
		grpc.WithEndpoint("discovery:///"+serviceName),
		grpc.WithDiscovery(r),
	)
}
```

调用 user-service：

```go
conn, _ := grpcclient.NewConn("user-service")
client := userv1.NewUserClient(conn)
```

服务名就是地址。

---

# 五、Gateway 服务（重点）

新建仓库：gateway

目录：

```
gateway/
├── cmd/server
├── internal/server
└── configs
```

---

## 1️⃣ gateway main.go

```go
func newApp() *kratos.App {
	cfg := api.DefaultConfig()
	cfg.Address = "127.0.0.1:8500"
	client, _ := api.NewClient(cfg)

	reg := consul.New(client)

	httpSrv := server.NewHTTPServer()

	return kratos.New(
		kratos.Name("gateway"),
		kratos.Server(httpSrv),
		kratos.Registrar(reg),
	)
}
```

---

## 2️⃣ 创建 HTTP 网关（grpc-gateway）

```go
// internal/server/http.go
package server

import (
	userv1 "company/kratos-api/user/v1"
	"github.com/go-kratos/kratos/v2/transport/http"
)

func NewHTTPServer() *http.Server {
	conn, _ := grpcclient.NewConn("user-service")

	client := userv1.NewUserClient(conn)

	srv := http.NewServer(
		http.Address(":8000"),
	)

	userv1.RegisterUserHTTPServer(srv, client)

	return srv
}
```

现在访问：

```
http://localhost:8000/v1/user/1
```

网关 → gRPC → user-service

---

# 六、配置中心（Consul KV）

以后不要写 config.yaml。

## 放配置到 Consul

```bash
consul kv put user-service/config \
'{
  "server":{"grpc":{"addr":":9000"}},
  "data":{"database":{"dsn":"root:123456@tcp(mysql:3306)/test"}}
}'
```

---

## 服务读取配置

```go
import consulconfig "github.com/go-kratos/kratos/contrib/config/consul/v2"

source, _ := consulconfig.New(client, consulconfig.WithPath("user-service"))

c := config.New(config.WithSource(source))
```

以后改配置不用发版。

---

# 七、链路追踪（Jaeger）

所有服务加入 middleware：

```go
import "github.com/go-kratos/kratos/contrib/middleware/tracing/jaeger/v2"

tp, _ := jaeger.NewTracerProvider(
    jaeger.WithEndpoint("http://localhost:14268/api/traces"),
)
otel.SetTracerProvider(tp)
```

访问一次接口 → Jaeger 就有完整链路。

---

# 八、Prometheus 监控

kratos 已自带 metrics。

Prometheus 加配置：

```yaml
scrape_configs:
  - job_name: "kratos"
    static_configs:
      - targets: ["host.docker.internal:9000"]
```

---

# 九、本地启动顺序（新人指南）

```bash
# 1 启动基础设施
docker compose up -d

# 2 启动服务
cd user-service && make run
cd order-service && make run

# 3 启动网关
cd gateway && make run
```

访问：

```
http://localhost:8000/v1/user/1
```

整套微服务跑通。

---

# 十、到这里你已经具备

团队能力：

✔ 多人并行开发
✔ 服务独立部署
✔ 本地一键联调
✔ 服务自动发现
✔ 配置热更新
✔ 链路追踪
✔ 统一网关

这已经是**完整企业级微服务底座**。

---

下一步如果继续完善，就是：

* 登录鉴权（JWT / OAuth）
* 灰度发布
* CI/CD流水线

要不要继续补 CI/CD 和鉴权？
