---
title: grpc
date: 2023-03-19 11:48:58
tags: grpc
categories: Spring+
#description: gRPC是Google开源的一种高性能远程调用框架，它能高效地连接服务，支持负载均衡、跟踪、健康检查和身份验证。
---

gRPC是Google开源的一种高性能远程调用框架，它能高效地连接服务，支持负载均衡、跟踪、健康检查和身份验证，也能用于各移动端和浏览器对后端服务端连接。  
它使用[Protocol Buffers](https://protobuf.dev/)作为二进制序列化，使用HTTP/2进行数据传输。
<!--more-->  

![](https://grpc.io/img/landing-2.svg)


## grpc-demo-run
通过官网的demo，理解grpc的Server和Client
https://grpc.io/docs/languages/java/quickstart/  
建议使用使用提供的gradlew命令，遇到问题请参考[demo-run问题](#遇到的问题)  

## code解析  

### 1. sayHello  
GreeterGrpc.GreeterImplBase中
``` java
    @Override
    public void sayHello(HelloRequest req, StreamObserver<HelloReply> responseObserver) {
      HelloReply reply = HelloReply.newBuilder().setMessage("Hello " + req.getName()).build();
      responseObserver.onNext(reply);
      responseObserver.onCompleted();
    }
```

### 2. 服务端  
``` java
int port = 50051;
    server = ServerBuilder.forPort(port)
        .addService(new GreeterImpl())
        .build()
        .start();
```

### 3. 客户端

1) 通过地址创建channel  
``` java
ManagedChannel channel = ManagedChannelBuilder.forTarget("localhost:50051")
        // 使用明文避免需要证书
        .usePlaintext().build();
```
2) 通过channel创建stub
``` java
HelloGrpc.HelloBlockingStub blockingStub = HelloGrpc.newBlockingStub(channel);
```
3) 创建request并设置参数  
``` java
HelloRequest request = HelloRequest.newBuilder().setName(name).build();
4) 通过stub发送request
    HelloReply response response = blockingStub.sayHello(request);
```
## 遇到的问题
1. 开始参考quick start跑demo的时候，maven提示找不到jar包，然后我果断切换到一个2021年的分支。但还是报找不到jar报, 在maven仓库中搜索了下确实找不到`grpc-stub 1.33.2-SNAPSHOT`,果断替换成`1.33.1`  
2. maven编译成功后，IDEA中始终会显示包没引用正确的报错标记。但其实这只是IDEA的显示问题，不影响run，前提是无法通过界面运行Client和Server的Main方法了。  
> 在IDEA显示问题上追究很浪费时间，别忘记要学习的内容   
> 这里需要一下mvn的知识:  
> `run server`
> ``` bash
> mvn  -Dexec.mainClass=io.grpc.examples.helloworld.> > > HelloWorldServer exec:java
> ```
> `run client`
> ``` bash
> mvn -Dexec.mainClass=io.grpc.examples.helloworld.> HelloWorldClient exec:java
> ```
> 3. 本来还很奇怪grpc官网连jdk7都可以支持，怎么mvn编译demo的>方式也不提供下。用过之后，只想说 [真香!] --> [安装使用gradle](#安装使用gradle)  
> 4. 注意生成的代码在`target`或`build`目录  
> 其他demo参考
https://www.cnblogs.com/zhongyuanzhao000/p/13783165.html  



