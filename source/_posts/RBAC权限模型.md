---
title: RBAC权限模型
date: 2024-01-16 17:05:39
tags: 
- RBAC
categories: 其他
---
权限管理模型有很多，但是最为经典实用的还是RBAC。若想设计权限模型或对其进行扩展，了解RBAC是必不可少的功课
<!--more-->
# RBAC权限模型

## Role-BasedAccess Control

* 传统：直接对用户授权
* RBAC: 基于角色的权限管理

​![image](/images/assets/image-20240116165959-vy9w5gu.png)​

‍

## RBAC0 : 用户与权限分离

​![image](/images/assets/image-20240116170020-sq3x22z.png)​

‍

## RBAC1 :角色存在层级，低级角色可以继承高级角色权限

​![image](/images/assets/image-20240116170034-fs3xv9q.png)​

‍

‍

## RBAC2 : 角色存在互斥关系，获取有限制条件

​![image](/images/assets/image-20240116170050-p7vvxyi.png)​

* 静态：互斥角色无法赋予同一用户
* 动态：登录时切换角色

‍

## RBAC3 统一角色模型

RBAC3=RBAC1+RBAC2

---

- 思考  
RBAC基本能满足大部分的授权要求，但是在一些实际业务场景中，时常会遇到需要对部分数据、按钮的进行“隐藏”（或授权才可访问）的需求。  
这时候，基于属性的访问控制模型(ABAC: Attribute-Based Access Control)或可满足需求。但是这种过于分散的模型本身并无实际意义，在实际业务中还是需要去落实到具体组件、行为。  

比如：  
1、RBAC模型中可以建立互斥抽象的高级角色来对应一些限制行为  
2、对于访问类要求，可以参考JeecgBoot的PermissionData](http://doc.jeecg.com/2044046)，允许用户自定义访问规则

