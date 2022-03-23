---
weight: 2
title: "【Cloudflare Workers】无服务器应用初体验"
summary: "Serverless 听得不少,来体验看看"
date: 2022-02-12T19:29:49+08:00
draft: false
categories: ["技术"]
tags: ["Cloudflare","Serverless","workers"]
---

## Serverless 简介

随着云服务发展日渐完善，以亚马逊云的 AWS Lambda 为首的 Serverless（无服务器应用）服务早在几年前就已经开始普及。

Serverless 服务商提供了一个无需预置或管理基础设施即可运行代码的环境，换句话说就是现在不仅不需要管理自己的硬件服务器、网络等，甚至连购置云服务器或者容器平台都不再需要，只要用户编写和部署代码到 Serverless 平台即可，真正做到了只关注业务而无需再担心基础设施的运维层面，大大降低成本和增加了灵活性。

&nbsp;

## Cloudflare workers

[Cloudflare](https://www.cloudflare.com/zh-cn/)（以下简称 CF）可能很多人不陌生，他们是全球最大的 CDN 服务供应商，但 CDN 不是我们这次讨论的重点。

CF 借助于他们分布在全球每个角落的大量边缘节点的优势，与传统无服务供应商将 Serverless 部署与各个中心机房不同，CF workers 将用户的函数服务部署在 他们的边缘网络（Edge Network）中，这使函数服务更加接近于客户端测，从而提供更优质的体验。

### 费用

CF workers 一共有两个 plans 可以选择：

即使是免费版，每天也可以支持10万个请求和10万次 KV 存储读取操作，虽然会限制请求的性能（最多占用 10ms cpu 时间），但个人使用的话应该是足够了。

![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/cloudflare/cfplan.png)

&nbsp;

## 开始体验

这次体验，我将尝试使用 CF workers 创建一个简单的查看天气的应用。

### 创建 workers

要使用 CF workers，首先需要创建一个 CF 的账号，这个就不再赘述了。

创建账号完成后，打开控制台，选择 Workers，你将需要创建一个全球唯一的`子域`，概念跟创建对象存储时的子域是一样的，指定好子域后，就可以开始创建服务了。

![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/cloudflare/%E5%88%9B%E5%BB%BA.png)

创建服务需要指定`服务名称`，`服务名称`将决定了你的 Serverless 服务最终的路径。

选择一个启动器，初始启动器有两个，选择哪个作为启动器实际上**并无区别**，只是如果选择第一项（简介）初始化代码会更多一些，这次我会选择第二个启动器，即HTTP 处理程序，选择完成后点击创建服务：



![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/cloudflare/start.png)

服务创建完成后，可以在控制台看到当前服务的状态，包括**请求数**、**持续时间**和**中值CPU时间**等状态信息，以及服务的全局地址：

![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/cloudflare/status.png)

### 编辑代码

CF workers 提供多种编辑代码的方式，推荐的方式是使用 Workers CLI 来初始化一个项目（作用类似 VUE CLI），但这次出于篇幅考虑就不做演示了，需要了解的朋友可以直接参考[官方文档](https://developers.cloudflare.com/workers/get-started/guide)。

这次会演示使用控制台提供的`快速编辑`功能，进入快速编辑，可以看到系统生成了一个初始化的 HelloWorld 函数，CF workers 默认基于 [Node.js](https://nodejs.org/zh-cn/) 作为基础开发语言（其他语言如 RUST 也通过 WebAssembly 支持），尝试发起一个请求看看：

![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/cloudflare/edit1.png)

成功返回了"Hello world "。

在 CF Workers 中，任何传入的HTTP请求都被称为`fetch`事件，然后通过一个`addEventListener`函数作为脚本的触发器，我们可以在`respondWith`方法中定义的回调函数`handleRequest`里面执行我们的业务逻辑。

### 实现天气查询小功能

像我们开始说的，我们来通过 CF workers 实现一个天气查询小功能，只要访问我们的 end point，可以获取到天气信息，天气信息通过[和风天气 api](https://www.qweather.com/)实现。

其实非常简单，只要我们定义一下我们需要返回的 html 格式，指定响应头的 content-type，直接返回给浏览器即可：

```javascript
addEventListener("fetch", event => {
  event.respondWith(handleRequest(event.request))
})

//定义返回的 html 结构
const html = `
<div id="he-plugin-standard"></div>
<script>
WIDGET = {
  "CONFIG": {
    "layout": "2",
    "width": 230,
    "height": 270,
    "background": "1",
    "dataColor": "FFFFFF",
    "key": "这里是和风天气的key"
  }
}
</script>
<script src="https://widget.qweather.net/standard/static/js/he-standard-common.js?v=2.0"></script>
`

async function handleRequest(request) {
  return new Response(html, {
      //设置响应头
    headers: {
      "content-type": "text/html;charset=UTF-8",
    },
  })
}
```

编辑完成后点击**保存并部署**，将服务发布到你的全局地址中：

![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/cloudflare/save.png)

&nbsp;

## 最终效果

发布完成后，在浏览器中输入你的全局地址，即可访问服务：

![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/cloudflare/result.png)

&nbsp;

## 总结

实际上，Serverless 可以做的远远不止是返回一段 html 内容这么简单，它可以结合 KV 存储服务充当一个完整后端功能，或者用来做 SSR 等真正关键的生产需求。得益于 Serverless 服务的灵活性，用户可以利用它开发出各种有用的功能而不许要投入大量的资金，希望这篇文章对你有所启发。



### 参考

[CF workers 官方文档](https://developers.cloudflare.com/workers/)

[和风天气官方文档](https://dev.qweather.com/docs/start/)
