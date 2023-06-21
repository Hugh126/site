---
title: Gradle入门
date: 2023-04-11 15:10:05
tags:
---
现在越来越多的开源项目都使用gradle来构建，而且基于gradle灵活的语法和增量构建等特性，在大型项目构建的时候大大节约了时间，实乃高效干活的利器，不得不学起来~
<!--more-->


# 1. Gradle与 Maven的比较  
> Gradle和Maven两种构建方式存在一些根本差异。 Gradle基于任务依赖关系图-其中任务就是工作，而Maven是基于固定的过程和线性模型。使用Maven构建项目时，目标将附加到项目阶段，目标的作用类似于Gradle的任务，即“完成任务的事物”。  
>
> 1.1 在性能方面，两者都允许多模块构建并行运行。但是，Gradle允许增量构建，因为它检查是否更新了哪些任务。如果是这样，则不执行任务，从而使构建时间大大缩短。Gradle上其他出色的性能功能包括：  
>- Java类的增量编译
>- 防止反编译
>- 对增量子任务使用API
>- 编译器守护程序加快编译速度  
> 
> 1.2 在管理依赖项时，Gradle和Maven都可以处理动态和传递性依赖项，以使用第三方依赖项缓存，并读取POM元数据格式。还可以通过中央版本控制定义声明库版本并强制执行中央版本控制。两者都从其artifact 仓库下载可传递依赖项。Maven具有Maven Central，而Gradle具有JCenter，也可以定义自己的私人公司存储库。如果需要多个依赖项，Maven可以同时下载它们。



# 2. 安装与配置  
## 2.1 安装
从[安装目录](https://services.gradle.org/distributions/)找一个不那么早的下载，然后配置环境变量：  
`GRADLE_HOME`,然后`%GRADLE_HOME%\bin`目录加入到path  
`gradle -v` 验证  

## 2.2 配置镜像
init.d文件夹下新增 **init.gradle** 设置国内代理
``` bash
allprojects {
    repositories {
        maven { url 'file://D:/maven_repo'}
        mavenLocal()
        maven { name "Alibaba" ; url "https://maven.aliyun.com/repository/public" }
        maven { name "Bstek" ; url "http://nexus.bsdn.org/content/groups/public/"}
        mavenCentral()
    }
    buildscript { 
        repositories { 
            maven { name "Alibaba" ; url 'https://maven.aliyun.com/repository/public'}
            maven { name "Bstek" ; url 'http://nexus.bsdn.org/content/groups/public/' }
            maven { name "M2" ; url 'https://plugins.gradle.org/m2/'}
        }
    }
}
```
**注意：** 这里有个指向本地目录是gradle本地仓库，其实也可以是maven本地仓库  


## 2.3 IDEA配置与wrapper  
如果我们直接使用IDEA新建一个用gradle构建的项目，会发现它的结构有点不一样：  
``` 
-src
-gradle
	-warpper
		-gradle-wrapper.jar
		-gradle-wrapper.properties
-build.gradle #构建脚本，类似pom.xml
-gradlew #Linux系统的包装启动脚本
-gradlew.bat #windows系统的包装启动脚本
-settings.gradle #定义项目信息
```
warpper文件夹其实是对gradle的一层包装，为了：  
1. 在未安装gradle的环境运行  
2. gradle的发展太快了，有时候需要切换版本，也可以使用gradlew代替。这样就使用的是附带jar的版本了。  
gradle-wrapper.properties
``` ini
# Gradle解包后存储的父目录
distributionBase=GRADLE_USER_HOME
# 指定目录的子目录
distributionPath=wrapper/dists
# Gradle指定版本的压缩包下载地址
distributionUrl=https\://services.gradle.org/distributions/gradle-7.6.1-bin.zip
# Gradle压缩包下载后存储父目录
zipStoreBase=GRADLE_USER_HOME
# Gradle压缩包下载后存储子目录
zipStorePath=wrapper/dists
```
`GRADLE_USER_HOME`类似maven的本地仓库，如果不指定一般会放到用户\.gradle目录，C盘估计又要爆炸了。这里可以指定为maven的本地仓库目录，然后设置到环境变量。虽然它并不会使用maven的缓存（两者缓存格式不一致）。  

以下是gradlew工作流程：  
![gradlew工作流程](https://docs.gradle.org/current/userguide/img/wrapper-workflow.png)  

以下是IDEA的配置截图
![](/images/gradle-idea-config.png)  

# 3. Gradle使用  
常用gradle指令,与maven类似  
```  
# 构建项目
gradle build
# 清空build目录
gradle clean
# 编译测试代码，生成测试报告
gradle test
# 跳过测试构建
gradle build-x test
```
IDEA自带的Gradle集成了这些常见命令，另外还对应用也做了支持：  
比如这里可以直接Run应用的Main：  
![](/images/gradle-command.png)

---
---
关于gradle自定义任务及groovy语言，以后需要再深入研究吧。。

Groovy语法参考：   
http://www.groovy-lang.org/testing.html#_introduction

gradle_wrapper更多：  
https://docs.gradle.org/current/userguide/gradle_wrapper.html#gradle_wrapper

