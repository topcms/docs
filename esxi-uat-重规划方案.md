# ESXi UAT 虚拟机重规划方案

> 配套文档：[`esxi-资源优化与救援操作手册.md`](./esxi-资源优化与救援操作手册.md)
>
> 本文档专注于 `data_os` / `pro_uat` / `tool_uat` / `ts_uat` 四台 UAT 机的**全新重做方案**，不考虑外部迁移与历史兼容；`openwrt` / `win2016` / `jumpserver` 三台核心 VM 的策略沿用主手册。

---

## 1. 背景

- 宿主：Intel N5105（4 逻辑 CPU）+ 32GB 内存 + 单盘 ESXi。
- 当前 4 台 UAT VM 按"历史命名"组合服务，存在如下结构性问题：
  - 正式数据（MinIO、业务 MySQL、业务 Redis）与学习组件（Kafka/ZK/ES/PG/Mongo）混跑，学习负载会拖累正式数据。
  - 对外服务（vaultwarden）与 PHP 测试环境混居，职能冲突。
  - Logstash、Jenkins、PHP-FPM 等"空跑占坑"但几乎不用。
  - 多份 Redis 重复常驻，浪费内存。
- 用户明确：`data_os` / `pro_uat` / `tool_uat` / `ts_uat` 四台可打散重做，按全新开始设计。

## 2. 设计原则

1. **稳定性分层**：正式数据 > 对外服务 > 内部工具 > 学习实验，分层对应不同 VM，故障互不影响。
2. **存储与应用解耦**：应用层允许随时重启、升级、崩溃；数据层尽量不动。
3. **按"稳定性 + 使用强度"抽存储**，而不是"一库一 VM"：
   - 正式长期使用 → 独立核心 VM。
   - 学习/偶尔使用 → 集中到实验 VM，按需启停。
4. **学习组件集中隔离 + 手动启停**：重量级中间件不常驻，用 `docker compose --profile` 按需拉起。
5. **资源减法优先**：用不上的服务直接砍，能换轻量等价物就换（Logstash→Promtail、Jenkins→Gitea Actions）。
6. **命名按职能**：`core-data` / `app-svc` / `devops` / `lab`，后续迁移扩容无歧义。
7. **所有服务容器化**：统一用 Docker + Compose 管理，VM 层只负责资源隔离与系统更新。

## 3. 现状诊断

| 现有 VM | 主要服务 | 问题定性 |
|---|---|---|
| `data_os` | minio、postgres:15、redis、rabbitmq、mongo:4.4.25、kafka+zk、elasticsearch:7.17.13 | 正式存储（MinIO、业务 Redis）与学习重量级中间件混跑；ES/Kafka 一吃内存整台抖。 |
| `pro_uat` | mindoc、gitea、mysql（正式）、etcd、redis、nginx、php | 正式数据与应用、PHP 环境耦合；应用崩会牵连正式 DB。 |
| `ts_uat` | vaultwarden、certd、mysql、redis、nginx、php | 对外生产服务（vaultwarden/certd）与随意测试环境混居，稳定性等级冲突。 |
| `tool_uat` | spug、jenkins、loki、logstash | Logstash 吃内存但未实际用；Jenkins 空跑；日志栈未形成闭环。 |

一句话：**正式数据被学习负载拖累、对外服务被测试污染、低频工具常驻占坑、多份 Redis 冗余。**

## 4. 新 4 VM 架构

```text
                    ┌─────────────────────────────────────┐
                    │         openwrt (网关 T0)           │
                    └──────────────┬──────────────────────┘
                                   │
         ┌──────────────┬──────────┴──────────┬──────────────┐
         ▼              ▼                     ▼              ▼
   ┌──────────┐  ┌──────────────┐      ┌──────────────┐  ┌──────────┐
   │ app-svc  │  │  core-data   │      │   devops     │  │   lab    │
   │ 对外/应用 │  │ 正式数据层   │      │  工具/日志   │  │ 学习实验 │
   │ (常驻)   │  │ (常驻,最稳)  │      │ (常驻,低优)  │  │ (手动)   │
   └──────────┘  └──────────────┘      └──────────────┘  └──────────┘
```

### 4.1 `core-data`（核心数据层）

**原则：只放"丢了会哭"的服务，一年重启次数应 ≤ 宿主重启次数。**

| 服务 | 必要性 | 说明 |
|---|---|---|
| MySQL 8.x | 必须 | mindoc / gitea / spug / 业务正式库统一落此 |
| MinIO | 必须 | 对象存储常用 |
| Redis（业务单实例） | 必须 | 统一的业务 Redis，低内存配置 |
| 备份脚本 | 必须 | `mysqldump` + `mc mirror` + crontab，详见 §10 |

**禁止**：Postgres / Mongo / Kafka / ES / RabbitMQ / etcd / 测试 MySQL / 测试 Redis（全部属于 `lab`）。

建议目录布局（VM 内）：

```text
/data/
  ├── mysql/               # 容器挂载 -> /var/lib/mysql
  ├── minio/               # 容器挂载 -> /data
  ├── redis/               # AOF 落盘
  └── backup/
       ├── mysql/YYYYMMDD/*.sql.gz
       └── minio/mirror/
/opt/stacks/core-data/
  └── docker-compose.yml   # 统一管理
```

### 4.2 `app-svc`（对外 / 对内应用层）

无状态或轻状态应用 + 统一反代入口。

| 服务 | 说明 |
|---|---|
| Caddy（或保留 nginx） | 统一反代 + 自动 HTTPS；对接 certd |
| vaultwarden | 对外密码服务 |
| mindoc | 文档平台，DB 连 `core-data` |
| gitea | Git 托管，DB 连 `core-data` |
| spug | 发布平台，DB 连 `core-data` |
| certd | 证书自动签发 |

**砍掉**：PHP-FPM（未使用）、本机 nginx 冗余实例、本机 MySQL/Redis（并入 core-data）。

> Caddy 推荐理由：一行 `example.com { reverse_proxy vaultwarden:80 }` 自动出 HTTPS，少一层 certd 对接。保留 nginx + certd 的组合也可接受。

### 4.3 `devops`（工具 + 日志）

常驻但低优先级，允许被挤压。

| 服务 | 说明 |
|---|---|
| Loki + Promtail + Grafana | 实际使用的日志栈 |
| Promtail agent | 部署在 `core-data` / `app-svc` / `lab` 采集容器日志 |
| Gitea Actions Runner | 替代 Jenkins，与 gitea 天然集成 |
| Uptime-Kuma（可选） | 对外服务可用性监控 |
| Prometheus + node_exporter（可选，§11） | 指标监控扩展 |

**替换**：
- `Logstash → Promtail / Vector`，省约 1G 内存。
- `Jenkins → Gitea Actions Runner`，省约 1.5G 内存；真想学 Jenkins 丢 `lab`。

### 4.4 `lab`（学习实验，手动启停）

所有"偶尔学一下"的重量级组件集中在这里，靠 `docker compose --profile` 按需拉。

| Profile | 组件 | 端口约定（建议） |
|---|---|---|
| `pg` | Postgres 15 + pgAdmin | 5432 / 5050 |
| `mongo` | MongoDB 7.0 + mongo-express | 27017 / 8081 |
| `mq` | RabbitMQ（management） | 5672 / 15672 |
| `kafka` | Kafka 3.x KRaft（免 ZK）+ Kafka UI | 9092 / 8082 |
| `es` | Elasticsearch 7.17 + Kibana | 9200 / 5601 |
| `etcd` | etcd 单节点 + etcdkeeper | 2379 / 8083 |
| `php` | php:fpm + nginx 测试栈 | 8084 |
| `jenkins` | Jenkins LTS | 8085 |
| `mysql-test` | MySQL 8（测试库，与正式隔离） | 3307 |
| `redis-test` | Redis 7（测试实例） | 6380 |

> **Mongo 4.4 已 EOL，建议直接升 7.0。**
> **Kafka 3.x 推荐 KRaft 单节点模式，免 Zookeeper，省一个容器。**

参考 compose 片段（节选，示意 profile 用法）：

```yaml
# /opt/stacks/lab/docker-compose.yml
services:
  postgres:
    image: postgres:15
    profiles: ["pg"]
    ports: ["5432:5432"]
    volumes: ["pg-data:/var/lib/postgresql/data"]
    environment:
      POSTGRES_PASSWORD: lab

  mongo:
    image: mongo:7.0
    profiles: ["mongo"]
    ports: ["27017:27017"]
    volumes: ["mongo-data:/data/db"]

  kafka:
    image: bitnami/kafka:3.7
    profiles: ["kafka"]
    ports: ["9092:9092"]
    environment:
      KAFKA_ENABLE_KRAFT: "yes"
      KAFKA_CFG_PROCESS_ROLES: "broker,controller"
      KAFKA_CFG_NODE_ID: "1"
      KAFKA_CFG_CONTROLLER_QUORUM_VOTERS: "1@kafka:9093"
      KAFKA_CFG_LISTENERS: "PLAINTEXT://:9092,CONTROLLER://:9093"
      KAFKA_CFG_ADVERTISED_LISTENERS: "PLAINTEXT://kafka:9092"
      KAFKA_CFG_CONTROLLER_LISTENER_NAMES: "CONTROLLER"

  elasticsearch:
    image: elasticsearch:7.17.13
    profiles: ["es"]
    environment:
      discovery.type: single-node
      ES_JAVA_OPTS: "-Xms512m -Xmx512m"
    ports: ["9200:9200"]

volumes:
  pg-data:
  mongo-data:
```

使用方式：

```bash
docker compose --profile kafka --profile es up -d     # 只启 kafka + es
docker compose --profile kafka down                   # 用完即关
```

## 5. 服务迁移映射表

| 现位置 | 服务 | 新位置 | 动作 |
|---|---|---|---|
| data_os | MinIO | core-data | 保留，独享 |
| data_os | Redis | core-data | 合并为业务 Redis，低内存 |
| data_os | Postgres 15 | lab | 改按需启动 |
| data_os | MongoDB 4.4 | lab | 升级到 7.0，按需 |
| data_os | RabbitMQ | lab | 按需 |
| data_os | Kafka + ZK | lab | 改 KRaft 单节点，免 ZK |
| data_os | Elasticsearch + Kibana | lab | 按需，学习用 |
| pro_uat | mindoc | app-svc | DB 指向 core-data |
| pro_uat | gitea | app-svc | DB 指向 core-data |
| pro_uat | MySQL（正式） | core-data | 升 MySQL 8，开 binlog |
| pro_uat | etcd | lab | 按需学习 |
| pro_uat | Redis | core-data（合并） | 不再单独 |
| pro_uat | nginx | app-svc（Caddy/nginx 统一） | 二选一 |
| pro_uat | PHP 环境 | 删除 | 需要时 lab 临时起 |
| ts_uat | vaultwarden | app-svc | 对外服务归一 |
| ts_uat | certd | app-svc | 与 Caddy/nginx 协同 |
| ts_uat | MySQL / Redis / nginx / PHP（测试） | lab 或删除 | 不与正式混 |
| tool_uat | spug | app-svc | DB 指向 core-data |
| tool_uat | Jenkins | lab 或替换 | 推荐替换为 Gitea Actions |
| tool_uat | Loki | devops | 升级为实际采集栈 |
| tool_uat | Logstash | 删除 | 换 Promtail |

## 6. 资源分配（替代主手册 §4.1 的 UAT 部分）

硬件边界：4 逻辑核 / 32G，ESXi 自身约 2G，核心层（openwrt + win2016 + jumpserver）约 7G，**UAT 可用预算 ≈ 20G 常驻 + 爆发。**

| VM | vCPU | 内存 | CPU 预留 | CPU 限制 | CPU 份额 | 内存预留 | 启动策略 |
|---|---:|---:|---:|---:|---|---:|---|
| core-data | 2 | 6G | 500 MHz | 不限制 | 高 | 4G | 自动 60s |
| app-svc | 2 | 5G | 300 MHz | 3000 MHz | 正常 | 3G | 自动 90s |
| devops | 2 | 3G | 0 | 2000 MHz | 低 | 1G | 自动 150s |
| lab | 2 | 6G | 0 | 2500 MHz | 低 | 0 | 手动 |

说明：

- `core-data` CPU 份额**高**、内存预留 4G：正式数据不能被挤压。
- `app-svc` 给 CPU 上限，避免对外服务被刷爆整机。
- `devops` 份额最低，允许被抢占。
- `lab` 手动启动；常驻 4 台时宿主仍余 2-3G 余量，学习时再手动开 lab。
- 常驻内存总计 = **6 + 5 + 3 = 14G**（比现状 6+6+6=18G 还省）。

## 7. 启动顺序（全局对齐）

```text
openwrt     → 0s
win2016     → 15s
jumpserver  → 30s
core-data   → 60s     # 数据先就绪
app-svc     → 90s     # 应用起来连 DB
devops      → 150s    # 工具最后
lab         → 手动
```

## 8. 关键技术选型调整

| 原方案 | 新方案 | 节省 | 理由 |
|---|---|---|---|
| Kafka + Zookeeper 两容器 | Kafka KRaft 单容器 | ~500M 内存 + 一套配置 | Kafka 官方推荐，学习更贴新版 |
| Logstash | Promtail / Vector | ~1G 内存 | Logstash 对 N5105 太重 |
| Jenkins（学习） | Gitea Actions Runner | ~1.5G 内存 | gitea 已常驻，学 CI 更贴实际 |
| nginx + php-fpm（未用） | Caddy 统一反代 | 少维护两个组件 | Caddy + certd 自动 HTTPS |
| Mongo 4.4.25（EOL） | Mongo 7.0 | —— | 4.4 已停支，学新版 |
| 多份 Redis | 1 × 业务 Redis + 1 × lab 测试 Redis（按需） | 2 份常驻内存 | 正式/测试彻底分离 |

## 9. 网络与反代规划

### 9.1 网段约定（示例，按你实际内网替换）

- ESXi 管理：`192.168.20.0/24`
- openwrt 内网：`192.168.10.0/24`

| VM | 固定 IP 建议 | 网段 |
|---|---|---|
| core-data | `192.168.10.21` | 内网 |
| app-svc | `192.168.10.22` | 内网 |
| devops | `192.168.10.23` | 内网 |
| lab | `192.168.10.24` | 内网 |

### 9.2 对外入口统一

- openwrt 只把 `80 / 443` 端口转发到 `app-svc` 的 Caddy/nginx。
- 所有对外域名（vaultwarden、gitea、mindoc、spug）都经过 `app-svc` 反代。
- `core-data` / `devops` / `lab` **不直接暴露到公网**，只通过 openwrt 内网可达。

### 9.3 Caddy 示例配置

```caddyfile
# /opt/stacks/app-svc/Caddyfile
vault.example.com {
    reverse_proxy vaultwarden:80
}

git.example.com {
    reverse_proxy gitea:3000
}

doc.example.com {
    reverse_proxy mindoc:8181
}

deploy.example.com {
    reverse_proxy spug-nginx:80
}
```

### 9.4 内网服务访问约定

- 应用容器之间走 docker network，但**跨 VM** 时走固定 IP，例如 `mindoc` 连 `192.168.10.21:3306`。
- 所有数据库端口**只监听内网**，不暴露到 openwrt WAN。

## 10. 备份与恢复

### 10.1 备份策略（core-data 上）

| 数据源 | 工具 | 频率 | 保留 | 目标 |
|---|---|---|---|---|
| MySQL 全库 | `mysqldump --single-transaction --routines --triggers` | 每日 02:00 | 14 天 | `/data/backup/mysql/YYYYMMDD/*.sql.gz` |
| MySQL binlog | 原生 binlog（开启 `log_bin`） | 实时滚动 | 7 天 | `/data/mysql/binlog/` |
| MinIO | `mc mirror --overwrite --remove` | 每日 03:00 | 镜像 | 外部对象存储或另一宿主 |
| Redis | `BGSAVE` + RDB 拷贝 | 每日 02:30 | 7 天 | `/data/backup/redis/` |

示例 cron（写到 `/etc/cron.d/core-data-backup`）：

```cron
0 2 * * * root /opt/scripts/backup-mysql.sh >> /var/log/backup.log 2>&1
30 2 * * * root /opt/scripts/backup-redis.sh >> /var/log/backup.log 2>&1
0 3 * * * root /opt/scripts/backup-minio.sh >> /var/log/backup.log 2>&1
```

示例 `backup-mysql.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail
DATE=$(date +%Y%m%d)
DIR=/data/backup/mysql/$DATE
mkdir -p "$DIR"
docker exec core-data-mysql \
  mysqldump -uroot -p"$MYSQL_ROOT_PWD" --single-transaction --routines --triggers --all-databases \
  | gzip > "$DIR/all.sql.gz"
find /data/backup/mysql -maxdepth 1 -type d -mtime +14 -exec rm -rf {} +
```

### 10.2 恢复演练（每季度一次）

1. 在 `lab` 起一个临时 MySQL 8。
2. 从 `core-data` 拉最新 `all.sql.gz`，`zcat ... | mysql -h lab -uroot -p`。
3. 验证 gitea / mindoc 关键表行数与最新记录时间。
4. 记录恢复耗时；若 > 30 分钟，考虑把备份频率或存储方式升级（见 §12.3）。

### 10.3 外部备份落地（强烈建议）

- 本机之外再落一份：
  - 方式 A：rsync / mc mirror 到另一台物理机或 NAS。
  - 方式 B：备份到 S3 兼容对象存储（腾讯云 COS / 阿里云 OSS 免费额度）。
- 密钥通过 `.env` 注入，不进 Git。

## 11. 监控与日志

### 11.1 日志栈（devops）

```text
Promtail (core-data/app-svc/lab)  ──>  Loki (devops)  ──>  Grafana (devops)
```

Promtail 采集源：

- `/var/lib/docker/containers/*/*-json.log`（容器日志）
- `/var/log/syslog`、`/var/log/messages`（系统日志）
- 关键应用日志文件（gitea / mindoc 的业务日志）

Grafana 面板：

- 综合日志视图（按 VM / 容器名过滤）
- 对外访问错误率（从 Caddy access log 解析）
- MySQL slowlog 仪表

### 11.2 指标栈（可选扩展）

```text
node_exporter / cadvisor (各 VM)  ──>  Prometheus (devops)  ──>  Grafana
```

- `node_exporter`：VM 级 CPU/内存/磁盘/网络
- `cadvisor`：容器级指标
- 额外：`mysqld_exporter`、`redis_exporter`、`minio`（MinIO 原生暴露 Prometheus 端点）

### 11.3 告警（可选扩展）

- Grafana Alerting → 企业微信 / 钉钉 / Telegram Webhook。
- 首批告警：
  - VM 内存使用 > 85% 持续 5 分钟
  - MySQL 主进程不存在
  - 对外服务 5xx 错误率 > 5%
  - 磁盘使用 > 85%

## 12. 扩展性与未来演进

### 12.1 横向扩容路径

| 场景 | 处置 |
|---|---|
| app-svc 应用变多 CPU 撑不住 | 新增 `app-svc-2`，Caddy 在 openwrt 侧做 DNS/路由分流 |
| core-data 存储容量告急 | 给 `core-data` VM 新挂一块 ESXi 虚拟磁盘，扩 `/data` |
| 日志量爆炸 | Loki 换 `boltdb-shipper + S3` 后端，历史日志落对象存储 |
| 宿主升级到更大硬件 | 直接把 4 台 VM 的 OVF 导出，迁移到新宿主不改结构 |

### 12.2 替代路径（同结构、不同实现）

| 层 | 默认方案 | 可替换方案 | 何时选用 |
|---|---|---|---|
| 反代 | Caddy | Traefik / nginx + certd | 需要 docker label 自动路由 → Traefik |
| 日志 | Loki + Promtail | ELK（es+kibana+filebeat） | 真要学 ELK → lab 长期开 |
| CI | Gitea Actions Runner | Jenkins / Drone / GitLab Runner | 团队协作需要更多插件 |
| 对象存储 | MinIO | SeaweedFS / Garage | 需要分布式/更轻量 |
| 消息队列 | lab Kafka KRaft | NATS JetStream / Redpanda | 追求更轻、更现代 |
| 监控 | Grafana + Loki + Prometheus | VictoriaMetrics + VictoriaLogs | 内存更省、查询更快 |

### 12.3 平滑升级路径

- **MySQL 8 → MySQL 9 / PG 迁移**：core-data 预留一块独立数据盘，升级时起新容器挂新盘，双写切流。
- **单节点 → 主从**：当出现读压力，core-data 上多起一个 `mysql-replica` 容器，或新开 `core-data-2`。
- **VM 级备机**：core-data 做定期 ESXi 快照 + 冷备导出 OVF，作为宿主灾备手段之一。
- **lab 零 VM 化**：如果 N5105 升级为更高配宿主，可把 lab 合并回 devops；反过来若宿主压力大，lab 甚至可以改到 win2016 里用 WSL2 按需跑。

### 12.4 服务生长规则（避免未来再次混乱）

**新增服务前先回答三问，决定去哪台 VM：**

1. 丢了数据会不会"哭"？会 → core-data；不会 → 进入第 2 问。
2. 对外服务 / 每天都用？是 → app-svc；否 → 进入第 3 问。
3. 是工具类（CI / 日志 / 监控）？是 → devops；否 → lab。

**禁止**：
- 在 `core-data` 跑任何非存储类容器（含应用、CI、日志）。
- 在 `app-svc` 直接存放正式业务持久化数据。
- 在 `lab` 放任何对外服务。

## 13. 落地步骤（全新重做顺序）

> 不考虑旧数据迁移，按"建新→验证→切流"的顺序。

1. **宿主侧**：在 ESXi 删除旧 4 台 UAT，按 §6 资源表新建 `core-data` / `app-svc` / `devops` / `lab` 四台空 VM。
2. **系统准备**（每台 VM）：
   - 最小化系统安装（发行版选型详见 §15；初始化命令详见附录 A/D）
   - 安装 Docker + docker compose 插件
   - 创建 `/data` 与 `/opt/stacks` 目录
   - 固定 IP（§9.1）
3. **core-data**：
   - 起 MySQL 8 / MinIO / Redis 三个 compose 栈，数据卷挂 `/data/...`
   - 开 MySQL binlog（`log_bin`、`server_id`）
   - 配置 §10 备份脚本与 cron
4. **app-svc**：
   - 起 Caddy（或 nginx + certd）作为统一入口
   - 按顺序部署 vaultwarden、certd、mindoc、gitea、spug（DB 都指向 core-data）
   - 在 openwrt 上配置 `80/443 → app-svc` 的端口转发
5. **devops**：
   - 起 Loki + Grafana
   - 在 core-data / app-svc / lab 装 Promtail agent，指向 devops 的 Loki
   - Grafana 加 Loki 数据源 + 综合日志 dashboard
   - 需要 CI 时安装 gitea-actions-runner，指向 app-svc 的 gitea
6. **lab**：
   - 写带 profiles 的 `docker-compose.yml`
   - 不启动任何服务；VM 设为手动启动
   - 在备忘记好："想学 X → 开机 lab → `docker compose --profile X up -d`"
7. **演练**：跑一遍主手册 §8 演练清单，新增：
   - 关掉 lab，观察宿主内存是否空出 ≥ 3G
   - 模拟 app-svc 崩溃，验证 core-data 的 MySQL 不受影响
   - 从备份恢复到 lab 的临时 MySQL，验证数据可用

## 14. 验收标准

满足以下条件即视为重规划完成：

- [ ] UAT 常驻 4 台总内存使用 ≤ 14G（不含 lab）。
- [ ] lab 手动关闭时，宿主可用内存 ≥ 3G。
- [ ] `core-data` 上只运行 MySQL / MinIO / Redis 三类容器，无任何应用或工具类容器。
- [ ] 所有对外服务（vaultwarden / gitea / mindoc / spug）均经 `app-svc` 反代，openwrt 仅转发 80/443 到 `app-svc`。
- [ ] Grafana 能看到 4 台 VM 的容器日志，检索一条任意时间段日志耗时 < 5 秒。
- [ ] 在 `lab` 从备份恢复 MySQL 到可用状态，耗时 < 30 分钟。
- [ ] 任意单台 UAT VM 强关，其他三台服务不受影响（尤其 `core-data` 不能依赖任何其它 UAT）。
- [ ] 新增服务时，按 §12.4 的三问能直接归位到某台 VM，无歧义。

---

## 15. 操作系统选型

### 15.1 选型诉求

原有现状：

- `data_os` / `tool_uat`：CentOS 7.9，历史上稳定，但 **2024-06-30 已 EOL**（无安全补丁）、内核 3.10 过旧、未利用 N5105 现代特性。
- `pro_uat` / `ts_uat`：Ubuntu 24.04，时不时挂机，早期版本与 ESXi 配合存在稳定性问题。

重规划后的三条诉求：

1. **稍微新一点的内核**（能利用 N5105 特性、支持现代容器能力）。
2. **追求稳定**（"装完不动就不坏"，不吃螃蟹）。
3. **与 CentOS 7 时代的运维经验可衔接**（可选，非硬性）。

### 15.2 推荐矩阵（三选一并列推荐）

| 维度 | Debian 12 (Bookworm) | Rocky Linux 9 / AlmaLinux 9 | Ubuntu 22.04 LTS |
|---|---|---|---|
| 发布时间 | 2023-06 | 2022-11 | 2022-04 |
| 基线内核 | **6.1 LTS** | 5.14（RHEL 9 backport 了大量 6.x 特性） | 5.15 LTS |
| 可选新内核 | 无需升级 | ELRepo `kernel-lt` 6.1 LTS / `kernel-ml` 最新 mainline | HWE 滚到 6.8 |
| 官方支持到 | 2028-06（含 LTS） | **2032-05（10 年）** | 2027-04 免费 / 2032 付费扩展 |
| 稳定性口碑 | **行业顶级** | **顶级**（RHEL 血统） | 优秀 |
| 资源占用 | **最低**（~200MB 启动） | 中等（~400MB） | 中等（~350MB） |
| 默认安全模块 | 无强制 LSM | SELinux（enforcing） | AppArmor |
| 容器生态 | Docker 官方源完备 | Podman 原生 + Docker 可装 | Docker 官方源完备 |
| 与 CentOS 7 习惯延续 | 中（apt 换 yum） | **高**（`dnf`/`systemctl`/`firewalld`/SELinux 全延续） | 低 |
| 自动升内核风险 | 低（走 point release） | 低（`dnf` 默认不自动升） | 中（`unattended-upgrades` 默认开） |

**任选一个作为四台 UAT 的统一系统**，三者都满足"新内核 + 稳定"的核心诉求。

### 15.3 三个候选的理由与取舍

#### 15.3.1 Debian 12 (Bookworm)

**优点**：

- 内核 6.1 LTS 原生支持，无需折腾 ELRepo。
- 资源占用最低，N5105 + 32G 能省内存就是省容器。
- "不动就不坏"文化，安全补丁走 point release（12.1/12.2/...），不会像 Ubuntu 那样 HWE 内核乱滚。
- Docker 官方源兼容零折腾，无 AppArmor profile 干扰。
- 社区文档丰富，生态中立。

**适合场景**：

- 追求"轻 + 稳 + 新内核"三合一。
- 不介意 `apt` 生态。
- 不需要 10 年级长期支持。

**注意点**：

- 默认不装任何防火墙，需手动装 `ufw` 或 `nftables`。
- 无默认 LSM（AppArmor/SELinux），安全加固需自己来。
- Debian 13（Trixie）已于 2025-08 发布、内核 6.12 LTS；如果希望更新内核、且接受"新发布 8 个月"的新鲜度，可直接跳 13。

#### 15.3.2 Rocky Linux 9 / AlmaLinux 9

**优点**：

- RHEL 9 血统，红帽做的企业级回归测试沉淀进 point release（9.1→9.2→...→9.5/9.6）。
- `yum/dnf`、`systemctl`、`firewalld`、SELinux 与 CentOS 7 延续，肌肉记忆不丢。
- **生命周期 10 年**（2032-05），装一次用到不想用为止。
- 基线 5.14 其实被 backport 了大量 6.x 特性：调度器、BPF、存储栈、KVM 全是新代码。
- 想要更新内核可通过 ELRepo 一条命令切到 `kernel-lt` 6.1 LTS 或 `kernel-ml` 最新（见 §15.6）。
- 默认开启 SELinux，安全基线比 Debian/Ubuntu 默认更高。

**适合场景**：

- 你本来就是 CentOS 7 老用户，切换成本最低。
- 追求 10 年级长期支持。
- 希望基线最稳、能"想升内核再升"的灵活性。

**注意点**：

- 默认内核 5.14，若你认为这个版本号"不够新"，需走 §15.6 升 ELRepo 通道。
- 从容器 bind-mount 目录时需处理 SELinux context（见 §15.6 末节）。
- 默认装 `cockpit`（9090 端口），不用就 `dnf remove cockpit-*`。

**Rocky 与 AlmaLinux 的区别**（见 §15.7）：功能上完全等价，路线哲学不同，任选其一即可。

#### 15.3.3 Ubuntu 22.04 LTS（备选）

**优点**：

- apt 生态 + HWE 内核 6.8（"已经在 22.04 上被验证一年"的 6.8，比 24.04 的原生 6.8 稳定得多）。
- 社区文档、第三方教程、NVIDIA 驱动等场景支持最广。
- 已发布 4 年，成熟度高。

**适合场景**：

- 某个特定软件文档只给 Ubuntu 示例（NVIDIA/AI 工具链等）。
- 偏好 apt 生态，同时需要比 Debian 更热门的发行版。

**注意点**：

- 免费支持只到 2027-04（比 AlmaLinux 短 5 年）。
- 默认带 `snap`，即使不用也占空间 + 偶发后台活动。
- `unattended-upgrades` 默认开启，夜里可能自动重启，需手动关（见 §15.8.3）。

### 15.4 明确不推荐的发行版

| 发行版 | 不推荐原因 |
|---|---|
| **CentOS 7.9** | 2024-06-30 已 EOL，无安全补丁，内核 3.10 过旧 |
| **Ubuntu 24.04 LTS**（早期） | 24.04.0/.1 在 ESXi 下挂机问题多；24.04.2+ 改善，但仍非"最稳"首选 |
| **CentOS Stream 9** | 滚动上游，定位"RHEL 之前的测试通道"，不适合追求稳定 |
| **Fedora** | 6 个月发布节奏、13 个月支持，更新过快 |
| **openSUSE Leap 15.6** | 稳定但社区偏小，生态不如 Debian/RHEL，`zypper` 习惯成本高 |
| **Alpine Linux** | musl libc 与部分 glibc 二进制有边缘兼容坑，不适合做"宿主" |
| **NixOS** | 学习曲线陡，不适合"追求稳定"诉求 |

### 15.5 四台 VM 的系统分配建议

**核心原则：四台统一同一发行版，不混搭。**

理由：

1. 批量补丁、`apt upgrade` / `dnf update` 一条脚本通吃。
2. Promtail / node_exporter / docker-compose 配置模板完全一致。
3. 出问题对比基线时变量只剩"容器"这一层。
4. 学多系统的诉求通过**容器**满足：`docker run -it debian:12 bash` / `rockylinux:9` / `ubuntu:24.04` 随用随起，零成本。

#### 15.5.1 方案 A：统一 Debian 12（轻量 + 新内核）

| VM | 系统 | 内核 | 备注 |
|---|---|---|---|
| core-data | Debian 12 | 6.1 LTS（默认） | 最稳，正式数据 |
| app-svc | Debian 12 | 6.1 LTS（默认） | 同上 |
| devops | Debian 12 | 6.1 LTS（默认） | 同上 |
| lab | Debian 12 | 6.1 LTS（默认） | 学多发行版走容器 |

**常驻内存估算**（基础系统占用）：每台约 200MB × 4 ≈ 800MB 基础开销。

#### 15.5.2 方案 B：统一 Rocky / AlmaLinux 9（RHEL 血统 + 10 年支持）

| VM | 系统 | 内核通道 | 备注 |
|---|---|---|---|
| core-data | Rocky/AlmaLinux 9 | 基线 5.14 | 正式数据，不折腾内核 |
| app-svc | Rocky/AlmaLinux 9 | 基线 5.14 | 同上 |
| devops | Rocky/AlmaLinux 9 | 基线 5.14 | 同上 |
| lab | Rocky/AlmaLinux 9 | **ELRepo `kernel-lt` 6.1 LTS** | 尝鲜新内核 + 容器玩多系统 |

**常驻内存估算**：每台约 400MB × 4 ≈ 1.6GB 基础开销（比方案 A 多约 800MB）。

#### 15.5.3 决策建议

按优先级回答以下问题：

1. **你更熟悉 apt 还是 yum/dnf？**
   - `apt` → 倾向方案 A（Debian 12）
   - `yum/dnf`（CentOS 7 延续）→ 倾向方案 B
2. **你是否需要 10 年级长期支持？**
   - 是 → 方案 B
   - 否 → 方案 A
3. **你是否在意资源占用（每台多 200MB 内存）？**
   - 是 → 方案 A
   - 否 → 方案 B

**我的默认推荐**：

- 如果选 apt 系 → **方案 A（Debian 12）**。
- 如果选 yum 系 → **方案 B（Rocky 9 或 AlmaLinux 9）**。

Ubuntu 22.04 只有在"有专门用 Ubuntu 的工具链"时才考虑作为第三方案。

### 15.6 新内核通道（仅 RHEL 系 / 方案 B）

RHEL 9 默认 5.14，若要"稍微新一点"的感觉可切换 ELRepo 内核通道：

| 通道 | 内核版本 | 适用 |
|---|---|---|
| 基线 | 5.14（带 backport） | **业务三台推荐**（最稳） |
| ELRepo `kernel-lt` | 6.1 LTS | **`lab` 推荐** / 想要 LTS 新内核 |
| ELRepo `kernel-ml` | 最新 mainline（6.12+） | 追新，不建议生产 |

**ELRepo 切换命令**（示例，按需执行）：

```bash
# 安装 ELRepo 仓库
dnf install -y https://www.elrepo.org/elrepo-release-9.el9.elrepo.noarch.rpm

# 安装 LT（长期支持）内核
dnf --enablerepo=elrepo-kernel install -y kernel-lt kernel-lt-devel kernel-lt-headers

# 或安装 ML（主线）内核
# dnf --enablerepo=elrepo-kernel install -y kernel-ml kernel-ml-devel kernel-ml-headers

# 设置为默认启动内核
grubby --set-default /boot/vmlinuz-*.el9.elrepo.x86_64

# 重启生效
reboot
```

**SELinux 与容器 bind-mount 的处理**（与内核通道无关，但 RHEL 系通用）：

```bash
# 方式一：给宿主数据目录打容器标签（推荐）
semanage fcontext -a -t container_file_t "/data(/.*)?"
restorecon -Rv /data

# 方式二：在 compose 中加 :z 后缀，让 Docker 自动处理
# volumes:
#   - /data/mysql:/var/lib/mysql:z
```

### 15.7 Rocky Linux 9 vs AlmaLinux 9 的区别

两者在功能、仓库、兼容性上**等价**，区别在于"路线哲学"：

| 维度 | Rocky Linux 9 | AlmaLinux 9 |
|---|---|---|
| 兼容策略 | **bit-for-bit** 克隆 RHEL | **ABI 兼容**（9.5 起） |
| 背后组织 | RESF 基金会 + CIQ 商业公司 | AlmaLinux OS Foundation（纯社区） |
| 源码获取 | 通过 UBI 镜像 + 云订阅拿 RHEL 源码 | 基于 CentOS Stream + 社区 CVE 补丁 |
| CVE 补丁速度 | 跟 RHEL 节奏（红帽延迟它也延迟） | **可自行合并紧急补丁**，偶尔快几小时 |
| 对 RHEL 100% 一致 | **是**（核心卖点） | 绝大多数一致，极少数用更新版本包 |
| 企业合规友好度 | 高（bit-for-bit 对合规更友好） | 高 |
| 社区活跃度 | 高 | 略高 |
| 维护者 | Gregory Kurtzer（CentOS 原创始人） | 社区基金会 |

**ISO / 仓库地址差异（其余操作完全一致）**：

```text
Rocky Linux 9:   https://download.rockylinux.org/pub/rocky/9/isos/x86_64/
AlmaLinux 9:     https://repo.almalinux.org/almalinux/9/isos/x86_64/
```

**如何选**：

- **选 Rocky Linux 9**：看重 "bit-for-bit" 严谨性、CIQ 商业背书、未来可能转 RHEL 订阅、认可 Kurtzer 路线。
- **选 AlmaLinux 9**：看重 CVE 响应速度、认可纯社区基金会模式、不希望被红帽源码获取政策波及。

**对个人实验室场景**：两者差异感知不到 5%，拍脑袋二选一即可；官网文档哪个读着更顺就选哪个。

### 15.8 ESXi 下 Linux 稳定性通用加固

与发行版无关，四台 VM 均应执行。

#### 15.8.1 VM 层（ESXi 虚拟机设置）

- **网卡类型**：`VMXNET3`（不要用 E1000/E1000e）。
- **SCSI 控制器**：`VMware Paravirtual (PVSCSI)`。
- **关闭内存热添加 / CPU 热插拔**：VM 编辑 → 取消勾选 "Enable memory hot add / CPU hot plug"。
  - 这是 Ubuntu 24 挂机的常见根因之一；RHEL/Debian 下也建议关闭。
- **Guest OS 类型**：精确选择对应发行版，不要用 "Other Linux"：
  - Debian 12 → `Debian Linux 12 (64-bit)`
  - Rocky/AlmaLinux 9 → `Red Hat Enterprise Linux 9 (64-bit)`
  - Ubuntu 22.04 → `Ubuntu Linux (64-bit)`
- **VMware Tools**：使用发行版仓库的 `open-vm-tools`，**不要用 VMware 官方 tarball 手装**：

  ```bash
  # Debian / Ubuntu
  apt install -y open-vm-tools

  # Rocky / AlmaLinux
  dnf install -y open-vm-tools
  systemctl enable --now vmtoolsd
  ```

- **虚拟硬件版本**：建议 vmx-19（vSphere 7.0 U2）或更高，太老的硬件版本会导致 6.x 内核部分驱动降级。

#### 15.8.2 系统层调优（所有发行版通用）

```bash
# /etc/sysctl.d/99-docker-host.conf
vm.swappiness = 10
vm.overcommit_memory = 1
fs.file-max = 1048576
fs.inotify.max_user_watches = 524288
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.ip_local_port_range = 10240 65535

# 应用生效
sysctl --system
```

**禁用透明大页（THP）**（MySQL / PG / Redis / MongoDB 全部推荐）：

```ini
# /etc/systemd/system/disable-thp.service
[Unit]
Description=Disable Transparent Huge Pages
After=sysinit.target local-fs.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c "echo never > /sys/kernel/mm/transparent_hugepage/enabled && echo never > /sys/kernel/mm/transparent_hugepage/defrag"
RemainAfterExit=yes

[Install]
WantedBy=basic.target
```

```bash
systemctl daemon-reload
systemctl enable --now disable-thp
```

#### 15.8.3 内核更新策略（防止自动重启挂掉）

| 发行版 | 默认行为 | 建议 |
|---|---|---|
| Debian 12 | 不自动升内核（走 point release 手动） | 保持默认 |
| Rocky/AlmaLinux 9 | `dnf automatic` 默认关闭 | 保持默认，**不要自行开启** |
| Ubuntu 22.04 / 24.04 | `unattended-upgrades` 默认开启，可能自动重启 | 改配置或禁用 |

Ubuntu 关闭自动重启的做法：

```bash
# /etc/apt/apt.conf.d/50unattended-upgrades
Unattended-Upgrade::Automatic-Reboot "false";

# 或彻底关闭
systemctl disable --now unattended-upgrades
```

#### 15.8.4 关键服务监控钩子

让挂机不再"静默"：

- 开启 `systemd-oomd` 或传统 oom-killer 的日志，`journalctl -b -p err` 有内容立刻报警。
- Promtail 采集 `journald`（见 §11.1），一旦出现如下关键字立即 Grafana 告警：
  - `oom-killer`
  - `watchdog: BUG: soft lockup`
  - `Hardware Error`
  - `kernel panic`
  - `Uncorrectable memory error`

### 15.9 Ubuntu 24.04 挂机问题诊断（保留 Ubuntu 场景时参考）

如果未来再次接手 Ubuntu 机器，优先执行以下命令定位根因：

```bash
# 最近一次崩溃前的错误
journalctl -b -1 -p err --no-pager | tail -200

# 硬件错误（CPU/内存/磁盘）
journalctl -b -1 | grep -iE 'mce|hardware error|soft lockup|hung task'

# OOM 记录
journalctl -b -1 | grep -i 'killed process'

# open-vm-tools 版本
vmware-toolbox-cmd -v

# AppArmor 状态（可能杀容器）
aa-status
```

常见根因排序：

1. open-vm-tools 与 6.8/6.11 内核兼容 bug（24.04.0/.1 高发，24.04.2+ 改善）。
2. 启用了内存热添加 / CPU 热插拔（§15.8.1）。
3. `unattended-upgrades` 自动升 HWE 内核触发重启。
4. AppArmor 3.x profile 变严杀容器（MySQL/ES 高发）。
5. systemd 256 早期版本 journald/logind 卡死（24.04.0 有过）。

### 15.10 迁移心智地图（CentOS 7 → 新系统）

全容器化场景下，宿主层差异其实很小：

| 操作 | CentOS 7 | Debian 12 | Rocky/AlmaLinux 9 |
|---|---|---|---|
| 安装包 | `yum install X` | `apt install X` | `dnf install X` |
| 搜索包 | `yum search X` | `apt search X` | `dnf search X` |
| 服务管理 | `systemctl` | `systemctl` | `systemctl`（一致） |
| 防火墙 | `firewalld` | `ufw` / `nftables` | **`firewalld`（延续）** |
| LSM | 可选 SELinux | 无默认 | **SELinux enforcing（延续）** |
| 日志 | `journalctl` | `journalctl` | `journalctl`（一致） |
| 网络配置 | `NetworkManager` / `ifcfg-*` | `systemd-networkd` / `ifupdown` | **`NetworkManager` / `nmcli`（延续）** |
| Docker 安装 | `yum + docker-ce.repo` | `apt + download.docker.com` | **`dnf + docker-ce.repo`（延续）** |
| 默认包管理器 | `yum` | `apt` | `dnf`（yum 兼容命令仍可用） |

Docker 装完之后，`docker compose` 层面所有东西两边**完全通用**，compose 文件、`.env`、Caddyfile、Promtail 配置不用改。

从 CentOS 7 平滑程度：**Rocky/AlmaLinux 9 ≫ Debian 12 ≈ Ubuntu 22.04**。

### 15.11 几个必须避开的坑（跨发行版汇总）

1. **别用 snap 装 Docker**（Ubuntu 系特别注意）：走官方仓库 `docker-ce`，避免路径映射问题。
2. **别在 Debian 默认开 `systemd-oomd`**：它与 Docker 内存管理有边缘情况冲突。
3. **别混用 `firewalld` 和 `ufw`**：Debian 上若用 `firewalld` 必须 `purge ufw`；RHEL 系不要装 `ufw`。
4. **时钟同步不要同时装两套**：`systemd-timesyncd` 与 `chrony` 任选其一，不要并存。
5. **装 Debian 时别勾 `Desktop`**：task-selection 只选 `SSH server` + `standard system utilities`，省一半内存。
6. **RHEL 系上不要随便 `setenforce 0`**：折中做法是保持 enforcing + 给 `/data` 打 `container_file_t` 标签（§15.6）。
7. **别用 `yum` 的 hard link（RHEL 系）**：`dnf` 已经是默认，习惯命令 `yum` → `dnf` 即可。
8. **ESXi 虚拟硬件版本别太老**：vmx-14 及以下对 6.x 内核驱动支持不全。

---

## 附录 A：VM 初始化清单（Debian 12，每台复用）

```bash
# 1. 系统更新
apt update && apt -y upgrade

# 2. 安装 Docker
curl -fsSL https://get.docker.com | bash
systemctl enable --now docker

# 3. 基础目录
mkdir -p /data /opt/stacks /opt/scripts /var/log/stacks

# 4. 时区 & NTP
timedatectl set-timezone Asia/Shanghai
apt -y install chrony && systemctl enable --now chrony

# 5. 防火墙（按需，示例 ufw）
ufw default deny incoming
ufw default allow outgoing
ufw allow from 192.168.10.0/24   # 仅允许内网
ufw enable
```

## 附录 B：Compose 栈文件布局

```text
/opt/stacks/
  ├── core-data/
  │    ├── docker-compose.yml     # mysql / minio / redis
  │    └── .env
  ├── app-svc/
  │    ├── docker-compose.yml     # caddy / vaultwarden / mindoc / gitea / spug / certd
  │    ├── Caddyfile
  │    └── .env
  ├── devops/
  │    ├── docker-compose.yml     # loki / grafana / promtail / (prometheus)
  │    ├── loki-config.yaml
  │    ├── promtail-config.yaml
  │    └── .env
  └── lab/
       ├── docker-compose.yml     # 带 profiles
       └── .env
```

## 附录 C：与主手册的字段对齐

本方案中 `core-data` / `app-svc` / `devops` / `lab` 分别对应主手册 §3.1 的服务分级：

| 本方案 | 主手册分级 |
|---|---|
| core-data | T2（长期业务，最核心） |
| app-svc | T2（长期业务，对外） |
| devops | T2（长期业务，低优先级） |
| lab | T3（按需开启） |

`openwrt` / `win2016` / `jumpserver` 的 T0 / T1 定位不变，配置沿用主手册 §4.1 / §5 / §6。

## 附录 D：VM 初始化清单（Rocky Linux 9 / AlmaLinux 9，每台复用）

同时适用于 Rocky Linux 9 和 AlmaLinux 9（两者等价）。

```bash
# 1. 系统更新
dnf -y update

# 2. 基础工具 & EPEL
dnf -y install epel-release
dnf -y install vim curl wget git net-tools bind-utils tmux htop lsof \
               policycoreutils-python-utils

# 3. open-vm-tools（ESXi 必装）
dnf -y install open-vm-tools
systemctl enable --now vmtoolsd

# 4. 基础目录
mkdir -p /data /opt/stacks /opt/scripts /var/log/stacks

# 5. 时区 & NTP
timedatectl set-timezone Asia/Shanghai
systemctl enable --now chronyd

# 6. Docker CE（RHEL 系官方仓库）
dnf -y remove podman buildah runc 2>/dev/null || true
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
dnf -y install docker-ce docker-ce-cli containerd.io \
               docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker

# 7. 防火墙（firewalld，只放内网）
firewall-cmd --permanent --new-zone=internal-net 2>/dev/null || true
firewall-cmd --permanent --zone=internal-net --add-source=192.168.10.0/24
firewall-cmd --permanent --zone=internal-net --add-service=ssh
firewall-cmd --permanent --zone=internal-net --add-port=80/tcp
firewall-cmd --permanent --zone=internal-net --add-port=443/tcp
firewall-cmd --reload

# 8. SELinux：给数据目录打容器标签（保持 enforcing 的折中方案）
semanage fcontext -a -t container_file_t "/data(/.*)?"
restorecon -Rv /data

# 9. 清理不需要的默认服务（按需）
dnf -y remove cockpit cockpit-* 2>/dev/null || true
```

**可选：升级到 ELRepo `kernel-lt` 6.1 LTS（仅 `lab` 建议，业务三台保持 5.14）：**

```bash
dnf install -y https://www.elrepo.org/elrepo-release-9.el9.elrepo.noarch.rpm
dnf --enablerepo=elrepo-kernel install -y kernel-lt kernel-lt-devel kernel-lt-headers
grubby --set-default /boot/vmlinuz-*.el9.elrepo.x86_64
reboot
```

**Rocky 与 AlmaLinux 的仅有差别（安装阶段）**：

```text
Rocky Linux 9 ISO:   https://download.rockylinux.org/pub/rocky/9/isos/x86_64/
AlmaLinux 9 ISO:     https://repo.almalinux.org/almalinux/9/isos/x86_64/
```

安装后 `/etc/os-release` 显示不同，其余操作命令完全一致。
