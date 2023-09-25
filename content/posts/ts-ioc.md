---
weight: 2
title: "【TypeScript】利用装饰器实现对象管理及装饰器路由"
date: 2023-09-25T10:16:23+08:00
draft: false
categories: ["技术"]
tags: ["TypeScript","IOC","对象管理","控制反转","后端"]

---

## 前言

TypeScript 的装饰器是个非常强大的特性，基于装饰器可以让开发者可以更加灵活的修改类的行为。

前端时间对装饰器特性有了个简单的认识，这篇博客尝试封装一个基于 `express` 的迷你后端框架， 主要会实现基于类装饰器的**对象管理**以及成员方法装饰器的**装饰器路由**功能。

&nbsp;

## 准备工作

### 初始化项目和依赖

第一步，先来初始化项目，以及安装需要用到的依赖。

```bash
npm init -y
```

这个项目是基于 `express` 封装的，所以需要安装 express，以及非常关键的反射库 `reflect-metadata`

```bash
npm i express reflect-metadata 
npm i @types/express -D
```

安装 TypeScript

```bash
npm i @types/node typescript -D
```



### 配置 tsconfig

由于装饰器是个实验性功能，要开启装饰器特性需要将 `experimentalDecorators` 配置为 true。

其余配置作为参考即可。

```json
{
  "compilerOptions": {
    "target": "ES2017",
    "declaration": true,
    "module": "esnext",
    "moduleResolution": "node",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "allowSyntheticDefaultImports": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "noImplicitAny": false,
    "skipLibCheck": true,
    "typeRoots": [
      "node_modules/@types"
    ],
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"],
  "ts-node": {
    "esm": true,
    "experimentalSpecifierResolution": "node"
  }
}
```



### 安装 ts-node

简单起见，我们可以使用 `ts-node` 来启动项目，如果先前没有安装的话可以全局安装一下：

```bash
npm install -g ts-node
```

安装完成后，将 `ts-node` 添加到 `package.json` 作为启动配置

```json
{
  "scripts": {
    "start": "ts-node src/main.ts"
  },
}

```

安装完成后可以创建 main.ts，执行 `npm run start` 测试一下

```typescript
// main.ts
console.log('Hello')
```

```bash
$ npm run start

> demo@1.0.0 start
> ts-node src/main.ts

Hello
```

&nbsp;

## 创建 HTTP 服务

如上文所说 HTTP 服务基于 express 为基础，进行一次简单的封装。

但尽管如此，为了后续的可扩展性，比如说如果希望像 Nest.js 一样支持多种 HTTP 服务的话，就需要将 Server 进行抽象处理，去对其他 HTTP 服务的兼容。

所以，我们可以先声明一个抽象类` ServerFactory`，定义 Server 的基本结构，包括：

* 成员变量 middlewareList：保存所有中间件的数组
* 成员函数 setMiddleware：用于往 middlewareList 中添加中间件
* 成员函数 start：启动 HTTP 服务，参数是端口号

```typescript
/**
 * file: Server.ts
 * ServerFactory 抽象类，用于对底层 web 服务的封装
 */
export abstract class ServerFactory {
  protected middlewareList: Array<any> = []
  public abstract setMiddleware(middleware: any): void
  public abstract start(port: number): void
}
```

基于 `ServerFacroty`抽象类，再创建 express 的 Server 服务：

```typescript
/**
 * file: Server.ts
 * Express 类实现，继承自抽象类 ServerFactiry
 */
export class ExpressServer extends ServerFactory {
  protected middlewareList: Array<any> = []

  public setMiddleware(middleware: any) {
    this.middlewareList.push(middleware)
  }

  public start(port: number = 3000) {
    const app: express.Application = express()
    // 注册中间件
    this.middlewareList.forEach((middleware) => {
      app.use(middleware)
    })
    // 测试服务是否开启
    app.get('/', (req: Request, res: Response) => {
      res.send('Hello ')
    })
    // 启动服务
    app.listen(port, () => {
      console.log(`server is running at http://localhost:${port}`)
    })
  }
}
```

然后可以执行`npm run start`，测试一下服务是否开启成功。

&nbsp;

## 对象管理



### Container 静态对象管理容器

对象管理容器只需要维护一个 Map ，并提供 `addClass` 以及 `getClass` 两个静态方法，在初始化进行依赖收集时，将被声明要注入到容器中的类添加到容器中即可。

Map 的键是类名，值是类的实例。

```typescript
/**
 * file: Container.ts
 * IOC 容器对象
 * 初始化及注入等相关装饰器，都会使用这个静态类
 */
export default class Container {
  private static classMapper = new Map<string, any>()

  public static addClass(className: string, beanClass: any) {
    console.log(`save class:${className}`)
    this.classMapper.set(className, beanClass)
  }

  public static getClass(className: string): any {
    console.log(`get class:${className}`)
    return this.classMapper.get(className)
  }
}
```



### 对象管理的两个关键装饰器

完成对象管理需要有两个关键的装饰器，分别为`Injectable` 以及 `Autoware`。

`Injectable` 是一个类装饰器，声明在类名之上，例如：

```typescript
@Injectable
export default class Dog {
}

@Injectable
export default class Cat {
}
```

他的作用是获取当前类的属性，在程序进行初始化的时候，将它注入到 Container 中。

`Autoware` 是一个成员属性装饰器，声明在成员变量之上，例如：

```typescript
@Injectable
export default class Dog {
  
  @Autoware
  private cat: Cat
}
```

他的作用是将类的实例注入到当前的成员变量中，这样使用者就不需要再手动去对这个类进行一次初始化。

所以，只需要通过这两个装饰器，就可以完成对象管理的整个流程。



### 装饰器实现

装饰器实现本身并不复杂，但需要借助 `reflect-metadata` 反射库：

```typescript
/**
 * file: Decorator.ts
 * 实现所有装饰器
 */
import 'reflect-metadata'
import Container from './Container'

// @Autoware 用于自动注入对象
export function Autoware(target: any, propertyName: string): void {
  // 通过反射获取到成员函数的类型，即类名
  const type = Reflect.getMetadata('design:type', target, propertyName)
  // 从容器中返回 class 实例
  Object.defineProperty(target, propertyName, {
    get: () => Container.getClass(type.name)
  })
}

// @Injectable 用于初始化应用程序
export function Injectable(target: any): void {
  // 将对象实例添加到容器中
  Container.addClass(target.name, new target())
}
```



### 装饰器的调用时机

现在执行 `npm run start` 来启动程序，我们希望看到的结果是 **save class: Dog** 和 **save class: Cat** 会被打印出来，意味着这两个类被注入到容器之中了。

但可惜实际上并不会看到预期的打印结果，原因也很简单，`Injectable` 没有被调用。

我们需要意识到 `Injectable` 本身只是一个普通函数，TypeScript 的装饰器会在 .ts 文件被 import 载入的时候，将这些装饰器函数执行。

来测试一下，在 main.ts 中对被 `Injectable` 声明的类进行引入：

```typescript
// file: main.ts
import ExpressServer from './Server'

function start() {
  // import 两个实例
  import('./Modules/Cat')
  import('./Modules/Dog')
  const app = new ExpressServer()
  app.start()
}

start()

```

再执行 `npm run start`，就会看到结果被正确打印：

```bash
$ npm run start

> demo@1.0.0 start
> ts-node src/main.ts

server is running at http://localhost:3000
save class:Cat
save class:Dog
```

但是，出于使用者的角度，如果每个类都要手动 import 一遍，那未免过于麻烦，所以我们可以对整个初始化过程进行封装。



### 封装框架主类

框架主类相当于框架的入口，在主类中，我们这次的任务就是提供启动程序的方法，并实现其中的逻辑。

在启动程序的时候，我们要做以下几件事情：

* 启动 HTTP 服务
* 初始化所有 class
* 完成路由注册（文章后半部分实现）

方便起见，给这个小型程序取个名，就叫 `TIoc` 好了：

```typescript
// file: main.ts
import { ServerFactory, ExpressServer } from './Server'

/**
 * TIoc 主类
 */
export default class TIoc {
  /**
   * 创建服务器，启动项目
   * @param httpServer
   */
  public async create(httpServer: ServerFactory) {
    httpServer.start(3000)
  }
}

function start() {
  // 初始化主实例
  const app = new TIoc()
  // 传入 http 服务器
  app.create(new ExpressServer())
}

start()
```

启动 HTTP 相关逻辑封装好之后，就是自动 import ，对所有被 `Injectable` 声明的类进行初始化。

不过实际上，我们并不知道哪些文件中的哪些类是被 `Injectable` 装饰过的，Nest.js 的做法是通过 `@Module`、`@Controller` 和`@Provider` 等好几种装饰器的方式进行声明，然后为所有对象建立起一个依赖关系图，但这样实现起来就太过复杂了。

为了简单起见同时又不失规范，我们可以约定工程中所有以 `.provider.ts` 结尾的文件会被视为一个 Module，程序在初始化的时候，对这些文件进行 import 操作：

```typescript
// file: utils/index.ts

import fs from 'fs'
import path from 'path'
import { pathToFileURL } from 'url'

/**
 * 扫描工作目录下所有 provide 文件
 * @param dirPath 入口目录
 * @returns
 */
export function scanDirectory(dirPath: string) {
  const files = fs.readdirSync(dirPath)
  const results = []
  for (const file of files) {
    const filePath = path.join(dirPath, file)
    const stats = fs.statSync(filePath)

    if (stats.isDirectory() && file !== 'node_modules') {
      results.push(...scanDirectory(filePath))
    } else if (stats.isFile() && file.endsWith('.provider.ts')) {
      results.push(pathToFileURL(path.resolve(filePath)).href)
    }
  }
  return results
}
```

在主类中，执行扫描逻辑：

```typescript
// file: main.ts
import ServerFactory from './factory/ServerFactory'
import { scanDirectory } from './utils/index'
import path from 'path'

/**
 * TIoc 主类
 */
export default class TIoc {
  /**
   * 创建服务器，启动项目
   * @param httpServer
   */
  public async create(httpServer: ServerFactory) {
    await this.settleAllBeans()
    httpServer.start(3000)
  }
  /**
   * 扫描所有被 Injectable 装饰的类
   */
  private async settleAllBeans() {
    const providers = scanDirectory(path.join(process.cwd()))
    for (let provider of providers) {
      await import(provider)
    }
  }
}
```

执行一下 `npm run start`，将可以看到类被正确初始化。

&nbsp;

## 装饰器路由

在 `express` 中使用路由，需要在初始化 `express` 的时候，对路由所有要用到的路由进行声明，例如声明一个 get 接口：

```typescript
const app: express.Application = express()

app.get('/', (req: Request, res: Response) => {
  res.send('Hello ')
})
```

app 就是 `express` 实例。

 `.get` 就是接口请求方法，如果是 post 请求的话，就换成 `.post`，如此类推。

第一个参数是接口路径，第二个参数就是回调函数。

基于这几个条件，我们需要实现的效果是，通过 `@get()`、`@post()`、`@put()` 和 `@del()` 等几个**成员方法装饰器**作为接口请求方法声明，添加到类方法上。

然后装饰器中传入的参数作为请求路径，例如：`@get('/')`，而被装饰的成员方法作为回调函数本身：

```typescript
@Injectable
export default class Dog {
  @Autoware
  private cat: Cat

  @get('/dog')
  public test() {
    console.log('I am dog')
  }
}
```



### 装饰器实现

需要注意的是，如果装饰器是带参数的，那么在装饰器函数中需要返回一个方法：

```typescript
// file: router/index.ts
type CallbackMapper = {
  method: string
  cb: Function
}

// 路由 & callback 映射关系
const routerMap = new Map<string, CallbackMapper>()

// 实现请求方法装饰器逻辑
function settleRouterDecorators(path: string, method: string) {
  return (target: any, propertyKey: string) => {
    // callback 的 this 记得绑定到 target(即被装饰的成员方法所属的类) 中
    routerMap.set(path, {
      method,
      cb: target[propertyKey].bind(target)
    })
  }
}

/**
 * 带参数装饰器
 * @param path 请求路径
 */
function get(path: string) {
  return settleRouterDecorators(path, 'get')
}

function post(path: string) {
  return settleRouterDecorators(path, 'post')
}

function put(path: string) {
  return settleRouterDecorators(path, 'put')
}

function del(path: string) {
  return settleRouterDecorators(path, 'delete')
}

export { get, post, put, del, routerMap }
```

以上装饰器的作用就是几个常用的请求方法，在程序初始化的时候，这些装饰器函数会被调用，然后将所有路由条目存储到 `routerMap` 中。



### 注册路由

获取到所有需要注册的路由后，我们需要在初始化的逻辑中，注册到 `express` 中。

```typescript
// file: router/index.ts
import { Application } from 'express'

/* 省略上面的代码 ...*/

// 获取所有路由装饰器的结果并注册到 express 中
function initRouters(app: Application) {
  for (const [path, mapper] of routerMap) {
    const { method, cb } = mapper
    app[method](path, cb)
  }
}

export { get, post, put, del, routerMap, initRouters }
```

然后在 start 方法里面，添加注册路由逻辑：

```typescript
import { initRouters } from './router/index'
/**
 * file: Server.ts
 * Express 类实现，继承自抽象类 ServerFactiry
 */
export class ExpressServer extends ServerFactory {
  protected middlewareList: Array<any> = []

  public setMiddleware(middleware: any) {
    this.middlewareList.push(middleware)
  }

  public start(port: number = 3000) {
    const app: express.Application = express()
    // 注册中间件
    this.middlewareList.forEach((middleware) => {
      app.use(middleware)
    })
    // // 注册路由
    initRouters(app)
    // 启动服务
    app.listen(port, () => {
      console.log(`server is running at http://localhost:${port}`)
    })
  }
}
```

&nbsp;

## 测试框架

最后，我们对这个迷你框架进行一个简单测试，准备一个类，名为 `Dog.provider.ts`，这是一个以 `.provider.ts` 结尾的类，将会在程序初始化时被扫描到。

Dog 类被 `@Injectable` 装饰，意味着这个类的实例在初始化时将会被注入到 `Container` 容器中。

Dog 类中的成员方法 `test` 有一个路由装饰器，意味着这个方法是一个 get 请求，路径为 `/dog`。

Dog 类中的成员变量 cat 被 `@Autoware` 装饰，意味着折将会被自动注入 `Cat` 类，在 `test` 方法中我们在没有初始化 `Cat` 类的情况下，直接调用 cat 中的 `print` 方法，预期结果是正确调用。

```typescript
import { Autoware, Injectable, get } from '../Decorator'
import { Request, Response } from 'express'
import Cat from './Cat.provider'

@Injectable
export default class Dog {
  @Autoware
  private cat: Cat

  @get('/dog')
  public test(req: Request, res: Response) {
    // 直接调用 cat.print()
    this.cat.print()
    console.log('I am dog')
    res.send('FirstPage index running')
  }
}

```

然后，再准备`Cat.provider.ts`：

```typescript
import { Injectable } from '../Decorator'

@Injectable
export default class Cat {
  public print() {
    console.log('I am cat')
  }
}
```

现在我们执行 `npm run start`，控制台打印：

```bash
$ npm run start

> demo@1.0.0 start
> ts-node src/main.ts

save class:Cat
save class:Dog
server is running at http://localhost:3000
```

浏览器访问 `http://localhost:3000/dog` 请求 get 接口，成功请求页面：

![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/Tlion/req.png)



控制台追加打印：

```bash
server is running at http://localhost:3000
get class:Cat
I am cat
I am dog
```

意味着 `@Autoware` 自动注入 `Cat` 对象成功。

至此，我们预期的基本功能都已经实现。

&nbsp;

## 总结

限于篇幅，本次只是对装饰器的一次小小的应用实验，实际上基于当前这个小项目，还可以扩展出非常丰富的功能，TypeScript 装饰器以及反射的特性，为代码带来了很多魔法。

只是现阶段装饰器作为一个实验性的特性，更新会比较多，stage 2 跟 stage 3 的装饰器变化也比较大，建议只是作为学习了解，还是等到稳定了再做实际使用。

&nbsp;

（完）
