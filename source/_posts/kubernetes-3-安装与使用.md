---
title: kubernetes-3-安装与使用
date: 2024-01-30 14:53:25
tags: kubernetes
---
k8s是未来，理解了基础架构和概念后，从实践入手
<!--more-->

## 一、基本应用

1. 仅使用容器能力

    > // 类似docker启动服务
    >
    > kubectl run redis --image\=redis
    >
    > // 查看pod
    >
    > kubectl get pods
    >
    > // 进入容器可直连
    >
    > kubectl exec -it redis -- bash
    >

2. 以上应用类似可以指定为k8s独有的Deployment（无状态应用，支持扩容缩容/滚动升级，自动重启等），如：

>  kubectl create deployment redis-deployment --image=redis

3. 具体应用往往不是简单的镜像复制，而包含了很多参数。kubectl支持通过yml配置文件来创建服务

    ```yml
    # pod.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: redis
    spec:
      containers:
      - name: redis
        image: redis
    ```

 指定配置文件启动： `kubectl create -f pod.yaml`​

‍

Pod不具有自愈能力，一般是通过Controller来控制。比如自定义一个nginx：

```yml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

‍

‍

## 二 、kubectl命令汇总

> 获取节点和服务版本信息  
> kubectl get nodes
>
> 获取pod的信息，以JSON格式展示
>
> kubectl get pod -o json
>
> 查看所有deployments信息  
> kubectl get deploy -A  
> 查看所有replicasets信息  
> kubectl get rs -A  
> 查看所有statefulsets信息  
> kubectl get sts -A  
> 查看所有jobs信息  
> kubectl get jobs -A  
> 查看所有ingresses信息  
> kubectl get ing -A  
> 查看有哪些名称空间  
> kubectl get ns

‍常用命令归类：

> kubectl get – 输出一个/多个资源
> kubectl create – 通过文件名或控制台输入，创建资源
> kubectl delete – 通过文件名、控制台输入、资源名或者label selector删除资源
> kubectl describe – 输出指定的一个/多个资源的详细信息
> kubectl edit – 编辑服务端的资源
> kubectl apply – 通过文件名或控制台输入，对资源进行配置
> kubectl exec – 在容器内部执行命令
> kubectl expose – 输入replication controller，service或者pod，并将其暴露为新的kubernetes service
> kubectl logs – 输出pod中一个容器的日志  


kuberctl命令行众多，不必死记硬背，在需要的时候查一查[kuberctl命令行概览](https://lib.jimmysong.io/kubernetes-handbook/cli/using-kubectl/)就知道了。  

另外，开始不熟练可以通过dashboard访问。

```sh
# 运行dashboard
minikube dashboard
# 让dashboard可以外部访问
kubectl proxy --port=8888 --address='0.0.0.0' --accept-hosts='^.*'
# 获取dashboard的url
minikube dashboard --url
```

‍

‍

## 三、创建无状态服务

一个 [Deployment](https://kubernetes.io/zh-cn/docs/tasks/run-application/run-stateless-application-deployment/) 为 [Pod](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/) 和 [ReplicaSet](https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/replicaset/) (副本控制器)提供声明式的更新能力  

‍

### 3.1  平滑升级

创建有两个备份的nginx，版本号14:

```yml
kubectl apply -f https://k8s.io/examples/application/deployment.yaml
# 查看deployment详情
kubectl describe deployment nginx-deployment
# 通过标签查询pod信息
kubectl get pods -l app=nginx
```

现在用一个新版本号的yml文件来， 升级nginx版本：

```yml
kubectl apply -f https://k8s.io/examples/application/deployment-update.yaml
# 也可以直接通过参数指定
kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1

# 不断查看pod观察
kubectl get pods -l app=nginx
# 重新查看deployment详情，校验版本号
kubectl describe deployment nginx-deployment
```

观察输出的Events信息，会先启动新的服务，然后停掉旧的；这个升级过程是平滑的

> StrategyType:       RollingUpdate  
> MinReadySeconds:    0  
> RollingUpdateStrategy:  1 max unavailable, 1 max surge

‍

### 3.2 动态扩容缩容

* 扩容

  ```yml
  kubectl apply -f https://k8s.io/examples/application/deployment-scale.yaml

  # 也可以使用scale缩放
  kubectl scale deployment/nginx-deployment --replicas=4
  ```

* 缩容

  ```yml
  # 指定备份数
  kubectl scale deployment/nginx-deployment --replicas=2
  # 按比例缩放
  kubectl autoscale deployment/nginx-deployment --min=2 --max=10 --cpu-percent=80
  ```

### 3.3 回滚

```yml
# 直接回滚
kubectl rollout undo deployment/nginx-deployment

# 获取历史
kubectl rollout history deployment/nginx-deployment
# 指定版本详情
kubectl rollout history deployment/nginx-deployment --revision=2
# 回滚到指定版本
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

## 四、创建有状态服务集群

StatefulSet 是用来管理有状态应用的工作负载 API 对象。

参考： https://juejin.cn/post/7033600843131666469  


创建一个主从结构的Redis集群：  

```yml
# server.yaml
apiVersion: apps/v1
kind: StatefulSet  # 类型为 statefulset
metadata:
  name: redis-sfs  # app 名称
spec:
  serviceName: redis-sfs  # 这里的 service 下面解释
  replicas: 2      # 定义了两个副本
  selector:
    matchLabels:
      app: redis-sfs
  template:
    metadata:
      labels:
        app: redis-sfs
    spec:
      containers:
      - name: redis-sfs 
        image: redis  # 镜像版本
        command:
          - bash
          - "-c"
          - |
            set -ex
            ordinal=`hostname | awk -F '-' '{print $NF}'`   # 使用 hostname 获取序列
            if [[ $ordinal -eq 0 ]]; then     # 如果是 0，作为主
              echo > /tmp/redis.conf
            else
              echo "slaveof redis-sfs-0.redis-sfs 6379" > /tmp/redis.conf # 如果是 1，作为备
            fi
            redis-server /tmp/redis.conf
```

执行`kubectl apply -f server.yaml` , 这里会创建redis-sfs-0 和 redis-sfs-1 两个 pod，他们正式按照 name-index 的规则来编号的。

上面的两个节点创建了，但是却是分开的两个主节点，而不是一主一从。需要建立Service，将它们放到一组：

```yml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-sfs
  labels:
    app: redis-sfs
spec:
  clusterIP: None   # 这里的 None 就是 Headless 的意思，表示会主动由 k8s 分配
  ports:
    - port: 6379
      name: redis-sfs
  selector:
    app: redis-sfs

```

执行`kubectl apply -f service.yaml` ，再查看日志 `kubectl logs -f redis-sfs-1`， 会获取主从连接信息。

‍

‍

‍

‍
