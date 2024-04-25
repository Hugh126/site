---
title: 一次OOM分析
date: 2024-01-04 16:28:59
tags: OOM
categories: troubleshooting
---
世界上没有无缘无故的爱，也没有无缘无故的恨。  
OOM了，肯定是要找到元凶的！
<!--more-->

# 一、问题描述
（tomcat的OOM信息会在catalina.log日志文件和catalina.out日志文件打印）
​![image](/images/assets/image-20240104152220-2izpt7h.png)​


OOM的时候记录并dump堆文件，JVM启动参数需要加上：
```java
  # 指定生成Dump文件的异常类型
  -XX:+HeapDumpOnOutOfMemoryError

  # 指定Dump文件生成的位置
  -XX:HeapDumpPath=/data/log/erp.hprof
```

> 如果是运行中需要分析堆，也可以使用jmap主动生成
>
> jmap -dump:format=b,file=heap_dump.hprof  $PID

# 二、问题思路
生成堆文件后，需要分析工具。可以采用开源的[Mat](https://eclipse.dev/mat/previousReleases.php)

> 注意：
>
> 1. 最新版本的1.15要求最少是JDK17, 我用的是1.10 可以在JDK8上运行
> 2. Mat运行要求的内存空间较大，实际根据堆文件大小来调整 MemoryAnalyzer.ini 配置文件 。
>
>     我堆文件有25G，实际我分配了20G的最大堆才导入成功

如果服务器上内存够用的话，可以直接生成相关报告：

> ./ParseHeapDump.sh /data/log/erp_202312.hprof   org.eclipse.mat.api:suspects  org.eclipse.mat.api:overview   org.eclipse.mat.api:top_components

‍
# 三、问题分析
好不容易导入堆文件后没找到重点，在参考一下别人怎么玩的之后，通两个功能（排列大对象+内存泄露分析）就找到元凶。  

​![image](/images/assets/image-20240104154329-g9hnw90.png)​


## 3.1 锁定上下文
首先，对象名称是一个定时任务线程池的线程。在圆饼上左键看看大对象的上下文信息，可以找到这个定时任务具体执行类是哪个

​![image](/images/assets/image-20240104155246-fieh1f7.png)​

​![image](/images/assets/image-20240104160611-xmslynf.png)​

‍
## 3.2 大对象分析
然后，选择大对象分析，发现两个List占用了11.5G，我最大堆分配的是15G。这里估计就是原因所在了

​![image](/images/assets/image-20240104155650-95jxrin.png)​

‍
## 3.3 内存泄露分析
使用内存泄露分析，发现大对象List是哪里出现的

​![image](/images/assets/image-20240104160141-n1db8go.png)​

‍
## 3.4 结合代码验证
反过来再参考代码和日志 ，印证了服务挂掉的就是这个位置  
（这里看起来只取了一堆ID，实际上每个值都是一个LinkedHashMap，数据中大概每个List都是2000w）：
​![image](/images/assets/image-20240104155916-dw9zsud.png)​


# 四、问题解决
原代码的JdbcService其实还是使用Jdbctemplate的RowMapper进行行数据提取，生成了大量的中间Map。  
可以使用`query(String sql, RowCallbackHandler rch)`, 也可以自定义ResultSetExtractor，主动提取id，省掉中间Map

参考：
> [Dump文件分析工具 - MAT图文解析_dump mat-CSDN博客](https://blog.csdn.net/F1004145107/article/details/106365672)

‍
