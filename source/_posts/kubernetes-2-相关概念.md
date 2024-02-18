---
title: kubernetes-2-相关概念
date: 2024-01-30 14:53:00
tags: kubernetes
categories: 云原生
---
k8s是未来，理解各类基本概念再开始
<!--more-->  

![image](/image/assets/image-20240116183723-ievwwyy.png)

## 元数据类型

* HPA：扩容缩容
* PodTemplate
* LimitRange


## 集群类型

NameSpace

Node

ClusterRole


## 命名空间


### 工作负载

* Pod： 容器组（共享网络、存储和依赖，它们被分配到同一Node，被同时调用；因此，应该是同一类型的应用运行在同一Pod中，一般只有一个容器）

* Controller： 可以创建和管理多个 Pod，提供副本管理、滚动升级和集群级别的自愈能力。例如，如果一个 Node 故障，Controller 就能自动将该节点上的 Pod 调度到其他健康的 Node 上

  * 无状态

    * Replication Controllers（RC， 动态控制副本数量）
    * ReplicaSet（RS，通过selector选择对哪些pod生效）
    * Deployment（对RS更高级的封装）

      1. 创建RS
      2. 滚动升级/回滚
      3. 平滑扩容/缩容
      4. 暂停与恢复
  * 有状态（针对持久化、网络标识、有序部署）

    * Headless Service（DNS域管理）
    * volumeClaimTemplates（持久化）
  * DaemonSet（日志、监控）
  * Job/CronJob


### 服务发现

- 	横向流量：Service
- 	纵向流量：Ingress

![image](/images/assets/image-20240116183723-ievwwyy.png)

- 集群内访问与集群外访问
![image](/imagesassets/image-20240116183756-jtgrcuo.png)


### 存储

* [Volume](https://lib.jimmysong.io/kubernetes-handbook/storage/volume/)
* CSI（容器持久化存储标准）


### 配置

[ConfigMap](https://lib.jimmysong.io/kubernetes-handbook/storage/configmap/)

[Secret](https://lib.jimmysong.io/kubernetes-handbook/storage/secret/)

DownwardAPI（将Pod信息暴露到容器内）


### 其他

* Role
* RoleBinding

---

