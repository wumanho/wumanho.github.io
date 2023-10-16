---
weight: 2
title: "【IDEA】Windows 配置通过终端命令行启动 Webstorm"
summary: "像 vscode 一样快捷启动 Webstorm"
date: 2022-08-10T21:17:29+08:00
draft: false
categories: ["前端"]
tags: ["Webstorm","命令行"]
---

## 前言

发现有些前端不知道环境变量怎么用，可以[参考我之前的博客文章](https://wumanho.cn/posts/cmd/#shell-%E7%9A%84%E6%9F%A5%E6%89%BE%E6%9C%BA%E5%88%B6-)。

&nbsp;

## 配置与命令行使用

找到你的 Webstorm 安装位置，例如 `"C:\Program Files\JetBrains\WebStorm 2021.2.2\bin\webstorm64.exe"`，复制一下

1. 我的电脑右键 => 属性
2. 高级系统设置 => 环境变量
3. 找到系统变量中的 `Path`，点击编辑 => 点击新建，把刚才的安装位置粘贴进去
4. 保存

![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/ws/wseve.jpg)

为了方便使用，建议进入你的 Webstorm 安装目录，将 `webstorm64.exe` 以及 `webstorm64.exe.vmoptions` 重命名为 `ws.exe` 和 `ws.exe.vmoptions`

最后注意将桌面快捷方式或者别的快捷方式指向的应用程序也重命名一下或者重新建一个快捷方式。

&nbsp;

### 通过绝对路径打开项目

在任意地方启动终端，执行 `ws` + 项目所在目录，例如：

```
PS C:\Users\wuman\Desktop> ws D:\codeSpace\core\
```

&nbsp;

### 通过相对路径打开目录

进入到项目所在目录，启动终端并执行 `ws`，例如：

```
PS C:\Users\wuman\Desktop> cd D:\codeSpace\core\
PS D:\codeSpace\core> ws
```

&nbsp;

### 打开指定的文件并具体到某一行

使用 `--line` 参数，例如：

```
PS C:\Users\wuman\Desktop> cd D:\codeSpace\core\
PS D:\codeSpace\core> ws --line 53 packages/reactivity/src/effect.ts
```

&nbsp;

### 对比两个文件的差异

使用 `diff` 参数，例如：

```
PS C:\Users\wuman> cd D:\codeSpace\core\packages\reactivity\src
PS D:\codeSpace\core\packages\reactivity\src> ws diff .\effect.ts .\effectScope.ts
```

![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/ws/wsdiff.jpg)

