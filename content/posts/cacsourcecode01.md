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

# 源码目录介绍

```shell
src	
├── CAC.ts
├── Command.ts
├── Option.ts
├── __test__						
│   ├── __snapshots__
│   └── index.test.ts
├── deno.ts
├── index.ts
├── node.ts
└── utils.ts
```

在 `src` 目录中，我们可以把里面的文件分成**四个部分**来看：

### deno.ts & node.ts

`deno.ts` 和 `node.ts` 这两个文件做的事情是一样的，都是收集一些**操作系统**的辅助数据，只是分别兼容 deno 环境和 node 环境。

以 `node.ts` 为例，该文件导出了两个变量，分别是 `processArgs` 和 `platformInfo`，其中 `processArgs` 用于作为命令行的默认参数，`platformInfo` 则用于在为打印帮助信息收集数据：

```typescript
export const processArgs = process.argv

export const platformInfo = `${process.platform}-${process.arch} node-${process.version}`
```

### __test__

测试目录

### utils.ts & index.ts

* utils 导出一些工具函数
* index 作为整个库的出口，导出一个 cac 实例，更准确来说应该是一个用于创建 cac 实例的工厂函数

```typescript
import CAC from './CAC'
import Command from './Command'

/**
 * @param name The program name to display in help and version message
 */
const cac = (name = '') => new CAC(name)

export default cac
export { cac, CAC, Command }
```

### CAC.ts & Command.ts & Option.ts

分别对应三个类 `CAC`、`Command` 和 `Option`，主要注意的是，这三个类涵盖了 CAC 这个库的所有核心功能实现。

&nbsp;

# 源码解析

## 类的关系

上面说到，CAC 这个库的所有核心功能由 `CAC`、`Command` 和 `Option` 这三个类实现，所以理清楚这几个类的关系就非常重要，也是对于 OOP 面向对象思想的一种学习。

首先看看 `CAC` ，`CAC` 通过自身的几个属性，直接**关联** `Command` 类，与 `Options` 类**没有依赖关系**：

```typescript
class CAC extends EventEmitter {
  /** 省略 **/
  commands: Command[]			 // 保存所有子命令	
  globalCommand: GlobalCommand   // GlobalCommand 是 Command 的子类，比较特殊，后面介绍
  matchedCommand?: Command 		 // Command 实例
  /** 省略 **/
}
```

然后是 `Command`，`Command` **被** `CAC` **关联**，同时自身也包含对 `CAC` 的**引用**，所以可以说 `Command` 跟 `CAC` **互相关联**：

```typescript
class Command {
  /** 省略 **/
  constructor(
    public cli: CAC
  ) {}
  /** 省略 **/
}
```

此外，`Command` 对于 `Option` 存在**依赖关系**：

```typescript
// 处理 option 
option(rawName: string, description: string, config?: OptionConfig) {
    const option = new Option(rawName, description, config)
    this.options.push(option)
    return this
  }
```

而 `Option` 类则只有被 `Command` 类依赖，自身主要用于处理用户传入的选项，作为一个工具类存在：

```typescript
export default class Option {
  name: string
  names: string[]
  isBoolean?: boolean
  required?: boolean
  config: OptionConfig
  negated: boolean

  constructor(
    public rawName: string,
    public description: string,
    config?: OptionConfig
  ) { /** 省略**/}
}
```

&nbsp;

## CAC

CAC 类是一个最外层的类，他的概念跟接口有一点点类似。

CAC 类提供了所有供用户直接使用的方法，包括 `command`、`option`、`help`、`parse` **等...** 

首先看到的是他继承了 `EventEmitter` 类：

```typescript
class CAC extends EventEmitter {
    /* .... */
}
```

### EventEmitter

继承 `EventEmitter` 意味着可以实现基于事件流的交互动作，让终端用户通过 `.on()` 等常见的手段实现事件监听。

例如，在 CAC 中，用户可以通过 `.on()` 监听某个子命令，当子命令运行时，执行回调函数：

```typescript
// cli 是 CAC 实例，通过 .on() 可以监听 build 命令的运行
cli.on('command:build', () => {
  // Do something
})
```

想要实现这种交互，只需要找到运行命令的代码，通过 `emit()` 函数来触发即可，参考 `parse` 函数：

```typescript
  parse(argv = processArgs,{run = true,} = {}): ParsedArgv {
    /* 省略 ... */
      
    // 遍历执行所有子命令
    for (const command of this.commands) {
      /* 省略 ... */
      const commandName = parsed.args[0]
      // 将命令拼接到 emitter 后面，触发监听
      this.emit(`command:${commandName}`, command)
      }
    }
  }
```

先大致浏览一下 `CAC` 的关键属性和方法：

```typescript
class CAC extends EventEmitter {
  /** name 主要用于在 help 和 version 中使用 */
  name: string
  /** 保存所有子命令 */  
  commands: Command[]
  /** 一个默认的 Command 实例 */  
  globalCommand: GlobalCommand
  /** 记录当前匹配到的子命令 */  
  matchedCommand?: Command
  /** 记录当前匹配到的子命令名称 */  
  matchedCommandName?: string
  /** 记录原始参数 */
  rawArgs: string[]
  /** 解析后的参数 */
  args: ParsedArgv['args']
  /** 选项 */
  options: ParsedArgv['options']
  
  showHelpOnExit?: boolean
  showVersionOnExit?: boolean

  /**
   *  添加一个全局的用法说明
   *  不适用于子命令，子命令的 usage 在 Command 里面	
   */
  usage(text: string) {}

  /**
   * 添加一个子命令
   */
  command(rawName: string, description?: string, config?: CommandConfig) {}

  /**
   * 添加一个全局选项.
   *
   * 全局选项会应用到子命令中.
   */
  option(rawName: string, description: string, config?: OptionConfig) {}

  /**
   * 生成 help 信息，可以通过 `-h, --help` 选项调出
   */
  help(callback?: HelpCallback) {}

  /**
   * 生成 version 信息，可以通过 `-v, --version` 选项调出
   */
  version(version: string, customFlags = '-v, --version') {}

  /**
   * 添加一个全局用例，不适用于子命令.
   *
   */
  example(example: CommandExample) {}

  /**
   * 输出帮助信息
   */
  outputHelp() {}

  /**
   * 输出版本号
   */
  outputVersion() {}
  
  /**
   * (私有) 只在 parse 中被调用，用于将解析到的信息保存
   */
  private setParsedInfo() {}
  
  /**
   * 如果本次执行是输出帮助信息或版本信息的话会调用
   * 将匹配的子命令信息设置为 undefined
   */
  unsetMatchedCommand() {
    this.matchedCommand = undefined
    this.matchedCommandName = undefined
  }
  
  /**
   *  解析执行命令
   */
  parse() {}
  
  /**
   *  (私有)借助第三方库 mri，解析选项
   */
  private mri() {}
  
  /**
   *  在 parse 函数中调用，执行匹配的子命令
   */
  runMatchedCommand() {}
}
```

### GlobalCommand

`GlobalCommand` 是 `Command` 的子类，继承了 `Command` 的所有属性和方法：

```typescript
class GlobalCommand extends Command {
  constructor(cli: CAC) {
    super('@@global@@', '', {}, cli)
  }
}
```

`GlobalCommand` 在 `CAC` 中起到一个比较关键的最用，从名字中可以看出来这是一个全局的命令，也是一种 OOP 面向对象思想的体现，这个对象的主要作用是当用户没有建立子命令时，`CAC` 需要通过这个 `GlobalCommand` 的实例来调用 `Command` 的方法。

在 `CAC` 类的构造器中，默认会 new 一个 `GlobalCommand` 实例，而由于 `Command` 和 `CAC` 是互相关联的，在 `Command` 中对 `CAC` 也有依赖，所以在构建时会把自己传进去：

```typescript
  constructor(name = '') {
    super()
    this.name = name
    this.commands = []
    this.rawArgs = []
    this.args = []
    this.options = {}
    this.globalCommand = new GlobalCommand(this)  // 创建全局 Command，将自己传进去
    this.globalCommand.usage('<command> [options]')
  }
```

有了 `globalCommand` 实例，用户就可以直接使用 `CAC` 而不需要创建子命令，例如我们可以对 basic-usage.js 作出一点修改：

```javascript
require('ts-node/register')
const cli = require('../src/index').cac()

cli.option('--type [type]', 'Choose a project type')

const parsed = cli.parse()

function logType(){
   console.log(`类型:${parsed.options.type}` || '未指定 type 参数')
}

logType()
```

在 `option()` 函数中，`CAC` 通过 `globalCommand` 实例生成了一个选项：

```typescript
  // CAC.ts
option(rawName: string, description: string, config?: OptionConfig) {
    // 调用 Command 的 option 方法为当前实例生成选项
    this.globalCommand.option(rawName, description, config)
    return this
 }
```

随后，经过 `parse()` 函数的解析，尽管没有指定可用于运行的子命令，但还是会将解析过的选项信息返回给调用者。

```code
# 调用命令
node basic-usage.js --type test-case
```

所以可以看到，最终就像一个 Linux 命令一样，同样包含了 `可执行文件` & `选项` & `参数` 这几个要素，敲下回车，就可以得到结果。

```code
类型:test-case
进程已结束,退出代码0
```

### 链式调用的实现和子命令

`CAC` 支持链式调用，例如通过链式调用添加多个选项：

```typescript
require('ts-node/register')
const cli = require('../src/index').cac()

cli.option('--type [type]', 'Choose a project type')
  .option('--dir [dir]', 'Choose a project dir')

const parsed = cli.parse()

function logType(){
  console.log(`类型:${parsed.options.type}` || '未指定 type 参数')
  console.log(`目录:${parsed.options.dir}` || '未指定 dir 参数')
}

logType()
```

```code
# 调用命令
node basic-usage.js --type test-case --dir /opt
```

在 JS 的世界中，实现链式调用的方法都是类似的，就是每次调用函数之后，都返回一个 this，指向实例本身，于是可以通过返回的 this 引用继续调用下一个函数，在 `CAC` 中，像 `option`、`help`、`version` 等基本操作都是支持链式调用的：

```typescript
  help(callback?: HelpCallback) {
    this.globalCommand.option('-h, --help', 'Display this message')
    this.globalCommand.helpCallback = callback
    this.showHelpOnExit = true
    return this
  }
 
  version(version: string, customFlags = '-v, --version') {
    this.globalCommand.version(version, customFlags)
    this.showVersionOnExit = true
    return this
  }
  
 option(rawName: string, description: string, config?: OptionConfig) {
    this.globalCommand.option(rawName, description, config)
    return this
  }
  
```

（更新中 ... ）









