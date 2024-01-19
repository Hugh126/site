---
title: redis集群-sentinel
date: 2024-01-19 21:22:58
tags:
---
redis哨兵模式实现了redis基本的高可用
<!--more-->
# redis-sentinel

‍

理解sentinel集群：https://redis.io/docs/management/sentinel/

sentinel的配置文件为 sentinel.conf

关键配置

> sentinel monitor mymaster 127.0.0.1 6379 2  
> sentinel down-after-milliseconds mymaster 60000  
> sentinel failover-timeout mymaster 180000  
> sentinel parallel-syncs mymaster 1

启动哨兵

> redis-sentinel /path/to/sentinel.conf  
> 或  
> redis-server /path/to/sentinel.conf --sentinel

‍

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

‍

启动Redis哨兵集群脚本：

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

查看集群状态，

```sh
redis-cli -h localhost -p 6379

127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=127.0.0.1,port=6381,state=online,offset=17714,lag=1
slave1:ip=127.0.0.1,port=6380,state=online,offset=17714,lag=1
master_failover_state:no-failover
master_replid:2f8a76c364d82eb9784b09db128f6acdf9059e46
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:17994
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:17994

```

如果是连接到从节点，那么角色就是slave，并且只读不可写

```sh
localhost:6380> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6379
master_link_status:up
master_last_io_seconds_ago:0
master_sync_in_progress:0
slave_read_repl_offset:344530
slave_repl_offset:344530
slave_priority:100
slave_read_only:1
replica_announced:1
connected_slaves:0
master_failover_state:no-failover
master_replid:2f8a76c364d82eb9784b09db128f6acdf9059e46
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:344530
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:822
repl_backlog_histlen:343709
localhost:6380> get 'a'
"1"
localhost:6380> set 'a' 2
(error) READONLY You can't write against a read only replica.

```

现在测试下哨兵的故障切换，连上6379，并执行shutdown

```sh
127.0.0.1:6379> shutdown
not connected>
```

然后在从节点看下集群信息，发现主已经切换到6381了，

```sh
localhost:6380> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6381
master_link_status:up
master_last_io_seconds_ago:0
master_sync_in_progress:0
slave_read_repl_offset:410079
slave_repl_offset:410079
slave_priority:100
slave_read_only:1
replica_announced:1
connected_slaves:0
master_failover_state:no-failover
master_replid:d908feaf1cdb34a3247f4c25afccc714e691f126
master_replid2:2f8a76c364d82eb9784b09db128f6acdf9059e46
master_repl_offset:410079
second_repl_offset:390779
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:822
repl_backlog_histlen:409258
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

---

另外， 我想测试下哨兵模式 的服务自动发现。 此处按照上面的脚本，同样准备一段执行代码

>  注意： 这里sentinel.conf自动对齐了主节点，
>
> sentinel monitor mymaster 127.0.0.1 6381 2
>
> 但是redis.conf中的端口还是6379，所以脚本略有变更

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

然后，在主节点6381下查看集群信息。可以看到6382确实是自动加入进来了。

```sh
127.0.0.1:6381> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=127.0.0.1,port=6380,state=online,offset=610579,lag=1
slave1:ip=127.0.0.1,port=6382,state=online,offset=610579,lag=1

```

---

另外，我还想尝试下挂掉的节点自动恢复功能。启动6379   nohup redis-server redis.conf &

然后在主节点查看

```sh
127.0.0.1:6381> info replication
# Replication
role:master
connected_slaves:3
slave0:ip=127.0.0.1,port=6380,state=online,offset=722511,lag=1
slave1:ip=127.0.0.1,port=6382,state=online,offset=722511,lag=1
slave2:ip=127.0.0.1,port=6379,state=online,offset=722644,lag=1

```

---

‍

> 如果想更仔细观察，建议配置日志 。在配置文件中，配置 logfile ，默认为空，输出丢弃

‍

