下面给你一份 **可直接开干的 go-kratos 微服务团队模板**。
这套结构是为 **多人并行开发**设计的。

---

# 一、公司级仓库布局（最终形态）

```text
company/
│
├── kratos-api          # 所有 proto 契约
├── kratos-common       # 公共基础库
├── kratos-template     # 服务脚手架模板
│
├── gateway             # API网关
├── user-service
├── order-service
└── pay-service
```

先重点给你 **最核心的 kratos-template**。

---

# 二、服务模板目录结构（重点）

```text
user-service/
│
├── api/                 # 从 kratos-api 拉取的 proto 生成代码
│
├── cmd/
│   └── server/
│       └── main.go
│
├── internal/
│   ├── biz/             # 业务逻辑层（纯业务）
│   ├── data/            # 数据访问层（mysql/redis/grpc）
│   ├── service/         # gRPC/HTTP handler
│   ├── conf/            # 配置结构体
│   ├── server/          # grpc/http server 启动
│   └── pkg/             # 当前服务私有工具
│
├── configs/
│   ├── config.yaml
│   └── config.local.yaml
│
├── deployments/
│   ├── Dockerfile
│   └── k8s.yaml
│
├── third_party/
├── Makefile
├── go.mod
└── README.md
```

这就是每个服务统一结构。

---

# 三、main.go（统一启动入口）

```go
package main

import (
	"flag"
	"os"

	"user-service/internal/conf"
	"user-service/internal/server"

	"github.com/go-kratos/kratos/v2"
	"github.com/go-kratos/kratos/v2/config"
	"github.com/go-kratos/kratos/v2/config/file"
)

var flagconf string

func init() {
	flag.StringVar(&flagconf, "conf", "../../configs", "config path")
}

func main() {
	flag.Parse()

	c := config.New(
		config.WithSource(file.NewSource(flagconf)),
	)
	defer c.Close()

	if err := c.Load(); err != nil {
		panic(err)
	}

	var bc conf.Bootstrap
	if err := c.Scan(&bc); err != nil {
		panic(err)
	}

	app := newApp(bc)
	if err := app.Run(); err != nil {
		panic(err)
	}
}

func newApp(bc conf.Bootstrap) *kratos.App {
	grpcSrv := server.NewGRPCServer(bc)
	httpSrv := server.NewHTTPServer(bc)

	return kratos.New(
		kratos.Name("user-service"),
		kratos.Server(grpcSrv, httpSrv),
	)
}
```

---

# 四、service 层（只做接口适配）

```go
// internal/service/user.go
package service

import (
	pb "company/kratos-api/user/v1"
	"context"
	"user-service/internal/biz"
)

type UserService struct {
	pb.UnimplementedUserServer
	uc *biz.UserUsecase
}

func NewUserService(uc *biz.UserUsecase) *UserService {
	return &UserService{uc: uc}
}

func (s *UserService) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.GetUserReply, error) {
	user, err := s.uc.GetUser(ctx, req.Id)
	if err != nil {
		return nil, err
	}
	return &pb.GetUserReply{
		Id:   user.ID,
		Name: user.Name,
	}, nil
}
```

👉 **原则：service 层不能写业务逻辑**

---

# 五、biz 层（真正业务）

```go
// internal/biz/user.go
package biz

import "context"

type User struct {
	ID   int64
	Name string
}

type UserRepo interface {
	FindByID(ctx context.Context, id int64) (*User, error)
}

type UserUsecase struct {
	repo UserRepo
}

func NewUserUsecase(repo UserRepo) *UserUsecase {
	return &UserUsecase{repo: repo}
}

func (uc *UserUsecase) GetUser(ctx context.Context, id int64) (*User, error) {
	return uc.repo.FindByID(ctx, id)
}
```

👉 **业务不依赖框架**

---

# 六、data 层（实现接口）

```go
// internal/data/user.go
package data

import (
	"context"
	"user-service/internal/biz"
)

type userRepo struct {}

func NewUserRepo() biz.UserRepo {
	return &userRepo{}
}

func (r *userRepo) FindByID(ctx context.Context, id int64) (*biz.User, error) {
	return &biz.User{
		ID:   id,
		Name: "test",
	}, nil
}
```

---

# 七、配置结构体（统一）

```go
// internal/conf/conf.go
package conf

type Bootstrap struct {
	Server Server `yaml:"server"`
	Data   Data   `yaml:"data"`
}

type Server struct {
	Grpc struct {
		Addr string `yaml:"addr"`
	} `yaml:"grpc"`
}

type Data struct {
	Database struct {
		Dsn string `yaml:"dsn"`
	} `yaml:"database"`
}
```

---

# 八、Makefile（非常关键）

```makefile
.PHONY: init proto build run

init:
	go mod tidy

proto:
	buf generate

build:
	go build -o ./bin/server ./cmd/server

run:
	go run ./cmd/server -conf ./configs
```

---

# 九、Dockerfile（统一）

```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN go build -o server ./cmd/server

FROM debian:bookworm
WORKDIR /app
COPY --from=builder /app/server .
COPY configs ./configs
CMD ["./server", "-conf", "./configs"]
```

---

# 十、kratos-api 仓库结构（接口仓库）

```text
kratos-api/
│
├── user/v1/user.proto
├── order/v1/order.proto
└── buf.gen.yaml
```

proto 示例：

```proto
syntax = "proto3";

package user.v1;

service User {
  rpc GetUser (GetUserRequest) returns (GetUserReply);
}

message GetUserRequest {
  int64 id = 1;
}

message GetUserReply {
  int64 id = 1;
  string name = 2;
}
```

---

# 十一、common 仓库建议内容

```text
kratos-common/
├── logger
├── redis
├── mysql
├── middleware
├── errors
└── tracing
```

所有服务统一引用。

---

# 十二、真正的开发流程

新增服务：

```bash
kratos new pay-service -r kratos-template
```

开发新接口：

1. 改 kratos-api proto
2. 打 tag v1.0.1
3. 服务更新依赖
4. 各自开发
5. 各自上线

没人互相等待。

---

如果你下一步要做网关（gateway）或服务注册(consul)，我可以把那一套也给你补齐。
