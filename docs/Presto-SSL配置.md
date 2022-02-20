# Presto SSL 配置

要启用Presto的认证和授权，前提是要配置Presto节点间的通讯为加密模式。

有关加密通讯更详细的说明和配置可以参考[官方文档](https://prestosql.io/docs/current/security/internal-communication.html)

这里仅记录在实际环境中如何配置，以下配置和分发使用[prestoadmin](https://github.com/prestodb/presto-admin)工具

注意以下操作均在配置有 `prestoadmin` 节点的 `.prestoadmin` 目录下操作

## 前提要求

所有节点主机名需要满足FQDN要求，也就是主机名加域名，我们生产环境的域名是 `bigdata.wgzhao.com`, 预发布环境为 `ds.wgzhao.com`

## 生成  Java Keystore File 文件

使用下面的命令生产

```shell
keytool -genkeypair -keyalg RSA -keysize 2048 -validity 365 \
-dname "CN=*.bigdata.wgzhao.com, OU=cfzq,O=com, L=Changsha, ST=Hunan, C=CN" \
-alias presto_prod -keypass JksPassword -keystore presto.jks -storepass JksPassword
```

这了要特别注意的是 `CN` 的配置，必须是泛域名方式，而且域名和当前节点的域名相匹配，否则会出现证书校验无法通过的错误，导致节点间通讯失败

## 将文件发布到所有节点

`presto-admin file copy ./presto.jks /etc/presto/`

## 配置 worker 节点

首先配置  `config.properties` 文件，文件内容修改类似如下：

```ini
coordinator=false
http-server.http.port=59999
discovery.uri=http://nn01.bigdata.wgzhao.com:59999
##SSL
internal-communication.shared-secret=UcGm5mjqgDPZ8ojxQ0c9pB
node.internal-address-source=FQDN
http-server.https.enabled=true
http-server.https.port=15999
http-server.https.keystore.path=/etc/presto/presto.jks
http-server.https.keystore.key=JksPassword
discovery.uri=https://nn01.bigdata.wgzhao.com:15999
internal-comunication.https.required=true
internal-communication.https.keystore.path=/etc/presto/presto.jks
internal-communication.https.keystore.key=JksPassword
http-server.https.secure-random-algorithm=SHA1PRNG
```
然后配置 `jvm.config` 文件，在最后增加一行

`-Djava.security.egd=file:/dev/urandom`

## 配置 Coordinator 节点

首先配置  `config.properties` 文件，文件内容修改类似如下：

```ini
node-scheduler.include-coordinator=false
discovery.uri=http://nn01.bigdata.wgzhao.com:59999
discovery-server.enabled=true
http-server.http.port=59999
coordinator=true
#query.max-memory-per-node=8GB
# auth setup
#http-server.authentication.type=PASSWORD

##SSL
internal-communication.shared-secret=UcGm5mjqgDPZ8ojxQ0c9pB
node.internal-address-source=FQDN
http-server.https.enabled=true
http-server.https.port=15999
http-server.https.keystore.path=/etc/presto/presto.jks
http-server.https.keystore.key=JksPassword
discovery.uri=https://nn01.bigdata.wgzhao.com:15999
internal-comunication.https.required=true
internal-communication.https.keystore.path=/etc/presto/presto.jks
internal-communication.https.keystore.key=JksPassword
http-server.https.secure-random-algorithm=SHA1PRNG
```

注意，这里我们依然启用了http协议，这是考虑到一些工具链接Presto在SSL方式下出现问题，后面会禁止http协议，仅启用https协议。

## 分发配置文件

执行下面的命令重新分发配置文件

`presto-admin configuration deploy`

## 重启服务

重启服务使得配置生效

`presto-admin server restart`

## 查看

访问 <https://nn01.bigdata.wgzhao.com:15999/ui> 可以看到通过HTTPS方式链接的Presto服务。
