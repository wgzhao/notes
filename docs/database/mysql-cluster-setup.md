---
title: MySQL Cluster 配置
description: 这篇帖子描述了如何快速创建一个基于 MySQL NDB 集群
tags: ["mysql", "ndb", "cluster"]
---

# MySQL Cluster 配置

## 准备服务器

计划建立有5个节点的MySQL CLuster体系，需要用到5台服务器，但是我们做实验时没有这么多机器，可以只用3台，提供5个节点的MySQL CLuster体系，将SQL节点和数据节点共用一台机器，具体如下。

| 主机名   | 节点     | 对应的IP和端口   |
| -------- | -------- | ---------------- |
| DB-mgm   | 管理节点 | 10.10.6.201:1186 |
| DB-node1 | 数据节点 | 10.10.6.211:2202 |
| DB-node1 | SQL节点  | 10.10.6.211:3306 |
| DB-node2 | 数据节点 | 10.10.6.212:2202 |
| DB-node2 | SQL节点  | 10.10.6.212:3306 |


## 准备软件包

现在的mysql提供了一个专门作集群的安装包，这样就不用一个个的下载所需要的工具了。从官网下载。
<http://cdn.mysql.com/Downloads/MySQL-Cluster-7.4/mysql-cluster-gpl-7.4.5-linux-glibc2.5-x86_64.tar.gz>

# 安装
## 数据节点和SQL节点（各两台）
### 添加mysql用户和组，这是必需的。

```shell
[root@DB-node1 ~]# groupadd mysql
[root@DB-node1 ~]# useradd -g mysql mysql
[root@DB-node1 ~]# 
```

### 开始安装，下载的版本是免编译的，复制过来就可以用了。

```shell
[root@DB-node1 ~]# http://cdn.mysql.com/Downloads/MySQL-Cluster-7.4/mysql-cluster-gpl-7.4.5-linux-glibc2.5-x86_64.tar.gz 
[root@DB-node1 ~]# tar -zxvf mysql-cluster-gpl-7.4.5-linux-glibc2.5-x86_64.tar.gz -C /usr/local/
[root@DB-node1 ~]# ln -sv /usr/local/mysql-cluster-gpl-7.4.5-linux-glibc2.5-x86_64/ /usr/local/mysql
[root@DB-node1 ~]#
```

### 修改目录权限，这也是必需的，不然第四步会报错的。

```shell
[root@DB-node1 ~]# chown -R mysql.root /usr/local/mysql
[root@DB-node1 ~]# chown -R mysql.root /usr/local/mysql/
[root@DB-node1 ~]# 
```

### 安装初始的数据库表

```shell
[root@DB-node1 ~]# /usr/local/mysql/scripts/mysql_install_db --user=mysql
FATAL ERROR: Could not find ./bin/my_print_defaults
If you compiled from source, you need to run 'make install' to
copy the software into the correct location ready for operation.
If you are using a binary release, you must either be at the top
level of the extracted archive, or pass the --basedir option
pointing to that location.
```

如果出现上面错误，说明没有指定mysql目录，需手工指定。
解决方法

```shell
[root@DB-node1 ~]# /usr/local/mysql/scripts/mysql_install_db --user=mysql --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data/
```

### 设置mysql服务为开机自启动并启动

```shell
[root@DB-node1 ~]# cp /usr/local/mysql/support-files/mysql.server /etc/rc.d/init.d/mysqld
[root@DB-node1 ~]# chmod +x /etc/rc.d/init.d/mysqld 
[root@DB-node1 ~]# chkconfig --add mysqld 
[root@DB-node1 ~]# chkconfig --list mysqld        
mysqld          0:off   1:off   2:on    3:on    4:on    5:on    6:off
[root@DB-node1 ~]#
[root@DB-node1 ~]# service mysqld start
```

到这一步，在数据节点和SQL节点上就算安装好了。

## 管理节点

管理节点的安装更简单，只要在数据节点和SQL节点服务器上复制些文件出来就行了，虽然只有一步，

```shell
[root@DB-mgm ~]# scp 10.10.6.211:/usr/local/mysql/bin/ndb_mgm* /usr/local/bin/
[root@DB-mgm ~]# ll /usr/local/bin/ndb*
-rwxr-xr-x 1 root root  7137184 Apr 13 09:45 /usr/local/bin/ndb_mgm
-rwxr-xr-x 1 root root 16336025 Apr 13 09:45 /usr/local/bin/ndb_mgmd
[root@DB-mgm ~]# 
```

管理节点只要ndb_mgm和ndb_mgmd两个文件和一个配置文件即可，因此把这三个文件复制到那里，那里就是管理节点了。ndb_mgmd是mysql cluster管理服务器，ndb_mgm是客户端管理工具，等一下会用到它们的。到目前为止两个SQL节点两个数据节点和一个管理节点都安装完成了，但是还不能工作，得进行配置，把这几个节点联系在一起协同工作。

# 配置
## 数据节点和SQL节点（各两台）
mysql服务启动时会默认加载/etc/my.cnf作为其配置文件，要将一个mysql服务器配置成一个数据节点和SQL节点也非常的简单：
只要在内容结尾加上下面4行就将这个mysql服务器变成了一个数据节点和SQL节点。

```shell
ndbcluster                             \\运行NDB存储引擎
ndb-connectstring=10.10.6.201          \\指定管理节点  这两行声明其为SQL节点
[mysql_cluster]
ndb-connectstring=10.10.6.201          \\指定管理节点  这两行声明其为数据节点
```

新建mysql的配置文件

```shell
[root@DB-node1 ~]# vi /etc/my.cnf
[root@DB-node1 ~]# more /etc/my.cnf 
[client]
port=3306
socket=/usr/local/mysql/mysql.sock
[mysqld]
basedir=/usr/local/mysql/
datadir=/usr/local/mysql/data
user= mysql
pid-file=/usr/local/mysql/mysqld.pid
log-error=/usr/local/mysql/mysqld.err
```

以下四行为新增加内容

```shell
ndbcluster                             \\运行NDB存储引擎
ndb-connectstring=10.10.6.201          \\指定管理节点  这两行声明其为SQL节点
[mysql_cluster]
ndb-connectstring=10.10.6.201          \\指定管理节点  这两行声明其数据节点
```

注意两台服务器都得这样配置。

## 管理节点

管理节点的配置复杂一点，在管理服务器`/usr/local/mysql/`目录中创建`config.ini`文件。
网上说 使用`mysql-cluster`，但是启动时 总时有如下提示：

```shell
[root@DB-mgm ~]# ndb_mgmd -f /usr/local/mysql-cluster/config.ini 
MySQL Cluster Management Server mysql-5.6.23 ndb-7.4.5
2015-04-13 14:13:28 [MgmtSrvr] INFO     -- The default config directory '/usr/local/mysql/mysql-cluster' does not exist. Trying to create it...
Failed to create directory '/usr/local/mysql/mysql-cluster', error: 2
2015-04-13 14:13:28 [MgmtSrvr] ERROR    -- Could not create directory '/usr/local/mysql/mysql-cluster'. Either create it manually or specify a different directory with --configdir=<path>
```

最后改成了`/usr/local/mysql` 就可以了

```ini
[root@DB-mgm ~]# mkdir /usr/local/mysql
[root@DB-mgm ~]# vi /usr/local/mysql/config.ini
[root@DB-mgm ~]# more /usr/local/mysql/config.ini 
[NDBD DEFAULT]
NoOfReplicas=1           \\每个数据节点的镜像数量
DataMemory=500M          \\每个数据节点中给数据分配的内存
IndexMemory=300M         \\每个数据节点中给索引分配的内存
[TCP DEFAULT]
portnumber=2202          \\数据节点的默认连接端口
[NDB_MGMD]               \\配置管理节点
hostname=10.10.6.201
datadir=/usr/local/mysql/            \\管理节点数据(日志)目录
[NDBD]
hostname=10.10.6.211                  \\数据节点配置
datadir=/usr/local/mysql/data/        \\数据节点目录
[NDBD]
hostname=10.10.6.212
datadir=/usr/local/mysql/data/
[MYSQLD]                             \\SQL节点目录
hostname=10.10.6.211
[MYSQLD]
hostname=10.10.6.212
```

-  `[NDBD DEFAULT]`：表示每个数据节点的默认配置在每个节点的`[NDBD]`中不用再写这些选项，只能有一个。 
- `[NDB_MGMD]`：表示管理节点的配置，只有一个。 
- `[NDBD]`：表示每个数据节点的配置，可以有多个。 
- `[MYSQLD]`：表示SQL节点的配置，可以有多个，分别写上不同SQL节点的IP地址，也可以什么都不写，只保留一个空节点，表示任意一个IP地址都可以进行访问，此节点的个数表明了可以用来连接数据节点的SQL节点总数。

# 启动
## 管理节点
### 启动管理节点

mysql cluster 需要各个节点都 进行启动后才可以工作，节点的启动顺序为管理节点->数据节点->SQL节点。首先启动管理节点

```shell
[root@DB-mgm ~]# ndb_mgmd -f /usr/local/mysql/config.ini         
MySQL Cluster Management Server mysql-5.6.23 ndb-7.4.5
2015-04-13 14:14:14 [MgmtSrvr] INFO     -- The default config directory '/usr/local/mysql/mysql-cluster' does not exist. Trying to create it...
2015-04-13 14:14:14 [MgmtSrvr] INFO     -- Sucessfully created config directory
2015-04-13 14:14:14 [MgmtSrvr] WARNING  -- at line 7: [TCP] portnumber is deprecated
[root@DB-mgm ~]#
[root@DB-mgm ~]# echo "/usr/local/bin/ndb_mgmd -f /usr/local/mysql/config.ini "  >>/etc/rc.local
[root@DB-mgm ~]#
```

命令行中的ndb_mgmd是mysql cluster的管理服务器，后面的-f表示后面的参数是启动的参数配置文件。如果在启动后过了几天又添加了一个数据节点，这时修改了配置文件启动时就必须加上--initial参数，不然添加的节点不会作用在mysql cluster中。

`./ndb_mgmd -f /var/lib/mysql-cluster/config.ini --initial`

启动时可能会报个`WARNING,如WARNING  -- at line 7: [TCP] portnumber is deprecated`，这个不用管。可以正常工作的。

### 查看有无启动成功

```shell
[root@DB-mgm ~]# ls /usr/local/mysql/
config.ini  mysql-cluster  ndb_1_cluster.log  ndb_1_out.log  ndb_1.pid
[root@DB-mgm ~]# 
[root@DB-mgm ~]# netstat -tlnp | grep 1186
tcp        0      0 0.0.0.0:1186                0.0.0.0:*                   LISTEN      3075/ndb_mgmd       
[root@DB-mgm ~]# ndb_mgm
-- NDB Cluster -- Management Client --
ndb_mgm> show
Connected to Management Server at: localhost:1186
Cluster Configuration
---------------------
[ndbd(NDB)]     2 node(s)
id=2 (not connected, accepting connect from 10.10.6.211)
id=3 (not connected, accepting connect from 10.10.6.212)
[ndb_mgmd(MGM)] 1 node(s)
id=1    @10.10.6.201  (mysql-5.6.23 ndb-7.4.5)
[mysqld(API)]   2 node(s)
id=4 (not connected, accepting connect from 10.10.6.211)
id=5 (not connected, accepting connect from 10.10.6.212)
ndb_mgm> exit
[root@DB-mgm ~]#
```

## 数据节点（两台）

安装后第一次启动数据节点时要加上--initial参数，其它时候不要加，除非是在备份、恢复或配置变化后重启时。

```shell
[root@DB-node1 ~]# /usr/local/mysql/bin/ndbd --initial
2015-04-13 14:49:59 [ndbd] INFO     -- Angel connected to '10.10.6.201:1186'
2015-04-13 14:49:59 [ndbd] INFO     -- Angel allocated nodeid: 2
[root@DB-node1 ~]# netstat -tlnp |grep 2202
tcp        0      0 10.10.6.211:2202            0.0.0.0:*                   LISTEN      14780/ndbd          
[root@DB-node1 ~]#    
[root@DB-node1 ~]# echo /usr/local/mysql/bin/ndbd >>/etc/rc.local
[root@DB-node1 ~]# 
```

## SQL节点（两台）

```shell
[root@DB-node1 ~]# service  mysqld start
Starting MySQL SUCCESS! 
[root@DB-node1 ~]# 
```

## 客户端管理

这时就进入到客户端或管理节点，可以对mysql cluster进行各项操作，进入后会有ndb_mgm > 提示符出现，首先来查看各节点的连接情况，在`ndb_mgm>` 提示符下输入`show`：

```shell
[root@DB-mgm ~]# ndb_mgm
-- NDB Cluster -- Management Client --
ndb_mgm> show
Connected to Management Server at: localhost:1186
Cluster Configuration
---------------------
[ndbd(NDB)]     2 node(s)
id=2    @10.10.6.211  (mysql-5.6.23 ndb-7.4.5, Nodegroup: 0, *)
id=3    @10.10.6.212  (mysql-5.6.23 ndb-7.4.5, Nodegroup: 1)
[ndb_mgmd(MGM)] 1 node(s)
id=1    @10.10.6.201  (mysql-5.6.23 ndb-7.4.5)
[mysqld(API)]   2 node(s)
id=4    @10.10.6.211  (mysql-5.6.23 ndb-7.4.5)
id=5    @10.10.6.212  (mysql-5.6.23 ndb-7.4.5)
ndb_mgm> exit
```

可以看到各个节点已经连接上了，至此，mysql cluster配置完成。

## 关闭

mysql cluster的关闭也很简单，只需在`ndb_mgm>` 提示符下输入 `shutdown`即可，这时会显示各节点的关闭信息，再输入exit即可退出ndb_mgm管理，回到shell中。虽然mysql cluster 关闭了，但是SQL节点的mysql服务并不会停止的。接下来就可以做各种试验了。

# 测试
## 在一个SQL节点新建一个测试数据库，jedy_db

```sql
mysql> create database jedy_db;
Query OK, 1 row affected (0.10 sec)
[DB-node1] mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| jedy_db            |
| mysql              |
| ndb_2_fs           |
| ndbinfo            |
| performance_schema |
| test               |
+--------------------+
7 rows in set (0.05 sec)
```

## 在另一个SQL节点查看

```sql
[DB-node2] mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| jedy_db            |
| mysql              |
| ndb_3_fs           |
| ndbinfo            |
| performance_schema |
| test               |
+--------------------+
7 rows in set (0.06 sec)
```

所有配置完成