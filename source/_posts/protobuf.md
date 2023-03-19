---
title: protobuf
date: 2023-03-19 11:52:58
tags:
---
## 1. 安装

[3.20RC版本下载地址](https://github.com/protocolbuffers/protobuf/releases/tag/v3.20.1-rc1)  

命令行基本使用:  
```
protoc --proto_path=. --java_out=../../ hello.proto
```
其中，`--proto_path`可以简写为`-I`

> 为啥用老版本？新的22RC版本有bug啊：  
[Java Compile Error on parseUnknownField]（https://github.com/protocolbuffers/protobuf/issues/10695）

如果是IDEA中使用，主要是通过插件调用本地安装的protoc来实现，或者通过org.xolstice提供的maven插件。个人感觉在初期学习的话，直接使用本地安装的protoc更简单明了，后期在项目中集成的话考虑不依赖本地的maven插件。  

[IDEA中集成protoc，讲的全面靠谱的](https://w3sun.com/1287.html)

## 2. 基本用法
推荐prot3  

https://protobuf.dev/programming-guides/proto3/

## 3. grpc
grpc的代码生成依赖于protoc-gen-grpc-java插件  

[插件下载地址](https://repo.maven.apache.org/maven2/io/grpc/protoc-gen-grpc-java/)  
主要是添加两项参数  
``--plugin=protoc-gen-grpc-java=`` 和``--grpc-java_out=``  
用法：
```
protoc -I=. --plugin=protoc-gen-grpc-java=D:\Tools\protoc-3.20.1-rc-1-win64\bin\grpc-java.exe   --java_out=../../   --grpc-java_out=../../    hello.proto
```










