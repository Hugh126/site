---
title: OLAP浅入
date: 2023-05-06 13:13:27
tags:
    - OLAP
    - clickhouse
categories: 大数据
---
随着业务的发展，数据时常是指数级增长，数据提取/BI报表/数据挖掘总是被提及。那么如何支撑？如何建设数仓，如何支持实时分析?

<!--more-->

# 序
传统行业随着业务深入，无可避免会涉及BI的需要。用传统的PGSQL有点跟不上时代，也跟不上BOSS门的'奇思妙想'了，大刀阔斧的追赶潮流又缺人缺资源。想选择最优实践方案，就必须'货比三家'，打开自己的脑壳。以下：  

[Doris的应用，360数仓演进进程](https://mp.weixin.qq.com/s/2oeEcqgLEbbPZHnp0kAElQ)


[OLAP介绍](https://zhuanlan.zhihu.com/p/448265353)

  
[开源OLAP引擎对比](https://segmentfault.com/a/1190000040428093)


## ClickHouse

- 官网安装体验  
安装(https://clickhouse.com/docs/zh/getting-started/
install)

- 导入数据  
支持外表引擎和第三方工具(https://cloud.tencent.com/developer/beta/article/1602662)


## Doris
[Doris简史](https://zhuanlan.zhihu.com/p/387050823)


## Doris与ClickHouse对比
[自己构建更复杂的用ClickHouse，省力用Doris](https://zhuanlan.zhihu.com/p/421469439)  
实则Doris脱胎于百度广告业务，国内社区资源更丰富,可以看下[其他公司的实践](https://doris.apache.org/zh-CN/blog/)  

![](https://cdn-tencent.selectdb.com/zh-CN/assets/images/page_3-zh-bb25c0ea2faa03912dea231b8b207d3e.png)
