---
weight: 2
title: "用 Node.js 写一个简易静态服务器"
date: 2020-05-12T00:29:18+08:00
draft: false
categories: ["技术"]
tags: ["JavaScript","Node.js"]
---

Node.js 是一个开源与跨平台的 JavaScript 运行时环境，通过 Node.js 开发者可以在浏览器之外运行 JavaScript 代码，使得 JS 也拥有了开发后端项目的能力。同时 Node.js 运行 V8 JavaScript 引擎（Google Chrome 的内核），这意味着 Node.js 拥有足够优秀的的性能。  

![logo](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/nodejslogo.png)

&nbsp;

## 构建基础框架 :hammer_and_pick:

先构建出一个基础 Web 服务器，再完善功能。

```javascript
const http = require('http')  
const fs = require('fs')
const url = require('url')

const port = process.argv[2] || 8888;


const server = http.createServer((request, response) => {
  			/*... ...*/
})


server.listen(port, hostname, () => {
  console.log(`服务器启动成功，请访问： http://${hostname}:${port}/`)
})
```

到目前为止，我们做了以下几件事，首先应用了一些必须的模块，Node.js 具有丰富的标准库，包括对网络的完善支持。

```javascript
const http = require('http')   //用于实现 HTTP 服务器或客户端
const fs = require('fs')       //用于操作文件系统
const url = require('url')	   //用于处理与解析 URL
```

如果用户没有指定端口号，则默认端口号为 8888

```javascript
const port = process.argv[2] || 8888;
```

创建一个服务器

```javascript
const server = http.createServer((request, response) => {
  			/*... ...*/
})
```

启动端口监听

```javascript
server.listen(port, hostname, () => {
  console.log(`服务器启动成功，请访问： http://${hostname}:${port}/`)
})
```

&nbsp;

## 实现服务器功能 :wrench:

可以将整个服务器代码分成几个部分去理解：

1. 获取查询参数；
2. 设置响应参数；
3. 路由查询；
4. 发送响应；

```javascript
const server = http.createServer(function(request, response){
    let parsedUrl = url.parse(request.url, true)
    let pathWithQuery = request.url
    let queryString = ''
    if(pathWithQuery.indexOf('?') >= 0){ queryString = pathWithQuery.substring(pathWithQuery.indexOf('?')) }
    let path = parsedUrl.pathname
    let query = parsedUrl.query
    let method = request.method

    response.statusCode = 200
   
    const filePath = path === '/' ? '/index.html' : path
    const index = filePath.lastIndexOf('.')
    
    const suffix = filePath.substring(index)
    const fileTypes = {
        '.html':'text/html',
        '.css':'text/css',
        '.js':'text/javascript',
        '.png':'image/png',
        '.jpg':'image/jpeg'
    }
    response.setHeader('Content-Type',
        `${fileTypes[suffix] || 'text/html'};charset=utf-8`)
    let content
    try{
        content = fs.readFileSync(`./public${filePath}`)
    }catch(error){
        content = '文件不存在'
        response.statusCode = 404
    }
    response.write(content)
    response.end()
})
```

### 获取查询参数 :mailbox_with_mail:

获取用户请求的各种信息，方便后面使用，包括路径、请求方法、请求参数等等。本示例没有实现查询参数对应的功能，只是列举比较常用的请求信息，也可以根据情况只获取所需的即可，不必全部获取。

```javascript
let parsedUrl = url.parse(request.url, true) //获取 url 对象
let pathWithQuery = request.url 			 //获取用户请求路径（带请求参数） 字符串

//获取查询参数字符串
let queryString = ''
if(pathWithQuery.indexOf('?') >= 0){ queryString = pathWithQuery.substring(pathWithQuery.indexOf('?')) }

let path = parsedUrl.pathname				 //获取用户请求路径（不带请求参数）字符串			
let query = parsedUrl.query					 //获取查询参数对象
let method = request.method					 //获取请求方法	
```

设置响应 HTTP 状态码

```javascript
response.statusCode = 200
```

初始化首页，如果用户没有指定具体访问路径，自动跳转到 index.html，filePath 也是不带查询参数的用户请求路径。

```javascript
const filePath = path === '/' ? '/index.html' : path
```

### 设置响应参数 :memo:

这一步主要目标是根据用户请求的资源位置，返回对应的 Content-Type，返回合适的数据格式。  

获取后缀

```javascript
const index = filePath.lastIndexOf('.')
const suffix = filePath.substring(index)
```

实现一个 fileType 哈希表，根据获取到的后缀，设置 Content-Type 格式

```javascript
//文件类型哈希表
    const fileTypes = {
        '.html':'text/html',
        '.css':'text/css',
        '.js':'text/javascript',
        '.png':'image/png',
        '.jpg':'image/jpeg'
    }
    //设置响应头，如果用户请求的类型不属于上述类型，Content-Type 默认为 text/html
    response.setHeader('Content-Type',
        `${fileTypes[suffix] || 'text/html'};charset=utf-8`)
```

### 路由查询 :thinking:

这一步将会根据用户查询的资源路径，读取对应的静态资源返回，如果找不到对应资源，则设置状态码为 404，返回文件不存在提示。

```javascript
let content
    try{
        content = fs.readFileSync(`./public${filePath}`)
    }catch(error){
        content = '文件不存在'
        response.statusCode = 404
    }
```

### 发送响应 :incoming_envelope:

最后，将响应返回，并关闭连接。

```javascript
response.write(content)
response.end()
```

&nbsp;

（完）

&nbsp;

### 参考 :sunflower:

[Node.js API文档](http://nodejs.cn/api/)

