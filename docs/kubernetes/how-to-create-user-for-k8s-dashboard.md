---
title: 在 Kubernetes Dashboard 创建基于 RBAC 的授权用户
description: 这篇帖子描述了如何自动创建基于 RBAC 的授权用胡，可以访问 Kubernetes Dashboard
tags: ["kubernetes", "dashboard", "rbac"]
---

# 在 Kubernetes Dashboard 创建基于 RBAC 的授权用户

Kuberenetes [Dashboard](https://github.com/kubernetes/dashboard) 默认情况下，创建了一个 Admin 角色的用户，属于超级管理员。

我们希望在创建一些普通用户，基于 namespace 进行划分，每个创建的用户只能在指定的 namespace 下进行一些执行的操作，下面是步骤


## 基于 namespace 内的用户创建

以下的操作是将创建的用户限定在指定的 namespace 下，集群资源以及其他 namespace 资源不可见

###  在 namespace 上创建 ServiceAccount 资源

```shell
kubectl create sa staff-user -n staff-center
```

### 创建用户角色

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: staff-center
  name: role-staff
rules:
- apiGroups: [""]
  resources: ["pods","services","deployments", "pods/log","ingresses", "endpoints", "statefulsets", "jobs","cronjobs","daemonsets"]
  verbs: ["get", "watch", "list"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["get", "watch", "list","create"]
```

保存为 `role-staff-user.yaml`

### 创建角色绑定

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: role-bind-staff
  namespace: staff-center
subjects:
- kind: ServiceAccount
  name: staff-user
  namespace: staff-center
roleRef:
  kind: Role
  name: role-staff
  apiGroup: rbac.authorization.k8s.io
```

保存为 `role-binding-staff-user.yaml`

### 执行上面个配置文件

```shell
kubectl apply -f role-staff-user.yaml
kubectl apply -f role-binding-staff-user.yaml
```

###  获得该用户的 token

```shell
## 1. get secret 
secpod=$(kubectl get sa -n staff-center staff-user -o jsonpath='{.secrets[0].name}')
## 2. get token
token=$(kubectl get secret -n staff-center ${secpod} -o jsonpath='{.data.token}' |base64 -d)
echo "your login token: ${token}"
```

将脚本保存为 `get-token.sh`， 然后执行，可以获得当前用户的 `token`，用这个 token 就可以登录到 Dashboard。

### 获取可以配置的资源列表

在创建角色的配置文件中，如何知道当前有哪些资源可以用于 `resources` 的配置呢？我们可以直接查询 `api-resources` 即可获得，如下：

```shell
[root@master dashboard]# kubectl api-resources --sort-by=name
NAME                              SHORTNAMES   APIVERSION                             NAMESPACED   KIND
apiservices                                    apiregistration.k8s.io/v1              false        APIService
apiservices                                    management.cattle.io/v3                false        APIService
apps                                           catalog.cattle.io/v1                   true         App
authconfigs                                    management.cattle.io/v3                false        AuthConfig
bgpconfigurations                              crd.projectcalico.org/v1               false        BGPConfiguration
bgppeers                                       crd.projectcalico.org/v1               false        BGPPeer
bindings                                       v1                                     true         Binding
blockaffinities                                crd.projectcalico.org/v1               false        BlockAffinity
caliconodestatuses                             crd.projectcalico.org/v1               false        CalicoNodeStatus
certificatesigningrequests        csr          certificates.k8s.io/v1                 false        CertificateSigningRequest
clusterflows                                   logging.banzaicloud.io/v1beta1         true         ClusterFlow
clusterinformations                            crd.projectcalico.org/v1               false        ClusterInformation
clusteroutputs                                 logging.banzaicloud.io/v1beta1         true         ClusterOutput
clusterregistrationtokens                      management.cattle.io/v3                true         ClusterRegistrationToken
clusterrepos                                   catalog.cattle.io/v1                   false        ClusterRepo
clusterrolebindings                            rbac.authorization.k8s.io/v1           false        ClusterRoleBinding
clusterroles                                   rbac.authorization.k8s.io/v1           false        ClusterRole
clusters                                       management.cattle.io/v3                false        Cluster
componentstatuses                 cs           v1                                     false        ComponentStatus
.....
```
上面的输出，你只要关心 `NAME` 这一列就可以了

## 设定集群资源访问权限

假定你还希望用户可以访问一些指定的集群资源，比如上面创建的用户，登录后，并不能直接跳转他对应的 namespace 上，而且在 namespace 处进行搜索，也不会给结果
namespace 本身属于集群资源，我们可以通过设置集群角色以及绑定集群角色来让用户可以列出当前已有的 namespace ，方便用户选择

### 创建集群角色

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  # "namespace" 被忽略，因为 ClusterRoles 不受名字空间限制
  name: cluster-role-staff-user
rules:
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "watch", "list"]
```

保存为 `cluster-role-staff-user.yaml`

### 创建集群角色绑定

```yaml
apiVersion: rbac.authorization.k8s.io/v1
# 此集群角色绑定允许 “manager” 组中的任何人访问任何名字空间中的 secrets
kind: ClusterRoleBinding
metadata:
  name: staff-user-read-namespaces
subjects:
- kind: ServiceAccount
  name: staff-user
  namespace: staff-center
roleRef:
  kind: ClusterRole
  name: cluster-role-staff-user
  apiGroup: rbac.authorization.k8s.io
```

保存为 `cluster-role-binding-staff-user.yaml`

### 执行这两个配置文件

```shell
kubectl apply -f cluster-role-staff-user.yaml
kubectl apply -f cluster-role-binding-staff-user.yaml
```

重新登录或刷新页面，就可以在 namespace 处看到当前已有的 namespace 了。

## 修改 token 的有效时常

默认情况下, dashboard 的 token 有效期是 5 分钟，这个时间有点短，我们可以增加，比如我们想增加到 8 小时

执行下面的命令

```shell
kubectl edit deployment  -n kubernetes-dashboard  kubernetes-dashboard
```
然后找到 

```yaml
spec:
  containers:
  - args:
    - --auto-generate-certificates
    - --enable-insecure-login
```
这个位置，增加一个 `token-ttl` 参数，编辑后，如下：

```yaml
yaml
spec:
  containers:
  - args:
    - --auto-generate-certificates
    - --enable-insecure-login
    - --token-ttl=28800
```

保存退出，等待一会，然后重新登录


## 参考文档

- https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac/
- https://github.com/kubernetes/dashboard/issues/2882