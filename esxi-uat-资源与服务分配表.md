# ESXi UAT 资源与服务分配表

> 源文档：[`esxi-uat-重规划方案.md`](./esxi-uat-重规划方案.md)
>
> 本文档将重规划方案中的资源规格（§6）、网络分配（§9.1）、启动顺序（§7）、每台 VM 部署的服务清单（§4.1 ~ §4.4）汇总为一张速查表，便于在 ESXi / 部署阶段直接对照执行。

---

## 1. 总览：四台 UAT VM 一览

| VM 名称 | 角色定位 | 稳定性等级 | vCPU | 内存 | 内存预留 | CPU 预留 | CPU 限制 | CPU 份额 | 启动策略 | 固定 IP | 启动延时 |
|---|---|---|---:|---:|---:|---:|---:|---|---|---|---:|
| `core-data` | 核心数据层（最稳） | T2（最高优先） | 2 | 6 GB | 4 GB | 500 MHz | 不限制 | 高 | 自动 | `192.168.10.21` | 60s |
| `app-svc` | 对外/对内应用层 | T2 | 2 | 5 GB | 3 GB | 300 MHz | 3000 MHz | 正常 | 自动 | `192.168.10.22` | 90s |
| `devops` | 工具 + 日志 | T2（低优先） | 2 | 3 GB | 1 GB | 0 | 2000 MHz | 低 | 自动 | `192.168.10.23` | 150s |
| `lab` | 学习实验（手动） | T3 | 2 | 6 GB | 0 | 0 | 2500 MHz | 低 | **手动** | `192.168.10.24` | — |
| **常驻合计** | — | — | **6**（共享 4 物理核） | **14 GB** | **8 GB** | — | — | — | — | — | — |
| **含 lab 合计** | — | — | **8**（共享 4 物理核） | **20 GB** | **8 GB** | — | — | — | — | — | — |

**资源边界提示**：

- 宿主：Intel N5105（4 逻辑 CPU）+ 32 GB 内存。
- ESXi 自身约 2 GB；核心层（openwrt + win2016 + jumpserver）约 7 GB。
- UAT 可用预算 ≈ 20 GB 常驻 + 爆发；常驻 4 台 14 GB 已留出 ≥ 3 GB 余量给 lab 临时启动。

---

## 2. 每台 VM 部署服务清单

### 2.1 `core-data`（核心数据层，192.168.10.21）

> 原则：只放"丢了会哭"的服务，一年重启次数应 ≤ 宿主重启次数。

| 服务 | 镜像/版本建议 | 端口 | 数据卷 | 说明 |
|---|---|---|---|---|
| MySQL | `mysql:8.x` | 3306（仅内网） | `/data/mysql` → `/var/lib/mysql` | mindoc / gitea / spug / 业务正式库统一落此；开 binlog（`log_bin` / `server_id`） |
| MinIO | `minio/minio` | 9000 / 9001 | `/data/minio` → `/data` | 对象存储常用 |
| Redis（业务） | `redis:7` | 6379（仅内网） | `/data/redis` | 业务唯一 Redis 实例，低内存配置，AOF 落盘 |
| 备份脚本 | crontab 调度 | — | `/data/backup/{mysql,minio,redis}` | `mysqldump` + `mc mirror` + `BGSAVE`，详见源文档 §10 |
| Promtail（agent） | `grafana/promtail` | — | 容器日志/系统日志 | 日志上报至 `devops` 的 Loki |

**禁止部署**：Postgres、Mongo、Kafka、ES、RabbitMQ、etcd、测试 MySQL、测试 Redis（全部归 `lab`）。

**目录布局**：

```text
/data/
  ├── mysql/           # 容器挂载 -> /var/lib/mysql
  ├── minio/           # 容器挂载 -> /data
  ├── redis/           # AOF 落盘
  └── backup/
       ├── mysql/YYYYMMDD/*.sql.gz
       └── minio/mirror/
/opt/stacks/core-data/
  └── docker-compose.yml
```

---

### 2.2 `app-svc`（对外 / 对内应用层，192.168.10.22）

> 原则：无状态或轻状态应用 + 统一反代入口；DB 全部连 `core-data`。

| 服务 | 镜像/版本建议 | 端口 | 依赖 | 说明 |
|---|---|---|---|---|
| Caddy（推荐）/ nginx | `caddy:2` | 80 / 443 | — | 统一反代 + 自动 HTTPS（Caddy 内置 ACME） |
| vaultwarden | `vaultwarden/server` | 80（容器内）→ 反代 | Caddy | 对外密码服务 |
| mindoc | `daocloud.io/mindoc/mindoc` | 8181 → 反代 | core-data MySQL | 文档平台，DB 指向 `192.168.10.21:3306` |
| gitea | `gitea/gitea` | 3000 → 反代 | core-data MySQL | Git 托管，DB 指向 `192.168.10.21:3306` |
| spug | `openspug/spug` | 80 → 反代 | core-data MySQL | 发布平台，DB 指向 `192.168.10.21:3306` |
| certd | `certd/certd` | — | Caddy/nginx | 证书自动签发（用 Caddy 时可省） |
| Promtail（agent） | `grafana/promtail` | — | devops Loki | 日志上报 |

**砍掉**：PHP-FPM（未使用）、本机 nginx 冗余实例、本机 MySQL/Redis（统一并入 core-data）。

**对外入口约定**：openwrt 仅把 `80 / 443` 转发到本机 Caddy；其它 VM 不直接暴露公网。

---

### 2.3 `devops`（工具 + 日志，192.168.10.23）

> 原则：常驻但低优先级，允许被挤压。

| 服务 | 镜像/版本建议 | 端口 | 必需 | 说明 |
|---|---|---|---|---|
| Loki | `grafana/loki` | 3100 | 必需 | 日志聚合 |
| Grafana | `grafana/grafana` | 3000 | 必需 | 日志/指标可视化 |
| Promtail（server 端） | `grafana/promtail` | 9080 | 必需 | 自身/汇聚 |
| Gitea Actions Runner | `gitea/act_runner` | — | 推荐 | 替代 Jenkins，对接 `app-svc` 的 gitea |
| Uptime-Kuma | `louislam/uptime-kuma` | 3001 | 可选 | 对外服务可用性监控 |
| Prometheus | `prom/prometheus` | 9090 | 可选 | 指标监控扩展 |
| node_exporter / cAdvisor | `prom/node-exporter` / `gcr.io/cadvisor/cadvisor` | 9100 / 8080 | 可选 | 节点 / 容器指标 |

**替换关系**：

- `Logstash → Promtail / Vector`，省约 1 GB 内存。
- `Jenkins → Gitea Actions Runner`，省约 1.5 GB 内存。

---

### 2.4 `lab`（学习实验，192.168.10.24，手动启停）

> 所有"偶尔学一下"的重量级组件集中在这里，靠 `docker compose --profile` 按需拉起。

| Profile | 组件 | 镜像 | 端口约定 | 说明 |
|---|---|---|---|---|
| `pg` | Postgres + pgAdmin | `postgres:15` / `dpage/pgadmin4` | 5432 / 5050 | 学习用 |
| `mongo` | MongoDB + mongo-express | `mongo:7.0` / `mongo-express` | 27017 / 8081 | 升级到 7.0（4.4 已 EOL） |
| `mq` | RabbitMQ | `rabbitmq:management` | 5672 / 15672 | 按需 |
| `kafka` | Kafka KRaft + Kafka UI | `bitnami/kafka:3.7` / `provectuslabs/kafka-ui` | 9092 / 8082 | KRaft 单节点免 ZK |
| `es` | Elasticsearch + Kibana | `elasticsearch:7.17.13` / `kibana:7.17.13` | 9200 / 5601 | 学习用 |
| `etcd` | etcd 单节点 + etcdkeeper | `bitnami/etcd` / `evildecay/etcdkeeper` | 2379 / 8083 | 按需 |
| `php` | php-fpm + nginx 测试栈 | `php:fpm` / `nginx` | 8084 | 临时调试 |
| `jenkins` | Jenkins LTS | `jenkins/jenkins:lts` | 8085 | 真要学 Jenkins 时启动 |
| `mysql-test` | MySQL 测试库 | `mysql:8` | 3307 | 与正式库隔离 |
| `redis-test` | Redis 测试实例 | `redis:7` | 6380 | 与正式 Redis 隔离 |
| Promtail（agent） | — | `grafana/promtail` | — | 日志上报至 devops Loki |

**禁止部署**：任何对外服务、任何正式持久化数据。

**使用方式**：

```bash
docker compose --profile kafka --profile es up -d   # 只启 kafka + es
docker compose --profile kafka down                 # 用完即关
```

---

## 3. 服务到 VM 的归位速查

| 服务/组件 | 归属 VM | 是否常驻 |
|---|---|:---:|
| MySQL（正式） | core-data | 常驻 |
| MinIO | core-data | 常驻 |
| Redis（业务） | core-data | 常驻 |
| Caddy / nginx | app-svc | 常驻 |
| vaultwarden | app-svc | 常驻 |
| mindoc | app-svc | 常驻 |
| gitea | app-svc | 常驻 |
| spug | app-svc | 常驻 |
| certd | app-svc | 常驻 |
| Loki + Grafana + Promtail | devops | 常驻 |
| Gitea Actions Runner | devops | 常驻（推荐） |
| Uptime-Kuma / Prometheus | devops | 可选 |
| Postgres / MongoDB / Kafka / ES / RabbitMQ / etcd | lab | 按需 |
| Jenkins / PHP-FPM | lab | 按需 |
| MySQL 测试库 / Redis 测试实例 | lab | 按需 |

---

## 4. 启动顺序与网络

### 4.1 启动顺序（含核心层）

```text
openwrt      → 0s     # 网关 T0
win2016      → 15s
jumpserver   → 30s
core-data    → 60s    # 数据先就绪
app-svc      → 90s    # 应用起来连 DB
devops       → 150s   # 工具最后
lab          → 手动
```

### 4.2 网段与对外暴露

| VM | 固定 IP | 是否对外暴露 | 暴露方式 |
|---|---|:---:|---|
| openwrt | — | 是 | WAN 入口，仅转发 80/443 → app-svc |
| core-data | `192.168.10.21` | 否 | 仅内网（DB/Redis/MinIO 端口仅监听内网） |
| app-svc | `192.168.10.22` | 是 | 通过 openwrt 接收 80/443 |
| devops | `192.168.10.23` | 否 | 仅内网 |
| lab | `192.168.10.24` | 否 | 仅内网 |

---

## 5. 验收快速对照

| 验收项 | 期望值 |
|---|---|
| UAT 常驻 4 台总内存使用 | ≤ 14 GB（不含 lab） |
| lab 关闭时宿主可用内存 | ≥ 3 GB |
| `core-data` 容器类型 | 仅 MySQL / MinIO / Redis 三类 |
| 对外服务统一入口 | 全部经 `app-svc` 反代 |
| Grafana 检索任意时段日志 | < 5 秒 |
| 从备份恢复 MySQL 到可用 | < 30 分钟 |
| 单台 UAT VM 强关 | 其他三台不受影响（core-data 不依赖任何其它 UAT） |

---

## 附：与源文档章节对照

| 本文表格 | 源文档章节 |
|---|---|
| §1 总览（资源规格） | `esxi-uat-重规划方案.md` §6 |
| §1 固定 IP 列 | §9.1 |
| §1 启动延时列 / §4.1 启动顺序 | §7 |
| §2.1 core-data | §4.1 |
| §2.2 app-svc | §4.2 |
| §2.3 devops | §4.3 |
| §2.4 lab | §4.4 |
| §3 服务归位速查 | §4 + §5（迁移映射） |
| §4.2 网段与对外暴露 | §9.1 + §9.2 |
| §5 验收 | §14 |
