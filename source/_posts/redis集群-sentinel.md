---
title: redis集群-sentinel
date: 2024-01-19 21:22:58
tags: redis
categories: 中间件
---
redis哨兵模式实现了redis基本的高可用
<!--more-->
# 理解哨兵集群结构

[理解sentinel集群](https://redis.io/docs/management/sentinel/)  
一个典型的哨兵集群：1主(M)2从(R)  3哨兵(S)​​  
```sh
      +----+
       | M1 |
       | S1 |
       +----+
          |
+----+    |    +----+
| R2 |----+----| R3 |
| S2 |         | S3 |
+----+         +----+

Configuration: quorum = 2
```
故障转移检测配置：  
``` ini
# 至少 1 个副本同步写不成功
min-replicas-to-write 1
# 超过max-lag秒数的时间内没有向主发送异步确认
min-replicas-max-lag 10
```
如果主 M1 发生故障，S2 和 S3 将就故障达成一致，并能够授权故障转移，使客户端能够继续。  

# Redis及Sentinel配置

## Redis从节点配置
redis主节点配置不需要变更，从节点只需要声明是主节点的备份  
`replicaof 127.0.0.1 6379`  
在5.0或更早的版本中的命令是  
`slaveof 127.0.0.1 6379`  
> 注意每个redis节点占用不同端口  

## Sentinel配置
sentinel的配置文件为 sentinel.conf,以下列出其中的关键配置：    
``` ini
# 哨兵通信端口
port 26379
# 监控主节点名称/端口 以及多少个节点同意方可故障转移
sentinel monitor mymaster 127.0.0.1 6379 2  
# Sentinel认为它已关闭时，实例不应访问的时间
sentinel down-after-milliseconds mymaster 60000  
# 多长时间内不再对同一节点重试故障转移
sentinel failover-timeout mymaster 180000  
# 故障转移后，可以重新配置为同时使用新主服务器的副本数量
sentinel parallel-syncs mymaster 1
```
> 注意：多个


启动哨兵:  
`redis-sentinel /path/to/sentinel.conf`   或    
`redis-server /path/to/sentinel.conf --sentinel`


# 启动集群  
基于以上Redis及哨兵配置启动的知识，编辑启动Redis哨兵集群脚本：  
```sh
#!/bin/bash
# config 2 slave
cp redis.conf redis6380.conf
cp redis.conf redis6381.conf
sed -i "s#$port 6379#$port 6380#g"  redis6380.conf
sed -i "s#$port 6379#$port 6381#g"  redis6381.conf
# 在5以下，旧的版本中为 slaveof 
echo "replicaof 127.0.0.1 6379" >> redis6380.conf
echo "replicaof 127.0.0.1 6379" >> redis6381.conf
# run 1 master and 2 slave
nohup redis-server redis.conf &
nohup redis-server redis6380.conf &
nohup redis-server redis6381.conf &
# config 2 sentinel
cp sentinel.conf sentinel6380.conf
cp sentinel.conf sentinel6381.conf
sed -i "s#$port 26379#$port 26380#g"  sentinel6380.conf
sed -i "s#$port 26379#$port 26381#g"  sentinel6381.conf
# run 3 sentinel
nohup redis-sentinel  sentinel.conf &
nohup redis-sentinel  sentinel6380.conf &
nohup redis-sentinel  sentinel6381.conf &
```

## 查看集群状态：  
- 主节点
```sh
redis-cli -h localhost -p 6379

127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=127.0.0.1,port=6381,state=online,offset=17714,lag=1
slave1:ip=127.0.0.1,port=6380,state=online,offset=17714,lag=1
```
- 从结点  
  如果是连接到从节点，那么角色就是slave，并且只读不可写  
```sh
localhost:6380> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6379
master_link_status:up
...
localhost:6380> get 'a'
"1"
localhost:6380> set 'a' 2
(error) READONLY You can't write against a read only replica.
```

# 故障切换

## 主节点挂掉
现在测试下哨兵的故障切换，连上6379，并执行shutdown  
```sh
127.0.0.1:6379> shutdown
not connected>
```
然后在从节点看下集群信息，发现主已经切换到6381了：  
```sh
localhost:6380> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6381
master_link_status:up
```
它其实是动态地修改了配置信息的。可以重新查看下redis节点的配置文件和哨兵的sentinel.conf配置文件中的，已经将主节点改为了6381  
```sh
# 节点配置文件
replicaof 127.0.0.1 6381

# 哨兵配置文件
sentinel monitor mymaster 127.0.0.1 6381 2
```

此时故障切换测试完成。 也可以再连上6381验证读写和info信息。  

此时，如果主再挂掉，剩下的一个将处于只读不可写状态，因为配置文件中配置了quorum 参数（故障转移需要投票同意的节点数）为2

## 自动发现

另外， 我想测试下哨兵模式 的服务自动发现。 此处按照上面的脚本，同样准备一段执行代码  
```sh
#!/bin/bash
cp redis.conf redis6382.conf
sed -i "s#$port 6379#$port 6382#g"  redis6382.conf
echo "replicaof 127.0.0.1 6381" >> redis6382.conf
nohup redis-server redis6382.conf &
cp sentinel.conf sentinel6382.conf
sed -i "s#$port 26379#$port 26382#g"  sentinel6382.conf
nohup redis-sentinel  sentinel6382.conf &
```
> 注意： 这里sentinel.conf自动对齐了主节点，
`sentinel monitor mymaster 127.0.0.1 6381 2`
但是redis.conf中的端口还是6379，所以脚本略有变更  

然后，在主节点6381下查看集群信息。可以看到6382确实是自动加入进来了。  
```sh
127.0.0.1:6381> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=127.0.0.1,port=6380,state=online,offset=610579,lag=1
slave1:ip=127.0.0.1,port=6382,state=online,offset=610579,lag=1
```

## 自动恢复
  
``` sh
# 重新启动6379结点，预期中它应该重新连上集群，并成为Redis从节点
nohup redis-server redis.conf &
```
然后在主节点查看集群信息  
```sh
127.0.0.1:6381> info replication
# Replication
role:master
connected_slaves:3
slave0:ip=127.0.0.1,port=6380,state=online,offset=722511,lag=1
slave1:ip=127.0.0.1,port=6382,state=online,offset=722511,lag=1
slave2:ip=127.0.0.1,port=6379,state=online,offset=722644,lag=1
```  
> 如果想更仔细观察，建议配置日志 。默认`logfile`为空，意思是输出丢弃

‍

