---
weight: 2
title: "【CAC】源码阅读笔记 - 源码解析"
date: 2022-07-09T14:08:52+08:00
draft: false
hiddenFromHomePage: true
categories: ["前端"]
tags: ["CAC","TypeScript","Node.js"]
---

## 快速跳转

[上一篇：CAC 工程化介绍](https://wumanho.cn/posts/cac-sourcecode/)

&nbsp;

# CAC 使用介绍

上一篇着重介绍了 CAC 项目中用到的工程化技术，但对 CAC 本身却只字未提，所以有必要先来介绍一下 CAC 到底是什么，以及怎么用。

CAC 用一句话介绍，就是一个**用于帮助用户构建命令行风格应用程序的 Nodejs 框架**，让用户可以像使用 Linux 命令一样执行 Nodejs 应用。

项目根目录中有一个`examples`目录，里面存放了一些指导用户使用的示例代码，简单介绍几个：

## basic-usage.js

```javascript
/** basic-usage.js **/
require('ts-node/register')
const cli = require('../src/index').cac()

cli.option('--type [type]', 'Choose a project type')

const parsed = cli.parse()

console.log(JSON.stringify(parsed, null, 2))
```

以上代码演示了一个基本使用方式，其中包含了 CAC 的核心实例以及其中两个关键 API，分别是`option`以及`parse`。

### option

option 函数用于为应用创建选项，可以包含多个，默认情况下 option 创建的选项属于全局选项，在用户添加了 command 子命令时，option 属于该子命令

### parse

parse 函数非常关键，用于执行匹配的命令的回调函数、输出帮助信息、输出版本号等关键操作

&nbsp;

## command-options.js

```javascript
require('ts-node/register')
const cli = require('../src/index').cac()

cli
  .command('rm <dir>', 'Remove a dir')
  .option('-r, --recursive', 'Remove recursively')
  .action((dir, options) => {
    console.log('remove ' + dir + (options.recursive ? ' recursively' : ''))
  })

cli.help()

cli.parse()
```

### command

在默认情况下，cac 会创建一个全局的 globalCommand，而 command 函数用于创建一条自定义子命令，实际上是创建了一个新的 Command 实例，在新的实例后面调用的函数（例如 option 函数）都只属于该 Command 实例

### action

action 函数定义一个回调函数，必须在调用 command 函数后，获取到子命令的实例才能调用。action 的作用是当匹配到对应的子命令时，执行回调函数

### help

help 函数用于为命令创建帮助文档

&nbsp;

## 其他

### example

example 函数接受一个字符串，会输出到帮助文档中，例如：

```javascript
cli
  .command('deploy [path]', 'Deploy to AWS')
  .option('--token <token>', 'Your access token')
  .example('deploy ./dist')

cli.help()

cli.parse()
```

```code
# cli deploy --help

Usage:
  $ sub-command.js deploy [path]

Options:
  --token <token>  Your access token 
  -h, --help       Display this message 

Examples:
deploy ./dist
```

&nbsp;

# CAC 实例详解

（持续更新中 ...）



