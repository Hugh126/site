---
title: Hexo Guide
date: 2023-03-23 11:45:58
tags: 
- hexo
- blog
description: hexo使用概述
---

### 1、安装
nodeJs安装，hexo安装，设置环境变量  
阅读[官网文档](https://hexo.io/zh-cn/docs/commands)  
注意设置下npm的源  
``` bash
npm config set registry https://registry.npm.taobao.org --location=global
npm config set disturl https://npm.taobao.org/dist --location=global
```


### 2、初始化  
`hexo init blog`  
> 注意: 
> 本命令相当于执行了以下几步：
> Git clone hexo-starter 和 hexo-theme-landscape 主题到当前>目录或指定目录。
 使用 Yarn 1、pnpm 或 npm 包管理器下载依赖（如有已安装多个，则列在前面的优先）。npm 默认随 Node.js 安装。

### 3、编译运行  
此时，已经有一篇hello文章了，可以直接生成静态资源（将md编译为html）和运行（启动一个webServer），然后看效果。  
两个命令可以一起运行   
``` bash
# hexo generate 和 hexo server
hexo g && hexo s
```


### 4、设置主题  
以经典nexT主题为例，只需将nexT放到theme目录下：  
```
git clone https://github.com/next-theme/hexo-theme-next themes/next 
``` 
然后，改下配置
```bash
# vim ./_config.yml
theme: next
```
>[其他主题](https://hexo.io/themes/index.html)  

### 5、新建文章  
```
hexo new protobuf
```
注意到source/_posts已经多了一篇文章 protobuf.md
打开编辑Ta  
> ---标记的是文章的信息，可以看做文章的header，在最下面添加md语法内容。
``` md
## 标题
say Hello
## coding
这里有一份代码
`print("say Hello")`
```
关于文章的基本信息设置，可以参考：  
https://hexo.io/zh-cn/docs/front-matter  


### 6、编译并运行  
```bash
hexo g &&  hexo s
```

### 7、图片资源问题 
对于相对路径图片指向错误，无法显示的问题。  
最后发现官网的资源文件 模块，其实早就给出了说明  
https://hexo.io/zh-cn/docs/asset-folders 
- 安装插件 
```
npm install hexo-renderer-marked --save
```
- 修改配置
```
_config.yml
post_asset_folder: true
marked:
  prependRoot: true
  postAsset: true
```
其实也可以通过安装其他插件，修改部署时图片的路径：  
https://leay.net/2019/12/25/hexo/  

### 8、部署到github  
安装 hexo-deployer-git  
``` bash
$ npm install hexo-deployer-git --save
```
参考[gitpage指南](https://hexo.io/zh-cn/docs/github-pages)  

### 9、设置菜单  
通过主题hexT的能力，参考：
https://theme-next.js.org/docs/theme-settings/custom-pages.html  
修改theme/next/_config.yml  
``` yml
# vi theme/next/_config.yml
sidebar:
  position: left
  display: always
menu:
  home: / || fa fa-home
  about: /about/ || fa fa-user
  tags: /tags/ || fa fa-tags
  categories: /categories/ || fa fa-th
  #archives: /archives/ || fa fa-archive
  #schedule: /schedule/ || fa fa-calendar
  #sitemap: /sitemap.xml || fa fa-sitemap
  commonweal: /404/ || fa fa-heartbeat
```
自定义页面命令
``` bash
hexo new page about
hexo new page tags
hexo new page categories
hexo new page commonweal
```
### 10、添加评论功能  

> nexT主题已经支持多种评论系统，有代表性的有两款：
> - Valine 评论系统  
> 无需登录，需要注册Valine 
> - Gitment评论系统
> 基于Github的issure，需要注册注册 OAuth application  
开始之前在`themes\next\package.json` 看看本地安装next版本。  
- 如果是Next8.1.0以下，建议直接使用[配置Valine](#12-配置valine)  
- 如果是最新8.1以上版本，next已经取消内置Valine，也不建议降级，毕竟后续有新的特性想升级呢？  
配置waline的配置请参考[配置waline](#11-配置waline)
### 11、配置waline
1. 官网的教程已经很清楚了。参照着截图的步骤来， 注意选择国际版
https://waline.js.org/guide/get-started/  
2. 最后`Visit`跳转后地址既是博客留言板的地址，复制Ta直接配置到next的配置文件
``` yml
# Waline
# For more information: https://waline.js.org, https://github.com/walinejs/waline
waline:
  enable: true #是否开启
  serverURL: xxxxxxxx.vercel.app # Waline #服务端地址，我们这里就是上面部署的 Vercel 地址
  locale:
    placeholder: 疑义相与析，畅所欲言，不登录也没关系哒 # #评论框的默认文字
  avatar: mm # 头像风格
  meta: [nick,mail] # 自定义评论框上面的三个输入框的内容
  pageSize: 10 # 评论数量多少时显示分页
  lang: zh-cn # 语言, 可选值: en, zh-cn
  visitor: true # 文章阅读统计
  comment_count: true # 如果为 false , 评论数量只会在当前评论页面显示, 主页则不显示
  requiredFields: [nick] # 设置用户评论时必填的信息，[nick,mail]: [nick] | [nick, mail]
  libUrl: https://unpkg.com/@waline/client@v2/dist/waline.js # Set custom library cdn url
  login: enable
```
### 12、配置Valine
以下介绍Valine的使用方式：
1. 注册账号  
https://console.leancloud.cn/  
2. 配置  
``` yml
valine:
  enable: true 
  appid:  # 将应用key的App ID设置在这里
  appkey: # 将应用key的App Key设置在这里
  notify: false
  verify: false
  placeholder: 留下小脚印~
  avatar: monsterid 
  guest_info: nick,mail,link # 自定义评论标题
  pageSize: 10 # 分页大小，10页就自动分页
  visitor: true # 是否允许游客评论
```  
> 更多valine的配置可以参考：  
> https://blog.csdn.net/wugenqiang/article/details/105744843

3. 管理评论  
进入 LeanCloud 官网，找到 控制台->存储->commet 中进行管理  

