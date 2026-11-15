# ansible-lab

常用中间件 Ansible Playbook 集合。兼容 **麒麟 V10 / CentOS 8 / Ubuntu**，支持 x86_64 与 arm64。

## 组件清单

| 入口 | 组件 | 部署形态 | 分组约定 |
|------|------|---------|---------|
| `common.yml` | 系统初始化 | 关防火墙/SELinux、chrony 时间同步、内核参数、limits | `all` |
| `mysql.yml` | MySQL 8.0 | 主从复制（GTID），tar.xz 通用二进制 | `mysql-master` / `mysql-slave` |
| `redis.yml` | Redis 7.x | 单机 / 主从+哨兵，源码编译 | `redis-master` / `redis-replica` |
| `nginx.yml` | Nginx | 反向代理/负载均衡，官方源 | `nginx` |
| `kafka.yml` | Kafka 3.x | 集群（ZK 模式或 KRaft 模式） | `kafka`（复用 `zookeeper-servers`） |
| `elasticsearch.yml` | ES 8.x + Kibana | 集群 + 可选 Kibana | `elasticsearch` / `kibana` |
| `minio.yml` | MinIO | 单机 / 分布式纠删码 | `minio` |
| `rabbitmq.yml` | RabbitMQ 3.x | 集群 + 管理插件 | `rabbitmq` |
| `rocketmq.yml` | RocketMQ 5.x | NameServer + Broker | `rocketmq-nameserver` / `rocketmq-broker` |
| `nacos.yml` | Nacos 2.x | standalone / 集群（外置 MySQL） | `nacos` |
| `mongodb.yml` | MongoDB 7.x | 副本集 | `mongodb` |
| `monitoring.yml` | Prometheus + Grafana + 采集 agent 全家桶 | 见下文 | `prometheus` / `grafana` + 各组件组 |
| `hadoop.yml` / `spark.yml` / `zk_install.yml` | Hadoop / Spark / Zookeeper（原有保留） | — | 见各自文件 |

### Prometheus 采集 agent（`monitoring.yml` 自动部署）

| exporter | 端口 | 安装位置 |
|----------|------|---------|
| node_exporter | 9100 | `all` 全节点 |
| mysqld_exporter | 9104 | `mysql-master:mysql-slave` |
| redis_exporter | 9121 | `redis-master:redis-replica` |
| mongodb_exporter | 9216 | `mongodb` |
| elasticsearch_exporter | 9114 | `elasticsearch` |
| kafka_exporter | 9308 | `kafka` |
| nginx-prometheus-exporter | 9113 | `nginx`（需在 nginx 配置 stub_status） |
| blackbox_exporter | 9115 | `prometheus` 组（拨测） |

## 快速开始

```bash
# 1. 准备 inventory
cp hosts.example hosts   # 编辑分组（hosts 不入库）

# 2. 按需部署（common 会自动作为前置 role 执行）
ansible-playbook mysql.yml
ansible-playbook monitoring.yml
```

> 零外部 collection 依赖：全部使用 `ansible.builtin` 内置模块，ansible-core ≥ 2.10 即可运行（控制端需 Linux/macOS/WSL）。

## 包下载缓存机制

不把二进制提交进仓库（`.gitignore` 已排除），也避免每次都从公网下载：

- 每个组件 role 通过公共 `fetch_package` role 取包：
  1. 先查控制端缓存目录 `{{ playbook_dir }}/.pkg-cache/`（`local_cache_dir` 可改）
  2. 缓存缺失才从 `*_download_url` 下载
  3. 再 copy 到目标机执行
- **内网镜像**：每个组件都有 `xxx_download_url` 变量，在 `group_vars` 或 `-e` 覆盖即可整体切到内部制品库
- 下载源默认混用 [华为云镜像](https://repo.huaweicloud.com/)（mysql/kafka/es/redis/nacos/rocketmq）与官方（minio/mongodb/grafana/exporter 的 GitHub Releases）

## 变量覆盖示例

```bash
# 改 MySQL root 密码与数据盘
ansible-playbook mysql.yml -e mysql_root_password='Strong_Pass' -e data_base_dir=/data2

# Redis 关闭哨兵
ansible-playbook redis.yml -e redis_sentinel=false

# Kafka 切 KRaft 模式
ansible-playbook kafka.yml -e kafka_kraft=true
```

各组件全部可调变量见 `roles/<组件>/defaults/main.yml`。

## 兼容性说明

- **麒麟 V10 / CentOS 8**：`os_family=RedHat` 走 yum/dnf 分支；EL8 生态包（nginx/erlang/grafana rpm 直接兼容）
- **Ubuntu**：`os_family=Debian` 走 apt 分支；chrony 配置路径 `/etc/chrony/chrony.conf` 已单独处理
- **arm64**：mysql/es/minio/exporter 均按 `ansible_architecture` 自动选择 aarch64/arm64 包
- 老组件（hadoop/spark/zookeeper）保留原有写法未做三 OS 适配

## 目录结构

```
ansible-lab/
├── {组件}.yml              # 各组件部署入口
├── hosts.example           # 分组约定模板（复制为 hosts 使用）
├── requirements.yml        # galaxy collections
├── group_vars/all.yml      # 全局：下载缓存目录、通用路径
└── roles/
    ├── fetch_package/      # 公共下载缓存机制
    ├── common/ jdk/        # 系统初始化、JDK（8/11/17）
    └── mysql/ redis/ nginx/ kafka/ elasticsearch/ kibana/ minio/
        rabbitmq/ rocketmq/ nacos/ mongodb/ prometheus/ grafana/ exporter/
```
