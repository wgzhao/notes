---
title: HAProxy + Percona XtraDB Cluster 部署
description: 这篇帖子描述了，如何安装 Percona XtraDB 集群，并利用 HAProxy 工具来做到负载均衡
tags: ["percona", "xtradb", "mysql", "cluster", "haproxy"]
---

# HAProxy + Percona XtraDB Cluster 部署

## 环境说明

假定Percona XtraDB Cluster(以下简称PXC)有4个节点，分别为：  

- 192.168.1.75
- 192.168.1.77
- 192.168.1.78
- 192.168.1.81

HAProxy 部署在`192.168.1.97`节点上。

## 配置haproxy.cfg

在`192.168.1.97`上安装HAProxy后，需要配置`/etc/haproxy/haproxy.cfg`文件，内容如下：


```
--8<-- "haproxy.cfg"
```

## 配置数据库检测脚本

需要在每个PXC节点上配置数据库检测脚本，该脚本的状态以HTTP的方式发布出去。
在上述四个PXC节点上，需要安装下面的两个脚本：

- mysqlchk
- clustercheck

`mysqlchk`是一个`xinetd`守护进程配置文件，默认位置为`/etc/xinetd.d`目录，内容如下：

```
--8<-- "mysqlchk"
```

创建`/var/log/clustercheck.log`文件，并设置属主为上述配置文件中指定的`nobody`。

在`/etc/services`文件增加对`9200`端口的服务描述，类似下面一行:
mysqlcheck  9200/tcp     #MySQL Status Check

上述配置文件中提到的`/usr/local/sbin/clustercheck`文件，来源于￼￼

在任意一个PXC节点上（比如是`192.168.1.75`）创建上述配置文件提供的数据库状态检测账号

```sql
mysql> grant process on *.* to 'clustercheck'@'localhost' identified by 'clustercheckpassword!';
Query OK, 0 rows affected (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)
```

在PXC四个节点上，启动`xinetd`服务：  

`service xinetd start`

查看9200端口是否已经打开。

## 启动HAProxy

上述过程完成后，就可以启动`HAProxy`了：   
`service haproxy start`

如果没有报错，则可以通过浏览器访问 `http://192.168.1.97:8080/haproxy/stats`，输入配置文件中设置账号密码，就可以看到HAProxy的状态了。