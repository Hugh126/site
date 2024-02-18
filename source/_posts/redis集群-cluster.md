---
title: redis集群-cluster
date: 2024-01-24 10:18:57
tags: redis
categories: 中间件
---
Redis的哨兵模式已经存在主备结构，并且能自动故障转移和动态扩容，为什么要整出cluster集群呢?  
哨兵模式只是一个Master节点的高可用，如果需要一堆节点的高可用呢？集群管理，故障发现，数据分片，这些问题自然应运而生。  
解决方案就是Cluster。
<!--more-->

### 特征

1. 多个 Redis 节点之间自动分片
2. 当小部分节点出现故障或无法与群集的其余部分通信时，集群可以继续运行

‍

### 端口

每个 Redis 集群节点都需要两个开放的 TCP 连接

* 一个用于为客户端提供服务的 Redis TCP 端口（6379）
* 集群总线端口（默认加上10000，如16379；作用：故障检测、配置更新、故障转移授权等）

‍

### 集群主备模型

集群每个节点，都有1个和N-1个副本构成。  
以基础的3主+3备为例：  
​![image](/images/assets/image-20240123142211-7frka5a.png)​


### 集群一致性

Redis 集群不保证强一致性，原因：  

1. 异步复制。 可以通过 WAIT 命令实现同步写入
2. 网络分区。客户端与少数实例隔离，客户端在主节点故障转移前（这个时间可配置<u>node timeout</u>）的写入都会丢失

‍

### 配置参数

* cluster-enabled `<yes/no>`​ ：如果是，则在特定 Redis 实例中启用 Redis 集群支持
* cluster-config-file `<filename>`​ ：集群节点在每次发生更改时自动保留集群配置（基本上是状态）的文件，用户不要编辑ta
* cluster-node-timeout `<milliseconds>`​ ：Redis 集群节点不可用的最长时间
* cluster-migration-barrier `<count>`​ ：主服务器将保持连接的最小副本数
* cluster-require-full-coverage `<yes/no>`​ ：如果设置为 yes，如果某个百分比的密钥空间未被任何节点覆盖，则停止写入
* luster-allow-reads-when-down `<yes/no>`​ ：默认否，当集群标记为失败时，是否允许可读

‍

#### 部署方式1（首次执行推荐）

1. 要创建集群，首先需要让几个空的 Redis 实例在集群模式下运行。

每个节点配置文件 redis.conf

```sh
port 7000
cluster-enabled yes
cluster-config-file nodes7000.conf
cluster-node-timeout 5000
appendonly yes
```

基于以上配置，创建一个启动6个节点的脚本（Maste或Slave，角色由集群分配）：

```sh
#!/bin/bash
if [ ! -d "clustertest" ]; then
    mkdir clustertest
fi
rm -rf ./clustertest/*
cd clustertest
for port in 7000 7001 7002 7003 7004 7005
do
    cp ../redis.conf redis${port}.conf
    sed -i "s#port 6379#port ${port}#g"  redis${port}.conf
    echo "cluster-enabled yes\ncluster-config-file nodes${port}.conf\ncluster-node-timeout 5000\nappendonly yes" >> redis${port}.conf
    nohup redis-server redis${port}.conf &
done
```

‍

2. 创建Redis Cluster集群，由集群为每个端口创建ID并分配节点角色(M/S)

    ```sh
    redis-cli --cluster create 127.0.0.1:7000 127.0.0.1:7001 \
    127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
    --cluster-replicas 1
    ```

‍

3. 创建完成反馈实例角色、槽位分配信息等

```sh
# redis-cli --cluster create 127.0.0.1:7000 127.0.0.1:7001 \
> 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
> --cluster-replicas 1
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 127.0.0.1:7004 to 127.0.0.1:7000
Adding replica 127.0.0.1:7005 to 127.0.0.1:7001
Adding replica 127.0.0.1:7003 to 127.0.0.1:7002
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: 9a7c5a97e34a57eee737166f54851edc27b80538 127.0.0.1:7000
   slots:[0-5460] (5461 slots) master
M: 61feac4042b4b3c3ab43ace2c85c45667c98ff8a 127.0.0.1:7001
   slots:[5461-10922] (5462 slots) master
M: b40862b42ebd1a3c40df1eabe1eef9c06026a39a 127.0.0.1:7002
   slots:[10923-16383] (5461 slots) master
S: b27d12f522eed9fbe3062b0d5be63598251b836a 127.0.0.1:7003
   replicates b40862b42ebd1a3c40df1eabe1eef9c06026a39a
S: a0883f0a14533b8fb5a3ddfe565330da089602b3 127.0.0.1:7004
   replicates 9a7c5a97e34a57eee737166f54851edc27b80538
S: a5f06fad49384780af55a938dd4aa553b48f7eaa 127.0.0.1:7005
   replicates 61feac4042b4b3c3ab43ace2c85c45667c98ff8a
Can I set the above configuration? (type 'yes' to accept): yes
...
[OK] All 16384 slots covered
```

这里集群将7000/7001/7002作为主节点，7004/7005/7003分别作为它们的备份，然后分别分配槽位[0-5460]/[5461-10922]/[10923-16383]。

之前提过任意节点的主备都会共享1个内部维护的配置文件，随便找一个看看它（nodes7000.conf）里面是什么：

```sh
b27d12f522eed9fbe3062b0d5be63598251b836a 127.0.0.1:7003@17003,,tls-port=0,shard-id=70775289706c7855eeb17d765170af0b26f5dca6 slave b40862b42ebd1a3c40df1eabe1eef9c06026a39a 0 1706000689000 3 connected   
a5f06fad49384780af55a938dd4aa553b48f7eaa 127.0.0.1:7005@17005,,tls-port=0,shard-id=49092a880937bcad546a472a5b04914d0c3d9fb0 slave 61feac4042b4b3c3ab43ace2c85c45667c98ff8a 0 1706000690000 2 connected   
61feac4042b4b3c3ab43ace2c85c45667c98ff8a 127.0.0.1:7001@17001,,tls-port=0,shard-id=49092a880937bcad546a472a5b04914d0c3d9fb0 master - 0 1706000689657 2 connected 5461-10922
b40862b42ebd1a3c40df1eabe1eef9c06026a39a 127.0.0.1:7002@17002,,tls-port=0,shard-id=70775289706c7855eeb17d765170af0b26f5dca6 master - 0 1706000690668 3 connected 10923-16383
a0883f0a14533b8fb5a3ddfe565330da089602b3 127.0.0.1:7004@17004,,tls-port=0,shard-id=f2820606ef71d32e5e0e4cda08f4622c98ab1635 slave 9a7c5a97e34a57eee737166f54851edc27b80538 0 1706000690972 1 connected   
9a7c5a97e34a57eee737166f54851edc27b80538 127.0.0.1:7000@17000,,tls-port=0,shard-id=f2820606ef71d32e5e0e4cda08f4622c98ab1635 myself,master - 0 1706000690000 1 connected 0-5460
vars currentEpoch 6 lastVoteEpoch 0
```

可以看到，这个配置文件内包含了集群所有子服务的ID、端口以及节点内的角色信息，如果是master还会带有对应slot信息。

基于此，可以得出结论：

	1. 任意节点都可以发现自己主不可用的时候，进行故障转移从而保障集群可用

	2. 新增节点的时候，集群可以为其分配合适角色，槽位变化后将对应槽位的key复制过去

‍

#### 部署方式2

使用 `utils/create-cluster`​ 脚本，其实这个脚本 的start函数就是去创建redis节点；create就是创建集群。

跟部署方式1的两步分别对应的。

1. create-cluster start

```sh
Starting 30001
Starting 30002
Starting 30003
Starting 30004
Starting 30005
Starting 30006
```

2. create-cluster create

```sh
M: 0d47125dcf264ca1e0e70feee5ba0bfb282a4f0f 127.0.0.1:30001
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 731db1e49b61a405a835c904ec9bf92ccb92c9ac 127.0.0.1:30006
   slots: (0 slots) slave
   replicates 0d47125dcf264ca1e0e70feee5ba0bfb282a4f0f
S: 7cccd9c0b4753fb67cc2b7f7dab221b80ee47e89 127.0.0.1:30004
   slots: (0 slots) slave
   replicates c0418e4d34fe627f2080ac4261e5dbc6f4a6d3c3
M: c0418e4d34fe627f2080ac4261e5dbc6f4a6d3c3 127.0.0.1:30002
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
M: dbac2f171108fdb22fb336a93febbbd46f6d22d8 127.0.0.1:30003
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: 71caec7e3a15a0a0bf4ffc59d8a45efd42361a01 127.0.0.1:30005
   slots: (0 slots) slave
   replicates dbac2f171108fdb22fb336a93febbbd46f6d22d8
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

3. create-cluster stop

    这个命令会停掉集群及所有实例

‍

### 集群交互

通过redis-cli连接到集群任一节点读写数据。  
可以看到key分配slot信息。如果不是本身连接节点，集群会帮忙重定向到对应节点，读写都是如此，并且客户端会跳转到该连接节点。  
`cluster info` 可以读取到集群的基本信息，这可以用于第三方监控。

```sh
127.0.0.1:7000> cluster info
# 集群状态
cluster_state:ok
# 与节点关联槽数量
cluster_slots_assigned:16384
# 映射到不处于FAIL或PFAI状态的槽的数量
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
# 集群中已知节点数
cluster_known_nodes:6
# 集群中Master节点数
cluster_size:3
...
127.0.0.1:7000> set abc 123
-> Redirected to slot [7638] located at 127.0.0.1:7001
OK
127.0.0.1:7001> set abcdef 123
-> Redirected to slot [15101] located at 127.0.0.1:7002
OK
127.0.0.1:7002> set abcdefhjk 123
OK
127.0.0.1:7002> get abc
-> Redirected to slot [7638] located at 127.0.0.1:7001
"123"
127.0.0.1:7001> get abcdef
-> Redirected to slot [15101] located at 127.0.0.1:7002
"123"
127.0.0.1:7002> get abcdefhjk
"123"
```

### Redis选举机制

Redis集群的选举机制是基于gossip协议的。  
当一个从节点发现自己的主节点不可用时，它会尝试进行 Failover，以便成为新的主节点。  
由于挂掉的主节点可能有多个从节点，因此存在多个从节点竞争成为主节点的过程。过程如下：  
1. 从节点发现自己的主节点不可用;
2. 从节点将记录集群的 currentEpoch（选举周期）加1，并广播 FAILOVER_AUTH_REQUEST 信息进行选举;
3. 其他节点收到 FAILOVER_AUTH_REQUEST 信息后，其他的主节点收到消息后返回 FAILOVER_AUTH_ACK 信息(对于同一个 Epoch，只能响应一次 ack);
4. 尝试failover的从节点收集主节点返回的 ack 消息;
5. 从节点判断收到大于半数的 ack 消息，成为新的主节点;
6. 广播 Pong 消息通知其他集群节点。

> 从节点并不是在主节点一进入 FAIL 状态就马上尝试发起选举，而是有一定延迟，一定的延迟确保我们等待FAIL状态在集群中传播，slave如果立即尝试选举，其它masters或许尚未意识到FAIL状态，可能会拒绝投票。  
延迟计算公式：  
```
# SLAVE_RANK表示此slave已经从master复制数据的总量的rank
 DELAY = 500ms + random(0 ~ 500ms) + SLAVE_RANK * 1000ms
```
Rank越小代表已复制的数据越新。理论上，持有最新数据的slave将会首先发起选举。

‍
