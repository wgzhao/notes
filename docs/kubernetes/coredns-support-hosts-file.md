---
title: CoreDNS 支持解析宿主机的 hosts 文件
description: 这篇帖子讲述通过配置，使得 kubernetes 的 DNS 组件 CoreDNS 支持宿主机的 hosts 文件的静态名称映射
tags: ["kubernetes", "coredns"]
---

# CoreDNS 支持解析宿主机的 hosts 文件

默认情况下， [CoreDNS](https://coredns.io/) 不支持节点的 `/etc/hosts` 文件里的静态域名解析。需要修改 `coredns` 的 `configmap` 配置，病在 `coredns` 增加节点 `/etc/hosts` 的挂载。

登录任意一台控制平面主机，进行下面的操作


## 修改coredns的configMap

执行下面的命令

```shell
kubectl -n kube-system edit configmap coredns
```

在 `health` 内容的后面增加如下内容：

```
hosts /etc/add_hosts {
    fallthrough
}
```

完整的内容如下：

```yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        hosts /etc/add_hosts {
           fallthrough
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  creationTimestamp: "2022-01-26T03:29:22Z"
  name: coredns
  namespace: kube-system
  resourceVersion: "227"
  uid: 5a1bacdf-0f57-4306-95a0-9f3fe967a921
```

## 修改coredns的deployment

执行下面的命令：

```shell
kubectl edit deployment -n kube-system coredns
```

在 `volumeMounts:` 这一行下增加如下内容：

```yaml
        - mountPath: /etc/add_hosts
          name: add-hosts
```

在 `volumes:` 这一行下增加如下内容：

```yaml
      - hostPath:
          path: /etc/hosts
          type: ""
        name: add-hosts
```

完整内容如下：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: "2022-01-26T03:29:22Z"
  generation: 1
  labels:
    k8s-app: kube-dns
  name: coredns
  namespace: kube-system
  resourceVersion: "640"
  uid: b1cf8b75-e995-43fe-8117-858e8e086e3d
spec:
  progressDeadlineSeconds: 600
  replicas: 2
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kube-dns
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        k8s-app: kube-dns
    spec:
      containers:
      - args:
        - -conf
        - /etc/coredns/Corefile
        image: k8s.gcr.io/coredns/coredns:v1.8.6
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 5
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 5
        name: coredns
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
         name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /ready
            port: 8181
            scheme: HTTP
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          readOnlyRootFilesystem: true
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /etc/coredns
          name: config-volume
          readOnly: true
        - mountPath: /etc/add_hosts
          name: add-hosts
      dnsPolicy: Default
      nodeSelector:
        kubernetes.io/os: linux
      priorityClassName: system-cluster-critical
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      serviceAccount: coredns
      serviceAccountName: coredns
      terminationGracePeriodSeconds: 30
      tolerations:
      - key: CriticalAddonsOnly
        operator: Exists
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
      - effect: NoSchedule
        key: node-role.kubernetes.io/control-plane
      volumes:
      - configMap:
          defaultMode: 420
          items:
          - key: Corefile
            path: Corefile
          name: coredns
        name: config-volume
      - hostPath:
          path: /etc/hosts
          type: ""
        name: add-hosts
status:
  availableReplicas: 2
  conditions:
  - lastTransitionTime: "2022-01-26T03:29:55Z"
    lastUpdateTime: "2022-01-26T03:29:55Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: "2022-01-26T03:29:37Z"
    lastUpdateTime: "2022-01-26T03:29:55Z"
    message: ReplicaSet "coredns-64897985d" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  observedGeneration: 1
  readyReplicas: 2
  replicas: 2
  updatedReplicas: 2
```

## 验证

在所有 k8s 节点的 `/etc/hosts` 文件里增加一条 IP 地址映射，然后登录任意一个 pod，
然后 ping 添加到 `/etc/hosts` 的映射主机，看是否解析成刚才增加的 IP 地址。

