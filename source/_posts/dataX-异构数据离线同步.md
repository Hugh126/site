---
title: dataX-异构数据离线同步
date: 2023-12-19 14:30:00
tags:
---
如果要实现不同数据库之前的数据同步，应该怎么做？直接执行sql文件，如果语法不兼容呢？dataX是一个阿里开源的，被广泛使用的离线数据同步的工具，通过各类插件的reader/writer兼容了各类数据的整合。可实现自定义全量/增量的数据同步。
<!--more-->
用一个示意图来理解dataX干的事情，就是  
![](https://cloud.githubusercontent.com/assets/1067175/17879841/93b7fc1c-6927-11e6-8cda-7cf8420fc65f.png)

> （百度到的一般都是阿里的商业版，而且也都是界面操作的文档。如果是开源版使用的话， 还是自行 quickStart 吧）  
https://github.com/alibaba/DataX/blob/master/userGuid.md


## 1. quickStart
以下以Mysql同步到postgresql为例说明。

### 检查环境配置
``` 
JDK1.8+
Python
Maven3
```

### 编辑Job配置
```
{
  "job": {
    "content": [
      {
        "reader": {
          "name": "mysqlreader",
          "parameter": {
            "connection": [
              {
                "jdbcUrl": ["jdbc:mysql://xxxx/xxx"],
				"querySql": [
			"select ...;"
				]
              }
            ],
            "password": "xxxxx",
            "username": "xxxxx"
          }
        },
        "writer": {
          "name": "mysqlwriter",
          "parameter": {
		    "column": [
			 "id",
			 "coupon_id",
			 "coupon_seq",
			 "value",
			 "use_date",
			 "crt_date",
			 "update_date",
			 "platform",
			 "member_id",
			 "coupon_act_id",
			 "end_time",
			 "begin_time",
			 "del_flag"
			],
            "connection": [
              {
                "jdbcUrl": "jdbc:mysql://localhost:3306/erp",
                "table": ["erp_coupon_list"]
              }
            ],
            "password": "123456",
            "preSql": [],
            "session": [],
            "username": "root",
            "writeMode": "insert"
          }
        }
      }
    ],
    "setting": {
      "speed": {
        "channel": "5"
      }
    }
  }
}
```

### 运行
``` python
python bin/datax.py job/job.json
```
> 如果是windows下执行会中文乱码，可以在窗口中执行`chcp 65001`, 临时变更编码格式为UTF-8。切勿去改注册表。  

运行结果1：
![](images/datax1.png)
这个是之前的测试实验。700w 数据，需要 3h 左右同步完成。我以为是网络问题导致，实则不然。

## 2.batchInsert 

上述实验结果居然性能太差，再来仔细看看文档[mysqlWriter](https://github.com/alibaba/DataX/blob/master/mysqlwriter/doc/mysqlwriter.md)的介绍:  
> 出于性能考虑，采用了 PreparedStatement + Batch，并且设置了：rewriteBatchedStatements=true，将数据缓冲到线程上下文 Buffer 中，当 Buffer 累计到预定阈值时，才发起写入请求。  
- jdbcUrl:
```
作业运行时，DataX 会在你提供的 jdbcUrl 后面追加如下属性：yearIsDateType=false&zeroDateTimeBehavior=convertToNull&rewriteBatchedStatements=true
```
- batchSize:
```
一次性批量提交的记录数大小，该值可以极大减少DataX与Mysql的网络交互次数，并提升整体吞吐量。但是该值设置过大可能会造成DataX运行进程OOM情况。默认值：1024
```
以上，我用本地mysql做了下实验，batchSize设置为5120，channel设置为5。430w的数据，近5分钟完成，大约1.4w行/s。虽然跟官网的测试（4条channel，4000的batchSize）速度8w/s来说，差距不小，但速度已然是飞一般的提升了。  

> 需要注意的是：  
1. 分布式数据库的写入速度肯定会比单机慢，要有心理预期
2. dataX同步的数据不保证在同一个事务内完成。因此如果同步任务复杂， 需要考虑失败的情况。可以通过pre任务来清理脏数据，通过post任务来做一下校验。









