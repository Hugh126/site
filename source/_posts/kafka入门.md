---
title: kafka入门
date: 2023-09-14 14:37:52
tags: 中间件
---
Kafka是一个分布式的事件流平台，这意味的你可以发布/订阅事件流，也可以可以用Ta来存储事件，在事件发生或发生后触发一些回调事件。这里的事件，其实就是消息、数据，只是当前潮流都是数据驱动模型，数据即意味着事件。
<!--more-->

# 序


# quickStart


## 常见问题
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

# Java生产者


# Java消费者

# Kafka-Eagle监控

## 环境准备

1. JDK安装


2. mysql安装


## eagle安装

## eagle使用

