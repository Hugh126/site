---
title: arthas简述
date: 2023-07-07 17:28:08
tags:
- arthas
categories: 问题经验
---
arthas作为一款监控诊断的工具，能实时查看JVM信息，可以对业务问题进行诊断，包括查看方法调用的出入参、异常，监测方法执行耗时等。实乃线上跟踪定位工具的不二之选!
<!--more-->
## 1. 查看类及子类信息（包含私有变量）
``` bash
sc -d -f com.mxzhang.erp.config.ErpConfig
```
![](/images/arthas1.png)

## 2. 调用方法，使用tomcat时使用OGNL需要制定类加载器
``` bash
ognl --classLoaderClass org.apache.catalina.loader.ParallelWebappClassLoader '@com.mxzhang.erp.config.ErpConfig@isRunningBatchTaskEnv()'
```

### 简单参数
``` bash
ognl --classLoaderClass org.apache.catalina.loader.ParallelWebappClassLoader '@com.mxzhang.erp.config.ErpConfig@getProperty("isHK")' -x 1

ognl --classLoaderClass org.apache.catalina.loader.ParallelWebappClassLoader '@com.mxzhang.erp.config.ErpConfig@isApiTestDesk(4)' -x 2
```
![](/images/arthas2.png)

## 3.调用构造方法执行非静态方法
New这个对象，再执行方法即可  

## 4. 调用任意Bean
基本步骤：  
    1.找到classLoaderHash  
    2. ognl通过类加载器调用方法  

``` bash
[arthas@22714]$ sc -d ApplicationContextUtil
Affect(row-cnt:0) cost in 16 ms.
[arthas@22714]$ sc -d *ApplicationContextUtil
 class-info        org.jeecgframework.core.util.ApplicationContextUtil
 code-source       /ayplot/erptomcat8.5/webapps/erp/WEB-INF/classes/
 name              org.jeecgframework.core.util.ApplicationContextUtil
 isInterface       false
 isAnnotation      false
 isEnum            false
 isAnonymousClass  false
 isArray           false
 isLocalClass      false
 isMemberClass     false
 isPrimitive       false
 isSynthetic       false
 simple-name       ApplicationContextUtil
 modifier          public
 annotation
 interfaces        org.springframework.context.ApplicationContextAware
 super-class       +-java.lang.Object
 class-loader      +-ParallelWebappClassLoader
                       context: erp
                       delegate: false
                     ----------> Parent Classloader:
                     java.net.URLClassLoader@3ac3fd8b

                     +-java.net.URLClassLoader@3ac3fd8b
                       +-sun.misc.Launcher$AppClassLoader@18b4aac2
                         +-sun.misc.Launcher$ExtClassLoader@7b7c0aa8
 classLoaderHash   3b165738

Affect(row-cnt:1) cost in 86 ms.
[arthas@22714]$ ognl -c 3b165738 '@org.jeecgframework.core.util.ApplicationContextUtil@getContext().getBean("health").access()'
@String[OK]
```

## 5. 记录每次方法的调用
``` bash
//tt -t com.mxzhang.erp.api.actuator.Health  access

[arthas@22714]$ tt -t  com.mxzhang.erp.employee.controller.ErpEmployeeController datagrid
Press Q or Ctrl+C to abort.
Affect(class count: 1 , method count: 1) cost in 503 ms, listenerId: 6
 INDEX       TIMESTAMP                      COST(ms)        IS-RET      IS-EXP       OBJECT                 CLASS                                         METHOD
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 1003        2020-12-04 12:44:39            1349.904497     true        false        0x64a3f100             ErpEmployeeController                         datagrid
 1004        2020-12-04 12:44:46            1526.542767     true        false        0xef690d8              ErpEmployeeController                         datagrid
 1005        2020-12-04 12:45:09            1268.608223     true        false        0x294385ad             ErpEmployeeController                         datagrid

```

### 查看入参出参详情
``` bash
[arthas@22714]$ tt -i 1003
 INDEX          1003
 GMT-CREATE     2020-12-04 12:44:39
 COST(ms)       1349.904497
 OBJECT         0x64a3f100
 CLASS          com.mxzhang.erp.employee.controller.ErpEmployeeController
 METHOD         datagrid
 IS-RETURN      true
 IS-EXCEPTION   false
 PARAMETERS[0]  @ErpEmployeeEntity[
```
> 此处参数被封装成引用对象，无法查看

### 重新发起请求
``` bash
[arthas@22714]$ tt -t com.mxzhang.erp.api.actuator.Health  access
Press Q or Ctrl+C to abort.
Affect(class count: 1 , method count: 1) cost in 320 ms, listenerId: 7
 INDEX       TIMESTAMP                      COST(ms)        IS-RET      IS-EXP       OBJECT                 CLASS                                         METHOD
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 1006        2020-12-04 13:50:49            0.09137         true        false        0x7417c9e4             Health                                        access
[arthas@22714]$ tt -i 1006 -p
 RE-INDEX      1006
 GMT-REPLAY    2020-12-04 13:51:14
 OBJECT        0x7417c9e4
 CLASS         com.mxzhang.erp.api.actuator.Health
 METHOD        access
 IS-RETURN     true
 IS-EXCEPTION  false
 COST(ms)      0.143933
 RETURN-OBJ    @String[OK]
Time fragment[1006] successfully replayed 1 times.

```
> ThreadLocal 信息丢失和引用对象变更会导致无法准确获取，此时需要用到watch  

## 观察每次方法调用
``` bash
watch
watch com.mxzhang.erp.employee.controller.ErpEmployeeController datagrid '{params,target,returnObj}' -x 3

// 观察具体参数
watch com.mxzhang.erp.employee.controller.ErpEmployeeController datagrid  'target'
```

## 7. 追踪方法调用栈
``` bash
[arthas@22714]$ trace com.mxzhang.erp.employee.controller.ErpEmployeeController datagrid '#cost > 10'
Press Q or Ctrl+C to abort.
Affect(class count: 1 , method count: 1) cost in 542 ms, listenerId: 15
`---ts=2020-12-04 15:00:42;thread_name=http-nio-8181-exec-21;id=c0;is_daemon=true;priority=5;TCCL=org.apache.catalina.loader.ParallelWebappClassLoader@3b165738
    `---[1208.379063ms] com.mxzhang.erp.employee.controller.ErpEmployeeController:datagrid()
        +---[5.28227ms] com.mxzhang.erp.employee.controller.ErpEmployeeController:createDatagridCriteriaQuery() #675
        +---[0.021024ms] javax.servlet.http.HttpServletRequest:getParameter() #676
        +---[0.016183ms] com.mxzhang.erp.utils.StringUtils:isNotEmpty() #677
        +---[0.016176ms] org.jeecgframework.core.common.hibernate.qbc.CriteriaQuery:add() #680
        +---[0.025832ms] org.hibernate.criterion.Restrictions:disjunction() #681
        +---[0.015999ms] javax.servlet.http.HttpServletRequest:getParameter() #683
        +---[0.016274ms] javax.servlet.http.HttpServletRequest:getParameter() #684
        +---[0.015918ms] com.mxzhang.erp.utils.StringUtils:isNotEmpty() #685
        +---[0.019184ms] com.mxzhang.erp.utils.StringUtils:isNotEmpty() #690
        +---[0.031676ms] org.jeecgframework.core.common.hibernate.qbc.CriteriaQuery:add() #693
        +---[44.915965ms] com.mxzhang.erp.employee.service.ErpEmployeeServiceI:getDataGridReturn() #694
        +---[0.031499ms] org.jeecgframework.core.common.model.json.DataGrid:getResults() #695
        +---[34.233436ms] com.mxzhang.erp.employee.controller.ErpEmployeeController:postProcessEmployeeList() #696
        `---[1122.992269ms] org.jeecgframework.tag.core.easyui.TagUtil:datagrid() #697
```

**这个功能在调优的时候非常实用，可以看到每行代码的耗时。这种功能一般只有GW的付费软件才有**  

### 次数限制
``` bash
trace demo.MathGame run -n 1
```
这种功能对并发高的函数很有用

### 调用耗时过滤
``` bash
$ trace demo.MathGame run '#cost > 10'
```

### trace多层
trace只能1层，因为多层扩展的代价很大。但是trace提供了多个类，可以使用此功能达到类似效果
``` bash
trace -E com.test.ClassA|org.test.ClassB method1|method2|method3
```

### 排除指定类
``` bash
trace javax.servlet.Filter * --exclude-class-pattern com.demo.TestFilter
```


## 8. 代码热更新
``` bash
[arthas@22714]$ redefine /home/deployer/Health.class
redefine success, size: 1, classes:
com.mxzhang.erp.api.actuator.Health
```
也可以结合jad和mc命令在服务器上完成操作  
1. jad反编译
``` bash
jad --source-only com.mxzhang.erp.api.actuator.Health > /app/Health.java
```
2. 修改代码
3. 生成class
``` bash
[arthas@22714]$ sc -d *Health | grep classLoaderHash
 classLoaderHash   3b165738
[arthas@22714]$ mc -c 3b165738 /app/Health.java
Memory compiler error, exception message: java.lang.RuntimeException: Wasn't able to open jar:file:/ayplot/erptomcat8.5/webapps/erp/WEB-INF/lib/commons-lang-2.6.jar!/ as a jar file, please check $HOME/logs/arthas/arthas.log for more details.
// 这里我一直是失败的，不知为何，官网也提示mc经常失败
```
4. 热加载
``` bash
redefine /home/deployer/Health.class
```

> **redefine的限制**
- 不允许新增加field/method
- 正在跑的函数，没有退出不能生效

**jrebel还是比arthas提供的热更新还是强大很多，但也要注意一点：以后热更新的时候要避免使用功能，防止更新失败**  

> 更复杂姿势参考[ognl表达式](https://commons.apache.org/proper/commons-ognl/language-guide.html)  

