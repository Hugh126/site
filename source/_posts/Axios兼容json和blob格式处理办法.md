---
title: Axios兼容json和blob格式处理办法
date: 2024-12-08 12:07:10
tags:
- Axios
categories: 前端
---
日常开发经常会遇到文件请求的需求，那如何优雅地处理正常场景下获取二进制文件，以及异常场景反馈异常信息呢
<!--more-->

> 先说结论:  
> 前端请求时使用`responseType: 'blob'`预先声明。  
> 1. 正常场景：后端返回二进制文件时添加文件类型Header如：`Content-Type : application/octet-stream`,此时前端获取内容即为二进制文件 
> 2. 异常场景：后端添加`Content-Type : application/json`直接返回带有异常信息的JSON。此时，前端需要将blob对象转JSON

## 1. Axios简介
Axios 是一个基于[promise](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise)的网络请求库。除了支持Promise的API，还支持取消请求和超时处理，支持自动序列化，支持请求和响应的拦截。使用简单，功能强大  
使用示例：
- 发起一个post请求
``` js
axios({
  method: 'post',
  url: '/user/12345',
  data: {
    firstName: 'Fred',
    lastName: 'Flintstone'
  }
});
```
- 用GET请求获取远程图片
``` js
axios({
  method: 'get',
  url: 'http://bit.ly/2mTM3nY',
  responseType: 'stream'
})
  .then(function (response) {
    response.data.pipe(fs.createWriteStream('ada_lovelace.jpg'))
  });
```

## 2. Axios前端处理
```
import axios from 'axios'

  downloadById(directoryId) {
    const url = '...';
    axios({
      method: 'get',
      url: url,
      responseType: 'blob',
      headers: { 'Authorization': '...' }
    }).then((res) => {
      const contentType = res.headers['content-type'];
      if (contentType.includes('application/json')) {
        // 处理 JSON 格式的响应（错误信息）
        const resText = await data.text();
        const rspObj = JSON.parse(resText);
        const errMsg = rspObj.msg || rspObj.code
      }else {
        // 获取二进制对象
        const blob = new Blob([res.data])
        // 可以使用 window.URL.createObjectURL(blob)方法创建临时URL（注意使用window.URL.revokeObjectURL）
        // 也可以保存到本地文件
        ... 
      }
    })
  }
```



## 3. 后端响应
ResponseEntity 是Spring框架中用于表示HTTP响应的实体，它封装了响应的状态码、响应头和响应体内容。它可以在控制器方法中用来返回响应，具有更高的灵活性和可控制性  
``` java
    @GetMapping("/downloadFile")
    @ResponseBody
    public ResponseEntity downloadFile(@RequestParam Long directoryId, HttpServletResponse response, HttpServletRequest request){
        if (checkInvalid) {
            return ResponseEntity.ok().body(AjaxResult.error("没有下载权限"));
        }else {
            return  commonController.downloadFile(directoryId, response, request);
        }
    }
```

## 4. 其他
1. 按照前面所说，是不是可以在前端请求时候，不要类型声明。然后它会自动转换成你想要的类型？  
很遗憾， 我测试了没有。不声明类型的时候，正常返回二进制的时候，js判断类型也不是blob，应该是当成默认的字符串处理了。 
2. 是不是同样声明接受json，然后字符串转二进制。理论上可行，但实际上大部分时候（正常场景）都需要将字符串转二进制是不合理的。  
3. 后端在返回前端时候，是否应该设置500或者其他的状态码？这个看各自的编码规范，需要注意的是，Axios提供了请求和响应拦截器。如果在response拦截器中对响应状态码有特殊处理，请注意适配。参考：  
``` js
// 添加请求拦截器
axios.interceptors.request.use(function (config) {
    // 在发送请求之前做些什么
    return config;
  }, function (error) {
    // 对请求错误做些什么
    return Promise.reject(error);
  });

// 添加响应拦截器
axios.interceptors.response.use(function (response) {
    // 2xx 范围内的状态码都会触发该函数。
    // 对响应数据做点什么
    return response;
  }, function (error) {
    // 超出 2xx 范围的状态码都会触发该函数。
    // 对响应错误做点什么
    return Promise.reject(error);
  });
```
4. 由于axios支持finally，因此最好是处理error，并在finally中定义好回调。以下是增加一个loading遮罩的例子：
``` js
    // 调用
      const loading = this.$loading({ // 声明一个loading对象
        lock: true, // 是否锁屏
        text: '正在下载...', // 加载动画的文字
        spinner: 'el-icon-loading', // 引入的loading图标
        background: 'rgba(0, 0, 0, 0.7)', // 背景颜色
        target: '.sub-main', // 需要遮罩的区域
        body: true,
        customClass: 'mask' // 遮罩层新增类名
      })
      this.$download.downloadById(row.id, function () {
        loading.close()
      });

      // 封装
      downloadById(directoryId, callback) {
        axios({
            method: 'get/post',
            url: url,
            responseType: 'blob',
            headers: { 'Authorization':  ''}
          }).then((res) => {
            
          }).catch(err => {

          }).finally(() => {
            callback()
          });
}
```

