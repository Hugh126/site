---
title: kafka入门
date: 2023-09-14 14:37:52
tags: 
- kafka
categories: 中间件
---
一个大家都在用的分布式的事件流平台...
<!--more-->
![](https://kafka.apache.org/images/streams-and-tables-p1_p4.png)
# quickStart
> 不建议在windows上瞎折腾，没有云主机就装个虚拟机，装个Ubantu玩一下吧。zookeeper在windows下总是莫名崩溃...  
> 建议结合官网[quickstart](https://kafka.apache.org/documentation/#quickstart)阅读


## 下载安装包
直接[官网下载](https://kafka.apache.org/downloads.html)版本2的二进制安装包。版本3中自己集成raft的模式在生产中应用不多。  
1. 下载后解压到合适目录，设置kafka环境变量
``` sh
sudo vim /etc/profile.d/my env.sh
# 增加如下内容:
# KAFKA HOME
export KAFKA HOME=/opt/module/kafkaexport PATH-SPATH:SKAFKA HOME/bin
# 生效
/etc/profilesource
```


## 修改kafkaServer配置文件
有几个重要配置需要check或修改
``` ini
# 如果存在多个kafkaServer，这个ID不能重复，且必须正整数
broker.id=0
# kafka 运行日志(数据) 存放的路径，路径不需要提前创建
log.dirs=/opt/module/kafka/datas
# 连接 Zookeeper 集群地址
zookeeper.connect=localhost:2181/kafka
# 如果是多个zookeeper,那么配置为 hadoop102:2181,hadoop104:2181/kafka
```

## 启动
从命令行来体会下基本的运用：创建主题、订阅、消费  
``` bash
# 启动zookeeper
bin/zookeeper-server-start.sh -daemon config/zookeeper.properties 

# 启动kafka
bin/kafka-server-start.sh  -daemon config/server.properties

# 查看主题
bin/kafka-topics.sh --bootstrap-server localhost:9092 --list

# 创建主题
bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --partitions 2 --replication-factor 1 --topic first

# 消费消息
## --from-beginning 
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092  --topic test

# 发送消息
bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic test
```


## 启动常见问题
- Cluster ID xxx doesn’t match stored clusterId in meta.properties
1. Step1:  日志中找到异常ID:  
`p2Ke6DSDzfdcxcfarkcxJscoQ`  
2. Step2:  
`cat  $KAFKA_HOME/config/server.properties  | grep log.dir`  
3. Step3: 编辑`meta.properties`并重启 
    ``` ini
    #Wed May 26 11:21:15 EET 2021
    cluster.id=P2Ka7bKGmJwBduCchqrhsP
    version=0
    broker.id=0
    ```

# 生产者与消费者

加入依赖  
``` xml
<dependency>
<groupId>org.apache.kafka</groupId>
<artifactId>kafka-clients</artifactId>
<version>3.0.0</version>
</dependency>
```

## java生产者
``` java
// 1. 创建kafka生产者的配置对象
Properties properties = new Properties();

// 2. 给kafka配置对象添加配置信息：bootstrap.servers
properties.put(ProducerConfigBOOTSTRAP_SERVERS_CONFIG, "172.16.90.164:9092");

// 3. 创建kafka生产者对象
KafkaProducer<String, String> kafkaProducer = new KafkaProducer<String, String>(properties);

// 发送完成回调函数.
// 注意：消息发送失败会自动重试，不需要我们在回调函数中手动重试
Callback callM = (metadata, exception) -> {
    if (exception == null) {
        // 没有异常,输出信息到控制台
        System.out.println("主题：" + metadata.topic()  + " ->"  + "分区：" + metadata.partition() );
    } else {
        // 出现异常打印
        System.out.println("成产消息异常" + exception.getMessage());
        exception.printStackTrace();
    }
};

// 4. 调用send方法,发送消息
for (int i = 0; i < 100; i++) {
    /**
        * 如果需要同步调用，直接future.get()即可
        * ProducerRecord 的构造函数中可以指定分区 详情见
        * @see org.apache.kafka.clients.producer.internals.DefaultPartitioner
        */
    String msg = "[序号]"+i + "[时间]" + LocalDateTime.now().toString();
    Future<RecordMetadata> future = kafkaProducer.send(new ProducerRecord<>("third", i%3, String.valueOf(i), msg), callM);
    TimeUnit.MILLISECONDS.sleep(100L);
}

// 5. 关闭资源
kafkaProducer.close();


```

## java消费者
``` java
//  Kafka从Broker中主动拉取数据
//  同一消费者组的消费者会瓜分消息

// 1.创建消费者的配置对象
Properties properties = new Properties();

// 2.给消费者配置对象添加参数
properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "172.16.90.164:9092");

// 配置序列化 必须
properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());

// 配置消费者组（组名任意起名） 必须
properties.put(ConsumerConfig.GROUP_ID_CONFIG, "consumer_group_001");

// 创建消费者对象
KafkaConsumer<String, String> kafkaConsumer = new KafkaConsumer<String, String>(properties);

// 注册要消费的主题（可以消费多个主题）
kafkaConsumer.subscribe(Arrays.asList("third"));

// 是否自动提交offset
properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, true);
// 提交offset的时间周期1000ms，默认5s
properties.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, 1000);

// 拉取数据打印
while (true) {
    // 设置1s中消费一批数据
    ConsumerRecords<String, String> consumerRecords = kafkaConsumer.poll(Duration.ofSeconds(1));

    if (consumerRecords.count() == 0) {
        continue;
    }
    // 打印消费到的数据
    for (ConsumerRecord<String, String> consumerRecord : consumerRecords) {
        System.out.println(consumerRecord);
        TimeUnit.MILLISECONDS.sleep(100L);
    }

}

```


# Kafka-Eagle监控
Eagle是一个带大屏的web监控，并支持新建主题、查看消息等强大功能

## 安装
> Ubantu默认安装了Jdk，却不设置home  
``` bash
# 先找到jdk安装地址
readlink -f `which java`

# 配置JAVA_HOME
sudo vim /etc/profile.d/env.sh
    export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
    export PATH=$PATH:$JAVA_HOME/bin
source /etc/profile
```

> 需要安装mysql  
``` bash
sudo apt install mysql-server
```

> 创建数据库和user  
``` bash
create database `ke` default character set utf8mb4 collate utf8mb4_general_ci;
CREATE USER 'ke'@'localhost' IDENTIFIED BY 'ke@123456';
GRANT ALL PRIVILEGES ON ke.* to 'ke'@'localhost';
flush privileges;
```


> 修改配置文件
``` sh 
vim conf/system-config.properties
# 特别注意
efak.zk.cluster.alias=cluster1
cluster1.zk.list=localhost:2181/kafka
# 重启
bin/ke.sh  restart
```

