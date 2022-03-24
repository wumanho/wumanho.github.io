---
weight: 2
title: "利用 Github Webhooks 实现博客自动部署"
summary: "顺便试一试 nest.js"
date: 2022-03-24T21:57:41+08:00
draft: false
categories: ["技术"]
tags: ["github","webhooks","CI/CD","nest.js"]
---

## 前言

「好电脑」上线这么久，工作流从来没变过，一直都是：

1. 在本地写作完成后直接构建

2. 手动上传到服务器
3. 删除旧的构建内容

4. 重启 docker 容器

之前还因为图片多导致构建出来的包，越来越大，将图片全部上传到对象存储服务，图片改成 oss 地址引用，真的是完全不嫌麻烦:rofl:

用 Webhooks 来自动部署的想法其实一早就有了，只是一直懒得动手。

循例简单介绍一下，[Github Webhooks](https://docs.github.com/cn/developers/webhooks-and-events/webhooks/about-webhooks) 以每个代码仓库为单位，提供一系列事件钩子，例如`push`、`issue`、`PR`等，基本上涵盖了所有 github 操作，当指定的事件被触发时，Webhook 会向定义好的 URL 地址发送一个 POST 请求，达到通知的目的。

就是这么简单，整体思路也不麻烦，用 node.js 在我的服务器上起一个服务来监听 PUSH 事件，然后调用写好的 shell 脚本执行我之前手动完成的操作就可以了。

然后既然是要用到 node ，且最近又在练习 typesctipt，就想到用一个完全支持 OOP 的后端框架 [Nest.js](https://docs.nestjs.cn/8/introduction)，整个框架的哲学跟 Spring 非常相似，代码结构也是 MVC 的模式，还有依赖注入等基本思想的体现，不过这次的需求比较简单，没有完全使用 Nest 丰富的功能，所以只能说是简单玩一下。

&nbsp;

## 配置 Webhook

![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/webhooks/hookseting.png)

在需要监听的仓库中，跳转到`Setting`页面，选择侧边栏`Webhooks`，点击 `Add webhook`按钮，就可以看到这个界面。

Payload URL：监听服务的 url 地址。

Content type：可选 `JSON` 和 `x-www-form-urlencoded`

Secret：用于加密

实现博客自动部署，只需要监听 push 事件即可。

配置完成后点击 Add webhook 即可，强烈建议设置 Secret，毕竟你的服务应该是直接暴露在公网的：

![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/webhooks/setting1.png)

&nbsp;

## Shell 脚本

```bash
#!/bin/bash

sourcePath="/opt/source"
rootPath="/opt"

# 切换至 sourcePath
cd $sourcePath
# 拉代码
git pull
# 构建
hugo
# 切换至 rootPath
cd $rootPath
# 删除旧的备份文件
rm -rf $rootPath/public_*
# 备份旧的文件
mv $rootPath/public $rootPath/public_`date +%Y%m%d%H%M%S`
# 创建新的文件
cp -r $sourcePath/public $rootPath/public
# 重启容器
docker restart mySite
```

脚本放在跟代码同一个目录下面了

&nbsp;

## node 服务

```typescript
import { Controller, Post, Headers, Req } from "@nestjs/common";
import { AppService } from "./app.service";
import { Request } from "express";

const { execFile } = require("child_process");
const crypto = require("crypto");

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {
  }

  @Post("/build")
  buildBlog(@Req() request: Request, @Headers("X-Hub-Signature-256") publicKey: string): void {
    const GITHUB_SECRET = "这里填自己的 secret";
    const signature = "sha256=" + 
     crypto.createHmac("sha256",GITHUB_SECRET)
    .update(JSON.stringify(request.body))
    .digest("hex");
      
    if (signature !== publicKey) {
      console.log("Invalid key");
      return;
    }
      //执行脚本
    execFile("/opt/blog-bot/exec.sh");
  }
}

```

这是 Nest.js 的控制器，这个熟悉的注解...

将应用打包后，使用`pm2`将 node 服务启动即可

```code
# pm2 start main.js
```

![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/webhooks/serve.png)



大功告成:tada:

&nbsp;

（完）
