# Phoenix 集成 HBase 并支持 Kerberos 认证

该文档用来简要描述当前生产环境下phoenix如何和hbase集成，并支持kerberos认证

## 降级

HDP 自带的hbase和phoenix因为与presto 不兼容，只能全部降级，其中 HBASE 降级到 1.4版本。phoenix降级到对应的 4.14.3 版本。

降级的简单流程如下：

1. 按正常方式通过Ambari安装HBASE
2. 删除每个节点上的 `/usr/hdp/<version>/hbase` 和 `/usr/hdp/<version>/phoenix` 目录
3. 从官方下载对应的软件，并复制到上述两个目录中（先创建目录）
4. `/usr/hdp/<version>/phoenix` 里需要创建如下软连接：
    ```shell
    # ln -sf phoenix-hive-4.14.3-HBase-1.4.jar phoenix-hive.jar
    # ln -sf phoenix-4.14.3-HBase-1.4-client.jar phoenix-client.jar
    # ln -sf phoenix-4.14.3-HBase-1.4-thin-client.jar phoenix-thin-client.jar
    # ln -sf phoenix-4.14.3-HBase-1.4-server.jar  phoenix-server.jar
    ```
5. `/usr/hdp/<version>/hbase` 里创建如下软连接
    ```shell
    ln -sf /usr/hdp/<version>/phoenix/phoenix-server.jar .
    ```
6. 增加 `/bin/hbase-config.sh` 的软连接
    ```shell
    ln -sf /usr/hdp/current/hbase-client/bin/hbase-config /bin/hbase-config.sh
    ```

## 增加namespace的映射配置

为了让phoenix能看到hbase已经存在的表，需要在hbase里增加如下配置(Ambari->HBase->Config->Custom hbase-site.xml)
```
phoenix.schema.mapSystemTablesToNamespace=true
phoenix.schema.isNamespaceMappingEnabled=true
```
重启hbase，让新配置生效

## 赋权

默认情况下，系统空间只有hbase账号才有权限，而我们希望hive账号也有权限，需要通过执行 `hbase shell` 命令后，执行以下命令 

```shell
hbase(main):008:0> grant 'hive', 'RWXCA'
0 row(s) in 1.0180 seconds
```

## 初始化

切换到 `hbase` 或 `hive` 账号，执行 `/usr/hdp/current/phoenix-client/bin/sqlline.py`

因为第一次要创建系统表，会比较慢，其输出如果和下面类似，则表示初始化成功：

```shell
$ /usr/hdp/current/phoenix-client/bin/sqlline.py
Setting property: [incremental, false]
Setting property: [isolation, TRANSACTION_READ_COMMITTED]
issuing: !connect jdbc:phoenix: none none org.apache.phoenix.jdbc.PhoenixDriver
Connecting to jdbc:phoenix:
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/usr/hdp/3.0.1.0-187/phoenix/phoenix-4.14.3-HBase-1.4-client.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/usr/hdp/3.0.1.0-187/hadoop/lib/slf4j-log4j12-1.7.25.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
Connected to: Phoenix (version 4.14)
Driver: PhoenixEmbeddedDriver (version 4.14)
Autocommit status: true
Transaction isolation: TRANSACTION_READ_COMMITTED
Building list of tables and columns for tab-completion (set fastconnect to true to skip)...
133/133 (100%) Done
Done
sqlline version 1.2.0
```