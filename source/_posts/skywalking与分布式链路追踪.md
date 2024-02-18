---
title: skywalking与分布式链路追踪
date: 2023-09-26 11:03:39
tags: skywalking
categories: 中间件
---
Skywalking是一款由国人主导开发的分布式链路追踪系统，它支持多种语言的探针，对国产开源软件有全面的支持，使用ES作为底层存储，具有强大的检索能力，而且有非常活跃的中文社区。¹³⁴
<!--more-->

# 一 分布式链路追踪系统异同

其他的分布式链路追踪系统有很多，比如谷歌的Dapper，韩国的Pinpoint，Twitter的Zipkin，以及Jaeger等。²³ 它们各有各的优缺点，比如：

- Dapper是链路追踪领域的始祖，它提出了基于采样的跟踪方法，以及跨进程传递Trace ID和Span ID的机制。² 但是Dapper并没有开源，只是公开了它的设计思想和架构。²
- Pinpoint是一款功能强大的APM软件，它支持Java和PHP语言，使用HBase作为存储，具有海量存储能力，而且跟踪数据粒度非常细，用户界面也很友好。² 但是Pinpoint在社区交流上会有一定滞后，而且需要运维住一套HBase集群。²
- Zipkin是一款基于Dapper论文实现的开源链路追踪系统，它支持多种语言和存储方式，而且有很多社区贡献的插件和扩展。² 但是Zipkin对代码有一定的侵入性，而且用户界面比较简陋。²
- Jaeger是一款由Uber开源的链路追踪系统，它也是基于Dapper论文实现的，但是加入了一些新的特性，比如自适应采样和OpenTracing支持。² Jaeger使用Cassandra或ES作为存储，具有良好的可扩展性和查询能力，而且用户界面也比较美观。²

总之，skywalking与其他分布式链路追踪系统的异同主要体现在以下几个方面：

- 代码侵入性：skywalking和Pinpoint都是基于字节码注入技术实现的代码无侵入性，而Zipkin则需要在代码中添加注解或拦截器等。²
- 存储方式：skywalking使用ES作为存储，具有强大的检索能力，而Pinpoint使用HBase作为存储，具有海量存储能力。Zipkin和Jaeger则支持多种存储方式。²
- 用户界面：Pinpoint和Jaeger都有功能强大和美观的用户界面，而skywalking和Zipkin则相对简单。不过skywalking有一款第三方定制UI，做得比Pinpoint更漂亮。²
- 社区活跃度：skywalking在国内社区非常活跃，可以与项目发起人零距离沟通，而Pinpoint在社区交流上会有一定滞后。Zipkin和Jaeger则在国外社区比较活跃。²
- 支持语言：skywalking支持5种语言：Java, C#, PHP, Node.js, Go。Pinpoint只支持Java和PHP。Zipkin和Jaeger则支持多种语言。²³
- 跟踪粒度：Pinpoint在跟踪粒度方面做得非常好，可以显示每个方法的调用时间和参数等信息。而skywalking则只显示服务之间的调用关系和时间等信息。²

> 参考:  
(1) 全网最详细的Skywalking分布式链路追踪 - 掘金. https://juejin.cn/post/7072709231949905957.  
(2) SkyWalking最全详解(作用原理及使用流程) – mikechen. https://mikechen.cc/21974.html.  
(3) 几款符合 OpenTracing 规范的分布式链路追踪组件介绍与 .... https://cloud.tencent.com/developer/article/1781084.  
(4) 全链路追踪技术选型：pinpoint vs skywalking - 知乎. https://zhuanlan.zhihu.com/p/436018582.

# 二 docker安装
## 2.1 安装依赖
``` sh
sudo apt update
sudo apt install apt-transport-https ca-certificates curl gnupg-agent software-properties-common
```

## 2.2 导入仓库源
``` sh
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

## 2.3 安装最新版本
``` sh
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io
```

> 参考: https://zhuanlan.zhihu.com/p/143156163


# 三 skywalking安装
skywalking中数据存储有两种方式， ES和默认的H2，由于ES同时有强大的分析能力，因此各类都会建议你使用ES的安装方式。  
例如：https://juejin.cn/post/7058129741554909197 ， 然而，实际上我使用docker安装ES，安装后总是会莫名崩溃；使用docker查看log日志，也是看不个所以然，甚至跟踪到skywalking本身的bug。其实，对于入门者体验为先。下面是H2存储的安装方式：  


## 3.1 拉取镜像
``` sh
docker pull apache/skywalking-oap-server:9.2.0
docker pull apache/skywalking-ui:9.2.0
```

## 3.2 安装skywalking-oap
``` sh
docker run --name skywalking-oap -e TZ=Asia/Shanghai -p 12800:12800 -p 11800:11800 --restart always -d apache/skywalking-oap-server:9.2.0
```

## 3.3 安装skywalking-ui
```
docker run -d --name skywalking-ui \
 --restart=always \
 -e TZ=Asia/Shanghai \
 -p 8088:8080 \
 --link skywalking-oap:oap \
 -e SW_OAP_ADDRESS=http://oap:12800 \
 apache/skywalking-ui:9.2.0
```
> 由于skywalking-ui默认的8080端口很容易跟很多web服务冲突，此处docker映射到外面的8088端口  
访问 http://{安装服务器的ip}:8088 可以看到skywalking的监控界面就安装成功了!  

# 四 应用接入
spring项目的接入方式一般都是修改java应用的启动参数，附加agent。  
## 4.1 agent下载
找到两个cdn地址，都可以。src为源码，sha为校验文件，下载另外一种压缩包：
https://archive.apache.org/dist/skywalking/java-agent/9.0.0/  
https://dlcdn.apache.org/skywalking/java-agent/9.0.0/

## 4.2 应用接入
需要添加agent地址 / 应用名称 / skywalking-oap的连接地址  
参数示例：  
``` ini
-javaagent:D:\\soft\\skywalking-agent\\skywalking-agent.jar
-Dskywalking.agent.service_name=shorturl
-Dskywalking.collector.backend_service=172.16.90.164:11800
```
> 如果是放到tomcat中启动，修改tomcat下的`bin/catalida.sh`，将JAVA_OPS变量附加以上参数即可。效果如下：    
![](/images/skywalking1.png)
![](/images/skywalking2.png)
上面可以看到每次请求的耗时，点进去则是trace每次。可以看到，当前agent可识别http/rpc请求，sql执行，druid拦截器等。

# 五 skywalking自定义扩展

## 5.1 自定义方法Trace
在实际业务监控中，不仅是各类组合外部调用会被Trace到一次Api中，有时候一些复杂方法也会希望被集成为一个Span放进去。这时通过插件`apm-toolkit-trace`可以完成。  

1. 需要接入的应用中新增依赖  
``` xml
   <dependency>
      <groupId>org.apache.skywalking</groupId>
      <artifactId>apm-toolkit-trace</artifactId>
      // 版本尽量和skywalking版本一致
      <version>${skywalking.version}</version>
   </dependency>
```

2. 在方法上加注解  
``` java
    @Trace
    @Tags({@Tag(key = "originalURL", value = "arg[0]"),
            @Tag(key = "tenantId", value = "arg[2]")})
    public String generateUrlMap(String originalURL, String expireDate, String tenantId, String status, String note) {
        ...
    }
```
被`@Trace`修饰的方法会形成一个span放到外层接口Trace中。  
![](/images/skywalking3.png)  
- 可以看到新增修饰的方法已经被标记到接口请求的trace中

> 非代码侵入的trace方式参考： https://blog.csdn.net/zxh1991811/article/details/115379470  


## 5.2 输出Span日志到skywalking

- skywalking的自定义Trace可以帮助我们融入业务指标，定位到业务异常。但是，如果还需要反过来去日志服务中寻找相关异常，岂不是很浪费时间。这一节就是要解决这个痛点!  
https://blog.csdn.net/wb4927598/article/details/119192594








