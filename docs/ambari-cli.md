---
title: 使用 Ambari API 来操作 Hadoop 集群
description: 这篇帖子主要描述一些 ambari 命令行操作的内容
tags: ["ambari", "cli"]
---


Ambari 管理的hadoop集群，大部分操作都可以在其管理页面上完成想要的操作，但是有些操作却不行，比如一些服务并不提供删除操作，另外当一个节点处于心跳丢失情况下，也无法在页面上将其从集群中剥离出来。

因此，有些特别的操作还是需要直接使用Amabri 提供的API 接口。

这里假定访问Ambari API所必要的几个参数

```bash
export AMBARI_URL=http://hadoop1.wgzhao.com:8080
export AMBARI_USER=admin
export AMBARI_PASS=admin
export AMBARI_CLUSTER=cluster
```

下面列出几个常用操作

## 查看某个节点上所安装的服务

```bash
curl -u ${AMBARI_USER}:${AMBARI_PASS} -H "X-Requested-By: ambari" -X GET \
  ${AMBARI_URL}/api/v1/clusters/${AMBARI_CLUSTER}/hosts/hadoop3.wgzhao.com
```

## 删除某一个节点上的指定服务或组件

```bash
curl -u ${AMBARI_USER}:${AMBARI_PASS}  -H "X-Requested-By: ambari" -X DELETE \
  ${AMBARI_URL}/api/v1/clusters/${AMBARI_CLUSTER}/hosts/hadoop3.wgzhao.com/host_components/JOURNALNODE
```

## 删除某个节点

注意，删除某一个节点之前，需要先删除节点上的所有服务

```bash
curl -u ${AMBARI_USER}:${AMBARI_PASS} -H "X-Requested-By: ambari" -X DELETE \
  ${AMBARI_URL}/api/v1/clusters/${AMBARI_CLUSTER}/hosts/hadoop3.wgzhao.com
```

## 增加节点到集群

```bash
curl -u ${AMBARI_USER}:${AMBARI_PASS} -H "X-Requested-By: ambari"  -X GET \
  ${AMBARI_URL}/api/v1/clusters/${AMBARI_CLUSTER}/hosts/hadoop25.wgzhao.com
```

## 新新增的节点上安装制定服务

```bash
curl -u ${AMBARI_USER}:${AMBARI_PASS}  -H "X-Requested-By: ambari" -X PUT -d '{"HostRoles": {"state": "INSTALLED"}}' \
  ${AMBARI_URL}/api/v1/clusters/${AMBARI_CLUSTER}/hosts/hadoop25.wgzhao.com/host_components/DATANODE
```
发出安装命令后，会给出类似以下的反馈信息

```json
{
  "href" : " ${AMBARI_URL}/api/v1/clusters/${AMBARI_CLUSTER}/requests/9",
  "Requests" : {
    "id" : 9,
    "status" : "InProgress"
  }
}
```

根据输出的ID，我们可以查询安装状态

```bash
curl -u ${AMBARI_USER}:${AMBARI_PASS}  -H "X-Requested-By: ambari" -X GET ${AMBARI_URL}/api/v1/clusters/${AMBARI_CLUSTER}/requests/9
curl -u ${AMBARI_USER}:${AMBARI_PASS} -H "X-Requested-By: ambari"  -X GET ${AMBARI_URL}/api/v1/clusters/${AMBARI_CLUSTER}/requests/9/tasks/101
 
{
  "href" : "${AMBARI_URL}/api/v1/clusters/${AMBARI_CLUSTER}/requests/9/tasks/101",
  "Tasks" : {
    ...
    "status" : "COMPLETED",
    ...
  }
}
```

## 启动刚安装的服务

```bash
curl -u ${AMBARI_USER}:${AMBARI_PASS} -H "X-Requested-By: ambari"  -X PUT -d  '{"HostRoles": {"state": "STARTED"}}' \
  ${AMBARI_URL}/api/v1/clusters/${AMBARI_CLUSTER}/hosts/hadoop25.wgzhao.com/host_components/DATANODE
```

## 停止 HDFS 服务

```bash
curl -u ${AMBARI_USER}:${AMBARI_PASS} -i -H 'X-Requested-By: ambari' -X PUT \
  -d '{"RequestInfo": {"context" :"Stop HDFS via REST"}, "Body": {"ServiceInfo": {"state": "INSTALLED"}}}' \
  ${AMBARI_URL}/api/v1/clusters/${AMBARI_CLUSTER}/services/HDFS
```

## 启动 HDFS 服务

```bash
curl -u ${AMBARI_USER}:${AMBARI_PASS} -i -H 'X-Requested-By: ambari' -X PUT \
  -d '{"RequestInfo": {"context" :"Start HDFS via REST"}, "Body": {"ServiceInfo": {"state": "STARTED"}}}' \
  ${AMBARI_URL}/api/v1/clusters/${AMBARI_CLUSTER}/services/HDFS
```