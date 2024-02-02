---
title: kubernetes-1-基本架构
date: 2024-01-03 20:36:05
tags: kubernetes
---
k8s是未来
<!--more-->


​![image](/images/assets/image-20240116140626-xdrpftt.png)​


​​![image](/images/assets/image-20240117120120-oyzznx1.png)​​

‍
## 组件及概念

### Master

1. ApiServer： 接口服务
2. kube-Controller-manager ： 控制器管理器
3. cloud-Controller-manager ： 第三方控制器管理器
4. kube-scheduler： （节点）调度器

‍

### Node

1. kubelet：负责Pod生命周期、存储、网络
2. kube-proxy： 网络代理、服务发现负载
3. Pod：kubernetes的最小控制单元，容器都是运行在pod中的，一个pod中可以有1个或者多个容器
4. container-runtime： 容器运行时环境（docker、containerd...）

‍

### k8s生态架构

* 核心层：Kubernetes 最核心的功能，对外提供 API 构建高层的应用，对内提供插件式应用执行环境
* 应用层：部署（无状态应用、有状态应用、批处理任务、集群应用等）和路由（服务发现、DNS 解析等）、Service Mesh（部分位于应用层）
* 管理层：系统度量（如基础设施、容器和网络的度量），自动化（如自动扩展、动态 Provision 等）以及策略管理（RBAC、Quota、PSP、NetworkPolicy 等）、Service Mesh（部分位于管理层）
* 接口层：kubectl 命令行工具、客户端 SDK 以及集群联邦
* 生态系统：日志监控/集群配置等
> 作为开发者，最主要的就是理解服务、任务的执行部署方式，熟悉kubectl和API的使用，了解流量控制  





---
学习资料:

[2.2.2_集群架构与组件-组件详解：核心组件与分层架构_bilibili_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1MT411x7GH?p=10&spm_id_from=pageDriver&vd_source=eeb1c6e015a7ca03cb75e71d6ae0a6e1)  

[Kubernetes 架构 | 云原生资料库 (jimmysong.io)](https://lib.jimmysong.io/kubernetes-handbook/architecture/)  


