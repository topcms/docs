# gorm.io/gen 最佳实践

> 适用于本项目（Kratos + Wire + gorm.io/gorm），基于现有 `internal/data` 分层编写。

---

## 1. 目录结构

引入 gen 后，推荐在 `internal/data` 下增加两个**只由工具生成、不手动修改**的子包：

```
kratos-template/
├── cmd/
│   ├── server/              # 服务入口，不含任何生成逻辑
│   └── gen/                 # ← 新增：代码生成器（开发/CI 工具，不进 server 二进制）
│       └── main.go
├── internal/
│   └── data/
│       ├── model/           # ← 新增（gen 生成）：GORM Model struct，禁止手改
│       ├── query/           # ← 新增（gen 生成）：类型安全 DAO，禁止手改
│       ├── data.go          # 现有：*gorm.DB 初始化，Wire Provider，新增 query.Use(db)
│       └── user.go          # 现有：UserRepo 实现，将 findLocalFromDB 改为调用 query 包
├── docs/
│   └── gorm-gen-best-practice.md
└── go.mod
```

**核心原则**
- `model/` 和 `query/` 里的文件全部由 `cmd/gen` 生成，**任何时候不手动编辑**。
- 需要定制字段类型/标签，在生成器配置中通过 `gen.FieldType`、`gen.FieldRename` 等选项处理。
- 自定义 SQL 查询通过生成器里定义 **接口注释** 的方式注入，而不是在生成代码里手写。

---

## 2. 添加依赖

```bash
go get gorm.io/gen@latest
```

确保 `go.mod` 中 `gorm.io/gen` 与 `gorm.io/gorm` 大版本一致（当前项目已锁定 `gorm.io/gorm v1.31.x`）。

---

## 3. 生成器：`cmd/gen/main.go`

```go
package main

import (
    "gorm.io/driver/mysql"
    "gorm.io/gen"
    "gorm.io/gorm"
)

// UserQuerier 为 users 表追加自定义查询（SQL 写在注释里，gen 自动生成实现）。
type UserQuerier interface {
    // SELECT id, name, avatar FROM users WHERE name LIKE concat('%',@name,'%') LIMIT @limit
    SearchByName(name string, limit int) ([]*gen.T, error)
}

func main() {
    // 仅在生成阶段连接数据库，DSN 可从环境变量读取，不硬编码到版本库。
    dsn := "root:root@tcp(127.0.0.1:3306)/kratos_demo?charset=utf8mb4&parseTime=True&loc=Local"
    db, err := gorm.Open(mysql.Open(dsn))
    if err != nil {
        panic(err)
    }

    g := gen.NewGenerator(gen.Config{
        OutPath:           "internal/data/query",  // DAO 输出目录（相对项目根）
        ModelPkgPath:      "internal/data/model",  // Model 输出目录
        Mode:              gen.WithDefaultQuery | gen.WithQueryInterface,
        FieldNullable:     true,   // 可为 NULL 的字段映射为指针，避免零值歧义
        FieldCoverable:    false,
        FieldSignable:     false,
        FieldWithIndexTag: false,
        FieldWithTypeTag:  true,
    })

    g.UseDB(db)

    // 只生成指定的表，避免误生成无关表
    userTable := g.GenerateModel("users")

    // 为 users 表挂载自定义查询接口
    g.ApplyInterface(func(UserQuerier) {}, userTable)

    // 其余表直接 ApplyBasic（无自定义方法）
    // g.ApplyBasic(g.GenerateModel("orders"))

    g.Execute()
}
```

**运行方式**（在项目根目录执行）：

```bash
go run ./cmd/gen
```

也可以在 `data.go` 或 `Makefile` 里加 `//go:generate go run ./cmd/gen`，统一用 `go generate ./...` 触发。

---

## 4. 更新 `internal/data/data.go`

在现有 `NewData` 里，用生成的 `query.Use(db)` 将 `*gorm.DB` 绑定到 gen Query，
然后把 `*query.Query` 存进 `Data` 结构体，供各 Repo 使用。

```go
package data

import (
    "github.com/topcms/kratos-template/internal/conf"
    "github.com/topcms/kratos-template/internal/data/query" // ← gen 生成

    infraMySQL "github.com/topcms/kratos-infra/mysql"
    infraRedis "github.com/topcms/kratos-infra/redis"

    "github.com/go-kratos/kratos/v2/log"
    "github.com/google/wire"

    goredis "github.com/redis/go-redis/v9"
    "gorm.io/gorm"
)

var ProviderSet = wire.NewSet(NewData, NewUserRepo)

type Data struct {
    db    *gorm.DB
    redis *goredis.Client
    q     *query.Query // ← 新增：gen 生成的类型安全 Query
}

func NewData(c *conf.Data) (*Data, func(), error) {
    db, err := infraMySQL.NewDB(infraMySQL.Config{
        DSN:             c.Database.Source,
        MaxIdleConns:    c.Database.MaxIdleConns,
        MaxOpenConns:    c.Database.MaxOpenConns,
        ConnMaxLifetime: c.Database.ConnMaxLifetime,
    })
    if err != nil {
        return nil, nil, err
    }

    rd := infraRedis.NewClient(infraRedis.Config{
        Addr:         c.Redis.Addr,
        Password:     c.Redis.Password,
        DB:           c.Redis.DB,
        DialTimeout:  c.Redis.DialTimeout,
        ReadTimeout:  c.Redis.ReadTimeout,
        WriteTimeout: c.Redis.WriteTimeout,
    })

    // 迁移：生产环境建议换成 golang-migrate/goose，这里保留 demo 用的 AutoMigrate
    if err := db.AutoMigrate(&userModel{}); err != nil {
        return nil, nil, err
    }

    d := &Data{
        db:    db,
        redis: rd,
        q:     query.Use(db), // ← 绑定 gen Query，不重新建连接
    }

    cleanup := func() {
        log.Info("closing the data resources")
        if sqlDB, err := d.db.DB(); err == nil {
            _ = sqlDB.Close()
        }
        _ = d.redis.Close()
    }
    return d, cleanup, nil
}
```

> `*gorm.DB` 仍由 `kratos-infra/mysql.NewDB` 统一创建，gen 只是消费同一个连接池，不额外建连接。

---

## 5. 更新 `internal/data/user.go`

将 `findLocalFromDB` 从手写链式 API 换成 gen 生成的类型安全调用：

```go
func (r *userRepo) findLocalFromDB(ctx context.Context, id int64) (*biz.User, error) {
    u := r.data.q.User // gen 生成的 UserDo，字段名全部强类型
    result, err := u.WithContext(ctx).Where(u.ID.Eq(id)).First()
    if err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, nil
        }
        return nil, err
    }
    return &biz.User{
        ID:     result.ID,
        Name:   result.Name,
        Avatar: result.Avatar,
    }, nil
}
```

**其余部分（Redis 缓存、远程 discovery 拉取）完全不变**，gen 只替换了最底层的 DB 查询。

同时，`userModel` struct 可以删除，改为直接引用 gen 生成的 `model.Users`：

```go
// 删除手写的 userModel，改用 gen 生成的 model（迁移期间可以共存）
import "github.com/topcms/kratos-template/internal/data/model"

// AutoMigrate 也改为：
db.AutoMigrate(&model.Users{})
```

---

## 6. 分层边界（不变）

```
HTTP/gRPC Handler
      ↓
   Service（biz）      ← 只依赖 biz.UserRepo 接口，看不到任何 DB/gen 类型
      ↓
  biz.UserRepo 接口
      ↓
  data.userRepo        ← 唯一调用 query.Xxx / model.Xxx 的地方
      ↓
  gen 生成的 query.*   ← 不手改，由 cmd/gen 维护
      ↓
   *gorm.DB            ← 由 kratos-infra/mysql 统一管理
```

`query` 和 `model` 包**绝不能**被 `biz`、`service`、`server` 层直接引用。

---

## 7. 表结构迁移策略

gen **不负责表结构变更**，只负责生成 Go 代码。推荐流程：

```
修改迁移脚本（SQL 文件 / golang-migrate / goose）
           ↓
运行迁移工具，将变更应用到数据库
           ↓
go run ./cmd/gen    （重新生成 model/ 和 query/，同步新字段）
           ↓
提交生成代码 diff（Code Review 可见改动）
```

> 当前项目用 `db.AutoMigrate` 做演示，生产环境建议改为独立迁移工具，
> 保证「迁移脚本 → gen → 业务代码」的单向依赖。

---

## 8. CI 校验：防止「忘了跑 gen」

在 CI Pipeline 中加一步校验，确保生成代码与数据库/配置始终同步：

```yaml
# .github/workflows/ci.yml（示例）
- name: Verify gen is up to date
  env:
    GEN_DSN: ${{ secrets.GEN_DSN }}
  run: |
    go run ./cmd/gen
    git diff --exit-code internal/data/model internal/data/query
```

若生成代码与提交不一致，流水线报错，强制开发者重新 gen 后再提交。

---

## 9. 注意事项

| 事项 | 说明 |
|------|------|
| **不手改生成代码** | 下次跑 gen 会覆盖；自定义逻辑通过接口注释注入 |
| **生成器不进 server 二进制** | `cmd/gen` 是纯开发工具，`go build ./cmd/server` 不会编译它 |
| **同一表只有一套 struct** | 迁移完成后删掉手写 `userModel`，避免两套定义并存 |
| **nullable 字段用指针** | 开启 `FieldNullable: true`，避免 `""` / `0` 与 NULL 混淆 |
| **渐进迁移** | 新表直接用 gen；旧表可保留手写，逐步替换，不必一次性全改 |
| **DSN 不硬编码** | 生成器从环境变量读取 DSN，不提交到版本库 |
