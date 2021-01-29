---
weight: 2
title: "利用 jQuery 发送 AJAX 请求"
date: 2020-04-18T11:36:57+08:00
draft: false
categories: ["前端"]
tags: ["JavaScript","jQuery","AJAX"]
---
&nbsp;

AJAX 全称 *ASynchronous JavaScript And XML*，异步 JS 和 XML，本质上是利用浏览器提供的全局构造函数 **XMLHttpRequest** 创建的对象来发送和接受请求的一种技术。
&nbsp;

由于 AJAX 是异步的（这里不对同步异步作更多介绍），这意味着利用 AJAX 可以同时向后台请求多个数据，而不必等待同步阻塞的时间，这对与效率和用户体验都是非常大的提升。  
&nbsp;

## 原生 JS 实现 AJAX 请求 :wave:
利用原生 JS 发送 AJAX 请求，只需要记住四个步骤：  
```JavaScript
//假设我们有一个单击事件
button.onclick = ()=>{
    //1.创建 XMLHttpRequest 对象
    let req = new XMLHttpRequest()
    
    //2.调用 open 函数，指定请求方法和url
    req.open("GET","/xxx")
    
    //3.判断结果，因为 AJAX 是异步请求，所以需要传递回调函数接收响应
    req.onreadstatechange = ()=>{
        //readyState 状态等于4，并且 http 状态码为 200，即请求成功
        if(req.readyState === 4 && req.status === 200){
            console.log(req.response)
        }
    }else{
        console.log("请求出错")
    }
    
    //4.记得最后要调用 send 发送请求
    req.send()
}
```

也可以稍微封装一下 ，方便处理成功和失败两种情况（非 `promise` 写法）  

```javascript
ajax = (method,url,options)=>{
    //ES6 析构赋值语法
    let {success,fail} = options
    //首先还是创建对象
    let req = new XMLHttpRequest()
    //调用open
    req.open(method,url)
    //成功就调用 succuss,失败就调用 fail
    req.onreadstatechange = ()=>{
        if(req.readyState === 4){
            if(req.status < 400){
                success(req.response) //简化一下，状态码小于 400 都认为是成功，调用seccuss
            }else if(req.status > 400){
                fail(req,req.status)  //失败调用fail
            }
        }
    }
    //发送
    req.send()
}

/*调用*/
ajax("GET","/xxx",{
    succecc(resp){/*...*/},    //传递成功回调
    fail(req,status){/*...*/}  //传递失败回调
})
```

&nbsp;

## 使用 jQuery 发送请求 :dash:

使用原生方式发送请求还是比较复杂的，而且更多功能还需要手动实现，例如如果需要添加 post 请求体，则需要在 `send()` 函数中传递。

所以先不折腾，只需要简单理解即可，因为生产中几乎没有人会用原生 JS 的方式请求数据，实际上 jQuery 为我们提供了更强大、使用更简单的 API，而且从 jQuery 1.5 开始，`$.ajax()`返回的  [jqXHR](https://www.jquery123.com/jQuery.ajax/#jqXHR) 对象 实现了 Promise 接口, 使它拥有了 Promise 的所有属性，方法和行为。  



语法( jQuery 1.5 )：`jQuery.ajax(url, [settings])`，它会返回一个 [jqXHR](https://www.jquery123.com/jQuery.ajax/#jqXHR) 对象，`settings `是可选的，列举一些常用参数：

- method：发送的Method，缺省为`'GET'`，可指定为`'POST'`、`'PUT'`等；
- contentType：发送POST请求的格式，默认值为`'application/x-www-form-urlencoded; charset=UTF-8'`，也可以指定为`text/plain`、`application/json`；
- data：发送的数据，可以是字符串、数组或 object。如果是 GET 请求，data 将被转换成 query 附加到 URL 上，如果是 POST 请求，根据 contentType 把 data 序列化成合适的格式；
- dataType：接收的数据格式，可以指定为`'html'`、`'xml'`、`'json'`、`'text'`等，缺省情况下根据响应的`Content-Type`自动判断。

```javascript
//发送一个 get 请求，接受一个 json 格式的返回值
$.ajax("/xxx",{
    dataType:"json"
})
```

&nbsp;

## jQuery 处理 AJAX 异步回调 :arrows_counterclockwise:

如上述所说，1.5 之后 jQuery 提供了类似 Promise 的方法来处理各种回调，解决了一些在使用传统异步和回调的方式下的问题。

**:warning:需要注意的是**，`jqXHR.success()`, `jqXHR.error()`, 和 `jqXHR.complete()`回调从 jQuery 1.8开始 被弃用。他们将最终被取消，您的代码应做好准备，使用以下展示的 API `jqXHR.done()`, `jqXHR.fail()`, 和 `jqXHR.always()` 代替。[详情可参考官方文档](https://www.jquery123.com/jQuery.ajax/#jqXHR)

&nbsp;

- **jqXHR.done(function(data, textStatus, jqXHR) {});**

一个可供选择的 success 回调选项的构造函数，`.done()`方法取代了的过时的`jqXHR.success()`方法。请参阅`deferred.done()`的实现细节。

- **jqXHR.fail(function(jqXHR, textStatus, errorThrown) {});**

一种可供选择的 error 回调选项的构造函数，`.fail()`方法取代了的过时的`.error()`方法。请参阅`deferred.fail()`的实现细节。

- **jqXHR.always(function(data|jqXHR, textStatus, jqXHR|errorThrown) { });**

  一种可供选择的 complete 回调选项的构造函数，`.always()`方法取代了的过时的`.complete()`方法。

  &nbsp;

:heavy_check_mark:所以，目前推荐的使用方法是：

```javascript
$.ajax("/api/v1/xxx",{
    dataType: 'json'
}).done((data)=>{
    console.log("data:" + JSON.stringify(data))
}).fail((xhr)=>{
    console.log("error:" + xhr.status)
}).always(()=>{
    console.log("complete")
})
```

&nbsp;

(完)

&nbsp;

### 参考

[jQuery 中文文档](https://www.jquery123.com/jQuery.ajax/)

[AJAX - 廖雪峰的官方网站](https://www.liaoxuefeng.com/wiki/1022910821149312/1023023601676640)

