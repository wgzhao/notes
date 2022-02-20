# zookeeper集群应对万级并发的调优

[Source](https://codeleading.com/article/77225589233/ "Permalink to zookeeper集群应对万级并发的调优")

* tickTime，这个参数叫各节点前心跳保持的频率，你即不能太高也不能太低，太高了各zk节点间万一有一个挂了那么整个zk的master群来不及选举出来了就会影响到整人本业务。如果太低了，那么zk群间因为频繁心跳而导致网络开销过大；
* initLimit，这个值是这样的，要看真实的并发连接的。类似这种initXXX值有一个通则，那就是理论上要把它设成和maxXXX一样，大家设想一下，一开始你设成1，然后整个connection pool发觉不够了开始+1，+1操作，这种“+1”操作是有系统开销的，它会影响整体系统的性能、吐吞、平均响应时间。但是这个通则也不是一成不变，如果你开了多会造成浪费，因此拿这边的案例来说，我们的pool有34个，每个启动都要去连zk，那么我们把这个最小值设成50是合理的，如果“弹性扩充了pool”后，那么再让它自增“1”；
* syncLimit，该参数有默认值5，即表示是参数tickTime值的5倍，必须配置，且需要配置一个正整数，不支持系统属性方式配置。 

  该参数用于配置Leader服务器和Follower之间进行心跳检测的最大延时时间。在ZooKeeper集群运行过程中，Leader服务器会与所有的Follower进行心跳检测来确定该服务器是否存活。如果Leader服务器在syncLimit时间内无法获取到Follower的心跳检测响应，那么Leader就会认为该Follower已经脱离了和自己的同步。因此这边我们认为30秒没有反馈，那么我们认为Leader和Follower间已经“挂”了；
* leaderServers=no，这个参数是相当有用的，在生产上是必开启的。它代表Leader的主机节点不接受连接，因为Leader是作为协调用的，而不是供对外服务它只是对内。它的设置会直接提升zk群的性能；
* maxClientCnxns，在压测环境建议设为1000，生产上以压测实际结果为准再做调整或者也不调整；
* forceSync=no，该参数有默认值：yes，可以不配置，可选配置项为“yes”和“no”，仅支持系统属性方式配置：zookeeper.forceSync。 

  该参数用于配置ZooKeeper服务器是否在事务提交的时候，将日志写入操作强制刷入磁盘（即调用java.nio.channels.FileChannel.force接口），默认情况下是“yes”，即每次事务日志写入操作都会实时刷入磁盘。如果将其设置为“no”，则能一定程度的提高ZooKeeper的写性能，但同时也会存在类似于机器断电这样的安全风险。在此处我是建议设成no，它会直接带来zk群的性能提升。我们的zk目前看来暂无事无提交，有一些拿zk做“tcc或者是2pC提交”的global transaction一类的是会用到zk事务的，我们的环境暂无因此此值设成no，增加其读写速度；

# zk1

```ini
# The number of milliseconds of each tick
#zk时间单元(毫秒)
tickTime=5000
# The number of ticks that the initial
# synchronization phase can take
# floower启动过程中从leader同部数据的时间限制
#如果集群规模大，数据量多的话，适当调大此参数
initLimit=50
# The number of ticks that can pass between
# sending a request and getting an acknowledgement
syncLimit=6
dataDir=/Users/chrishu123126.com/opt/zookeeper/data/zk1
dataLogDir=/Users/chrishu123126.com/opt/zookeeper/logs/zk1
clientPort=2181
server.1=localhost:2187:2887
server.2=localhost:2188:2888
server.3=localhost:2189:2889
#leader不接受客户端连接,专注于通信和选举等
leaderServes=no
maxClientCnxns=1000
forceSync=no
```

# zk2

```ini
# The number of milliseconds of each tick
#zk时间单元(毫秒)
tickTime=5000
# The number of ticks that the initial
# synchronization phase can take
# floower启动过程中从leader同部数据的时间限制
#如果集群规模大，数据量多的话，适当调大此参数
initLimit=50
# The number of ticks that can pass between
# sending a request and getting an acknowledgement
syncLimit=6
dataDir=/Users/chrishu123126.com/opt/zookeeper/data/zk2
dataLogDir=/Users/chrishu123126.com/opt/zookeeper/logs/zk2
clientPort=2182
server.1=localhost:2187:2887
server.2=localhost:2188:2888
server.3=localhost:2189:2889
#leader不接受客户端连接,专注于通信和选举等
leaderServes=no
maxClientCnxns=1000
forceSync=no
```

# zk3

```ini
# The number of milliseconds of each tick
#zk时间单元(毫秒)
tickTime=5000
# The number of ticks that the initial
# synchronization phase can take
# floower启动过程中从leader同部数据的时间限制
#如果集群规模大，数据量多的话，适当调大此参数
initLimit=50
# The number of ticks that can pass between
# sending a request and getting an acknowledgement
syncLimit=6
dataDir=/Users/chrishu123126.com/opt/zookeeper/data/zk3
dataLogDir=/Users/chrishu123126.com/opt/zookeeper/logs/zk3
clientPort=2183
server.1=localhost:2187:2887
server.2=localhost:2188:2888
server.3=localhost:2189:2889
#leader不接受客户端连接,专注于通信和选举等
leaderServes=no
maxClientCnxns=1000
forceSync=no
```


### zk群“四字”真言

1. `echo stat|nc 127.0.0.1 2181` 来查看哪个节点被选择作为follower或者leader
2. `echo ruok|nc 127.0.0.1 2181` 测试是否启动了该Server，若回复imok表示已经启动。
3. `echo dump| nc 127.0.0.1 2181` ,列出未经处理的会话和临时节点。
4. `echo kill | nc 127.0.0.1 2181` ,关掉server
5. `echo conf | nc 127.0.0.1 2181` ,输出相关服务配置的详细信息。
6. `echo cons | nc 127.0.0.1 2181` ,列出所有连接到服务器的客户端的完全的连接 / 会话的详细信息。
7. `echo envi |nc 127.0.0.1 2181` ,输出关于服务环境的详细信息（区别于 conf 命令）。
8. `echo reqs | nc 127.0.0.1 2181` ,列出未经处理的请求。
9. `echo wchs | nc 127.0.0.1 2181` ,列出服务器 watch 的详细信息。
10. `echo wchc | nc 127.0.0.1 2181` ,通过 session 列出服务器 watch 的详细信息，它的输出是一个与 watch 相关的会话的列表。
11. `echo wchp | nc 127.0.0.1 2181` ,通过路径列出服务器 watch 的详细信息。它输出一个与 session 相关的路径。