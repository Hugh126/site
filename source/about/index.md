---
title: 关于自己
date: 2023-03-22 11:22:22
---

# 个人信息
**洪宗敏**  意向: 效能研发
<table>
    <tr>
        <td>34岁</td> 
        <td>男</td> 
        <td>武汉</td> 
   </tr>
    <tr>  
  		 <td>15527282430</td> 
         <td colspan="2">283341330@qq.com</td> 
    </tr>
</table>



# 自我评价
- 有9年的web开发经验、4年运维经验，熟悉阿里云诸多产品
- 能运用python/shell等脚本语言，有H5/微信小程序/移动端项目经验
- 熟悉Spring源码，熟练掌握Java/Spring/Mybatis/Hibernate/MySQL/Redis
- 熟悉Springboot, 熟练运用Nacos/Kafka/SpringCache/grpc等组件
- 理解JVM，有参数调优和线上故障定位经验
- 熟悉MySQL/PostgreSQL/TIDB等多种数据库，熟悉SQL调优
> [个人博客 https://hugh126.github.io](https://hugh126.github.io)

# 工作经历
### 1. 海伦司科技有限责任公司 2018.11~现在  

1. 为了让研发便于稳定发版但又不接触生产服务器。从0到1研发公司自动化运维平台：  
主要集成发版、补丁、监控、SQL审批执行等功能
2. 公司没有运维，积累大量一线问题解决及运维经验
3. 有实际的公众号开发、小程序开发、IOSApp开发、智能IOT设备接入等项目经验  
> 总结: 近5年时间，帮助团队从3人扩展到30人，承担运维开发多重职责，优化系统效率保障线上稳定

### 2. 软通动力技术服务有限公司 2016.05~2018.09
1. 协同华为武汉SDN团队，完成沙箱项目开发及展示
2. 扩展团队从2人到9人，承接独立项目；熟悉了微服务开发的流程，积累大量CI/CD以及LLT经验

### 3. 广州世纪龙信息网络有限责任公司(中国电信)  2014.7~2016.04
使用python处理分段日志，完成流量监控分析平台开发与维护；其中熟悉了Linux与日志处理


# 项目经验
### 1. Ayplot自动运维平台
运维平台是参考开源项目Walle的思想，借助开源平台jeecgboot实现自己的企业运维平台。
- 技术栈：
Ant-design-vue + Vue2 + SpringBoot2 + MybatisPlus3 + Redis+ Druid+ Mysql
- 个人贡献：
0. 集成环境管理、服务器管理、项目管理和部署回滚功能
1. 支持mvn和ant打包成war或jar的打包方式
2. 支持了前后置任务服务切量上下线，通过自定义shell关联前端更新部署
3. 为方便审批发布，支持钉钉移动端审批
4. 支持各服务自定义各自启动JVM参数
5. 集成SQL的审批操作:发布前对比测试与正式数据库结构并同步,以及临时开发申请执行的SQL

### 2. 企业ERP
企业ERP是海伦司的基础服务配置平台，包含了人事、运营、采购、营销、财务几大模块。  
- 技术栈：    
JSP+ EasyUI + Jeecg(SpringMVC) + Hibernate+ Redis/Hazelcast + Druid+ Tidb
- 个人贡献：  
1. JVM参数调优，通过arthas解决线上OOM或CPU占用过高等问题
2. 优化审批流：事务不完整问题；数据库记录与钉钉通知不一致问题；使用策略模式解决分支问题；封装Guava的MapDiff优化审批数据对比
3. 优化quartz定时任务：使用Reids的发布订阅完成集群任务变更；自定义group控制是否执行；通过Listener记录任务执行详情
4. 使用SSE优化批量同步金蝶过程中的信息反馈
5. 优化慢SQL

### 3. 华为开发者社区SDN-WAN沙箱项目开发
为开发者社区演示SDK控制器操作物理网络设备,最终播放远端高清视频流。
- 技术栈：
Structs2+JSP+VLC
- 个人贡献：
1. 完成复杂JSON的encode/decode，兼容不同版本
2. HttpUrlConnection解决http数据分包传输

### 4. NCE-Super控制器
基于华为ServiceStage的微服务开发。总共近80个微服务，每个微服务中集成启动以及SQL变更脚本，通过封装Jekins功能实现CICD。
- 个人贡献：
1. 大量CodeReview及代码质量修复，大量单元测试(LLT)
2. ForkJoinTask优化任务执行速度

# 教育经历
|学校|专业|期间|
|---|---|---|
|中国地质大学(武汉)|计算机技术|2012-2014|
|湖北师范大学|信息与计算科学|2007-2011|

