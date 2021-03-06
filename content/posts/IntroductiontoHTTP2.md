---
weight: 2
title: "利用 HTTP/2 为网站提速"
summary: "HTTP/2 协议简介以及如何为你的个人网站启用 HTTP/2"
date: 2020-06-17T09:50:23+08:00
draft: false
categories: ["技术"]
tags: ["HTTP/2"]
---

## 为什么需要 HTTP/2

在介绍 HTTP/2 之前，我们首先要讨论一下当前被广泛使用的 HTTP/1.x ，如果你是一名 WEB 开发者，那么你一定不会对这个名词感到陌生。  

HTTP 协议建立在 TCP 协议的基础之上，是网络传输中重要的一环，举个例子，假设当我们用浏览器向一个网站发送一个静态资源请求，那么必然会经历如下几个过程：

1. 建立 TCP 连接：就是经典的三次握手，浏览器与服务器建立可靠的 TCP 连接；
2. 客户端发送请求：建立 TCP 连接后，客户端会向服务端发送一个 HTTP 请求，该请求包含了请求头、请求体等描述了客户端相关的信息以及所需要申请的资源；
3. 服务器响应：服务器接收到请求后，进行路由、鉴权、获取客户端请求资源等等一系列处理后，返回给客户端一个 HTTP 响应信息；  

![HTTP/1.1 通信过程](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/http2/http01.png)

&nbsp;

由于 TCP 是全双工的工作模式，意味着收发可以同时进行，但是 HTTP/1.x 却是半双工的，也就是说接受和发送只能同时做一个，协议要么是发送状态，要么是接受状态，可以说 HTTP 是对 TCP 传输能力的一种浪费。  

于是 HTTP/1.1 引入了管道机制（pipelining），pipelining 基于 HTTP/1.1 支持的*长连接* 特性，将多个 HTTP 请求放到一个 TCP 连接中一一发送，也就是说在服务器处理前面的请求时，客户端依然可以发送请求而无需等待。  

不过，这依然只是一种低效的全双工模式，因为 HTTP/1.x 有严格的串行返回响应机制，通过 TCP 连接返回响应时，只能串行请求，前一个响应没有完成，下一个响应就不能返回，这样如果前一个请求的回应特别慢，后面就会有更多的请求排队等待，这就是 HTTP 的“队头阻塞”问题。     

当然，你可以在选择队伍时候就做好功课，去排一个你认为最快的队伍，或者甚至另起一个新的队伍。  

在 HTTP/1.1 下，浏览器确实支持同时打开多个 TCP 会话，但不管怎么样，你总归得先选择一个队伍，而且一旦选定之后，就不能更换队伍了，另外另起新队伍会导致资源耗费和性能损失。  

这种另起新队伍的方式只在新队伍 数量很少的情况下有作用，因此它并不具备可扩展性。

&nbsp;

## HTTP/2

HTTP/2 来源于 Google 开发的一个实验性协议 SPDY ，在 2015 年发布，至今已经有五个年头。

与 HTTP/1.x 相比，HTTP/2 的主要变化在于性能提升，HTTP/2 没有改动 HTTP 的核心概念，例如 HTTP 方法、状态代码、URI 和标头字段等，不过 HTTP/2 修改了数据格式化以及在客户端与服务器间传输的方式，解决了 HTTP/1.x 的性能限制。

### 二进制分帧层

HTTP/2 定义了一种全新 HTTP 消息封装格式。

![HTTP2 编码](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/http2/http02.png)

从上图可以看到，与 HTTP/1.x 使用纯文本编码不同，HTTP/2 将所有传输的信息分割为更小的消息和帧，一个 POST 请求，请求行、请求头等内容被分割到 *HEADERS* 帧，而请求体内容则被分割到  *DATA* 帧，并采用**二进制格式**对它们编码，这也是实现多路复用的关键。

### 多路复用

说明 「二进制分帧层」 如何改变客户端与服务器的数据交换方式，需要了解 HTTP/2 另外三个重要概念：

* 流（Stream）：已建立的连接内的双向字节流，可以承载一条或多条消息。
* 消息（Message）：与逻辑请求或响应消息对应的完整的一系列帧。
* 帧（Frame）：HTTP/2 通信的最小单位，每个帧都包含帧头，至少也会标识出当前帧所属的数据流。

![数据交换方式](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/http2/http03.png)

上图可以看到，一个 TCP 连接（Connection）是一条双向的运输通道，一个连接可以包含多个**流**，每个流包含了一端向另一端发送的**消息**，每个消息又被拆分成了几个**帧**，这些消息被发送出去之后，同一个流的帧会在另一端被重新组合，还原成一个完整的 HTTP 请求或响应。

这就是为什么说利用二进制而非文本的方式传输，是性能提升的关键。

如上文所说，HTTP/1.x 如果客户端想发起多个并行请求以提升性能，则必须使用多个 TCP 连接。而且，由于没有流的概念，在使用并行传输传递数据时，接收端在接收到响应后，不能区分多个响应分别对应的请求，所以不能将多个响应的结果重新进行组装，也就实现不了真正的多路复用，造成性能浪费。

HTTP/2 中新的二进制分帧层突破了这些限制，实现了完整的请求和响应复用：客户端和服务器可以将 HTTP 消息分解为互不依赖的帧，然后交错发送，最后再在另一端把它们重新组装起来。

![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/http2/http04.png)

关于 HTTP/2 的更多新特性，请参考：[Google Developers HTTP/2 简介](https://developers.google.com/web/fundamentals/performance/http2)

&nbsp;

## 为服务器启用 HTTP/2

启用 HTTP/2 有一个前提，就是必须同时启用 HTTPS 。

所以，以本站为例，我使用了一个 Nginx Docker 容器作为服务器，并启用了 HTTPS ，那么开启 HTTP/2 非常简单，首先需要确保基础环境版本：

- Nginx 版本不低于 1.9.5
- OpenSSL 版本不低于 1.0.2 版本

然后，在 Nginx 配置文件里，在 listen 后面添加 `http2`：

```bash
server{
	
	listen 443 ssl http2;
}
```

重启 Nginx 容器，HTTP/2 就生效了。

![效果图](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/http2/http05.png)

&nbsp;

（完）

&nbsp;

### 参考

[Google Developers HTTP/2 简介](https://developers.google.com/web/fundamentals/performance/http2)