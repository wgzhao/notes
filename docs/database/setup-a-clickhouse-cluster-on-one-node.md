---
title: 在一个节点上部署 ClickHouse 集群
description: 这篇帖子描述了，如何通过设置不同存储路径，监听端口等配置来实现在一个节点上运行一个 3 实例的 ClickHouse 集群
tags: ["clickhouse", "cluster"]
---

# 在一个节点上部署 ClickHouse 集群

在一个节点上通过不同端口部署 3 节点的 ClickHouse 集群，可以通过为每个实例分配不同的端口和数据目录来实现。以下是详细步骤：

## 环境准备

确保你的服务器有足够的资源（CPU、内存、磁盘）来运行多个 ClickHouse 实例。下载 ClickHouse 的二进制文件，并进行安装。

```bash
LATEST_VERSION=$(curl -s https://raw.githubusercontent.com/ClickHouse/ClickHouse/master/utils/list-versions/version_date.tsv | \
    grep -Eo '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' | sort -V -r | head -n 1)
export LATEST_VERSION

case $(uname -m) in
  x86_64) ARCH=amd64 ;;
  aarch64) ARCH=arm64 ;;
  *) echo "Unknown architecture $(uname -m)"; exit 1 ;;
esac

curl -fO "https://packages.clickhouse.com/tgz/stable/$PKG-$LATEST_VERSION-${ARCH}.tgz" \
  || curl -fO "https://packages.clickhouse.com/tgz/stable/$PKG-$LATEST_VERSION.tgz"
tar -xzvf "clickhouse-common-static-$LATEST_VERSION.tgz"
clickhouse-server-$LATEST_VERSION/install/doinst.sh
(cd /usr/bin && ln -s clickhouse clickhouse-server && ln -sf clickhouse clickhouse-client)
```

## 创建多个 ClickHouse 实例

我们将为每个实例创建独立的配置目录和数据目录。

### 创建目录结构

假设基础目录为 `/clickhouse`，创建以下目录结构：

```bash
mkdir -p /clickhouse/{instance1,instance2,instance3}/{config,data,logs}
```

- `instance1`、`instance2`、`instance3` 分别对应 3 个 ClickHouse 实例。
- 每个实例的 `config` 目录用于存放配置文件，`data` 目录用于存放数据，`logs` 目录用于存放日志。

## 配置每个实例

### 实例 1 配置

拷贝 `clickhouse-server-$LATEST_VERSION`下的 `config/config.xml` 文件和 `config/users.xml` 到
`/clickhouse/instance1/config/`。

创建 `/clickhouse/instance1/config/config.d` 目录,在该目录下分别创建以下文件:

**config.xml**

```xml
<clickhouse replace="true">
    <path>/clickhouse/instance1/data/</path>
    <tmp_path>/clickhouse/instance1/tmp/</tmp_path>
    <user_files_path>/clickhouse/instance1/user_files/</user_files_path>
    <format_schema_path>/clickhouse/instance1/format_schemas/</format_schema_path>
    <logger>
        <log>/clickhouse/instance1/logs/clickhouse-server.log</log>
        <errorlog>/clickhouse/instance1/logs/clickhouse-server.err.log</errorlog>
    </logger>
    <storage_configuration>
        <disks>
            <default>
                <keep_free_space_bytes>0</keep_free_space_bytes>
            </default>
            <data>
                <path>/clickhouse/instance1/disk_data/</path>
                <keep_free_space_bytes>0</keep_free_space_bytes>
            </data>
        </disks>
    </storage_configuration>
</clickhouse>
```

**port.xml**

```xml
<?xml version="1.0"?>
<yandex>
    <interserver_http_host>localhost</interserver_http_host>
    <http_port>8123</http_port>
    <tcp_port>9001</tcp_port>
    <mysql_port>9004</mysql_port>
    <postgresql_port>9005</postgresql_port>
</yandex>
```

**macros.xml**

```xml
<?xml version="1.0"?>
<yandex>
    <macros>
        <shard>1</shard>
        <replica>1</replica>
    </macros>
</yandex>
```

**remote_servers.xml**

```xml
<?xml version="1.0"?>
<yandex>
    <remote_servers>
        <testcluster>
            <shard>
                <weight>1</weight>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>127.0.0.1</host>
                    <port>9001</port>
                </replica>
                <replica>
                    <host>127.0.0.1</host>
                    <port>9002</port>
                </replica>
                <replica>
                    <host>127.0.0.1</host>
                    <port>9003</port>
                </replica>
            </shard>
        </testcluster>
    </remote_servers>
</yandex>
```

**zookeeper**

```xml
<?xml version="1.0"?>
<yandex>
    <zookeeper>
        <node index="1">
            <host>node1</host>
            <port>2181</port>
        </node>
        <node index="2">
            <host>node2</host>
            <port>2181</port>
        </node>
        <node index="3">
            <host>node3</host>
            <port>2181</port>
        </node>
    </zookeeper>
</yandex>
```

### 实例 2 配置

```bash
cp -a /clickhouse/instance1/config/* /clickhouse/instance2/config
sed -i 's/instance1/instance2/g' /clickhouse/instance2/config/config.xml
sed -i 's#<replica>1</replica>#<replica>2</replica>#' /clickhouse/instance2/config/macros.xml
sed -i 's/8123/8124/;s/9001/9002/;s/9004/19004/;s/9005/19005/' /clickhouse/instance2/config/port.xml
```

### 实例 2 配置

```bash
cp -a /clickhouse/instance1/config/* /clickhouse/instance3/config
sed -i 's/instance1/instance3/g' /clickhouse/instance3/config/config.xml
sed -i 's#<replica>1</replica>#<replica>3</replica>#' /clickhouse/instance2/config/macros.xml
sed -i 's/8123/8125/;s/9001/9003/;s/9004/29004/;s/9005/29005/' /clickhouse/instance2/config/port.xml
```

## 启动 ClickHouse 实例

```bash
/usr/bin/clickhouse-server --daemon --config-file=/clickhouse/instance1/config/config.xml
/usr/bin/clickhouse-server --daemon --config-file=/clickhouse/instance2/config/config.xml
/usr/bin/clickhouse-server --daemon --config-file=/clickhouse/instance3/config/config.xml
```

## 创建复制表以及分布式表进行验证

### 创建复制表

```sql
CREATE DATABASE IF NOT EXISTS dev ON CLUSTER testcluster;
CREATE TABLE dev.test_replicated on cluster testcluster
(
    id UInt32,
    value String,
    timestamp DateTime
)
ENGINE = ReplicatedReplacingMergeTree('/clickhouse/tables/{shard}/test_replicated', '{replica}', timestamp)
ORDER BY id;
```

### 创建分布式表

```sql
CREATE TABLE dev.test_ds_table
(
    id UInt32,
    value String,
    timestamp DateTime
)
ENGINE = Distributed('testcluster', 'dev', 'test_replicated');
```

### 插入数据

```sql
insert into dev.test_replicated values (1, 'test', now());
```

### 验证

```sql
```sql
SELECT *
FROM dev.test_replicated

Query id: 68f9117c-105b-49a2-831e-864ec5954f70

┌─id─┬─value─┬───────────timestamp─┐
│  1 │ test  │ 2025-02-01 20:03:19 │
└────┴───────┴─────────────────────┘

1 row in set. Elapsed: 0.003 sec.


select * from dev.test_ds_table;

SELECT *
FROM dev.test_ds_table

Query id: 7733ea97-d70c-4122-9416-2fb34f7c1f7a

┌─id─┬─value─┬───────────timestamp─┐
│  1 │ test  │ 2025-02-01 20:03:19 │
└────┴───────┴─────────────────────┘

1 row in set. Elapsed: 0.004 sec.
```

## 集群信息

```sql
SELECT *
FROM system.clusters
WHERE cluster = 'testcluster'

Query id: 5cd714df-3a5f-4eeb-975e-572c00d43c29


Row 1:
──────
cluster:                 testcluster
shard_num:               1
shard_weight:            1
replica_num:             1
host_name:               127.0.0.1
host_address:            127.0.0.1
port:                    9001
is_local:                1
user:                    default
default_database:
errors_count:            0
slowdowns_count:         0
estimated_recovery_time: 0
database_shard_name:
database_replica_name:
is_active:               ᴺᵁᴸᴸ

Row 2:
──────
cluster:                 testcluster
shard_num:               1
shard_weight:            1
replica_num:             2
host_name:               127.0.0.1
host_address:            127.0.0.1
port:                    9002
is_local:                0
user:                    default
default_database:
errors_count:            0
slowdowns_count:         0
estimated_recovery_time: 0
database_shard_name:
database_replica_name:
is_active:               ᴺᵁᴸᴸ

Row 3:
──────
cluster:                 testcluster
shard_num:               1
shard_weight:            1
replica_num:             3
host_name:               127.0.0.1
host_address:            127.0.0.1
port:                    9003
is_local:                0
user:                    default
default_database:
errors_count:            0
slowdowns_count:         0
estimated_recovery_time: 0
database_shard_name:
database_replica_name:
is_active:               ᴺᵁᴸᴸ

3 rows in set. Elapsed: 0.003 sec.
```
