---
title: TiDB解决事务冲突
date: 2023-10-18 15:43:26
tags: Tidb
categories: 数据库
---
TiDB的事务提交是使用二阶段提交实现的。在客户端commit发起后，TiDB会从所有需要写入的Key中选取一个作为Primary Key，然后向所有涉及的TiKV发起prewrite。当发现有其他事务写当前Key，则发生写冲突。  
<!--more-->

# 一 故障描述
``` log
Write conflict, txnStartTS=445014188662194445 ,conflictStartTS= 445014189016088693 ,conflictCommitTS= 445014189016088709, key={tablelD=5500,indexID=9,indexValues={1851371555435577344,151,}} primary={tablelD=1475,handle=6780523)[try again later]
```
# 二 故障分析

## 日志释义
``` log
Write conflict：表示出现了写写冲突

txnStartTS=445014188662194445 ：表示当前事务的 start_ts 时间戳，可以通过 pd-ctl 工具将时间戳转换为具体时间

conflictStartTS=445014189016088693 ：表示冲突事务的 start_ts 时间戳，可以通过 pd-ctl 工具将时间戳转换为具体时间

conflictCommitTS=445014189016088709 ：表示冲突事务的 commit_ts 时间戳，可以通过 pd-ctl 工具将时间戳转换为具体时间

key={tablelD=5500,indexID=9,indexValues={1851371555435577344,151,}}：表示当前事务中冲突的数据，tableID 表示发生冲突的表的 ID，indexID 表示是索引数据发生了冲突。如果是数据发生了冲突，会打印 handle=x 表示对应哪行数据发生了冲突，indexValues 表示发生冲突的索引数据

primary={tablelD=1475,handle=6780523)：表示当前事务中的 Primary Key 信息
```

# 三 故障解决

## 获取冲突的信息
> 由于没有`pd-ctl`工具，暂时略过时间戳信息  

``` sql
select * from information_schema.tables where tidb_table_id in (5500,1475);

# TABLE_NAME
# erp_employee
# simple_workflow_record

```
通过以上sql结果可以获知是`erp_employee`和`simple_workflow_record`表冲突了  
``` sql
select * from information_schema.tidb_indexes WHERE table_name = (select TABLE_NAME from information_schema.tables where tidb_table_id =5500) and index_id = 9;

```    

|KEY_NAME|COLUMN_NAME|
|--|--|
|erp_employee_update_date_index|update_date|
通过以上sql结果可以获知是字段`update_date`冲突了。

``` log
primary={tableID=47, indexID=1, indexValues={string, }}
```



