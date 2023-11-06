---
weight: 2
title: "【RPC】如何封装一个远程过程调用 - 实现迷你全栈 rpc 框架"
summary: "tRPC 源码学习，并动手实现一个包含客户端和服务端的 rpc 框架"
date: 2023-10-25T18:09:23+08:00
draft: false
categories: ["技术","推荐阅读"]
tags: ["TypeScript","RPC","远程过程调用","tRPC","全栈"]
---

## 前言

上一篇博客对全栈框架 tRPC 和 Vue 如何进行结合使用进行了一个简单的介绍，这次我们来尝试更加深入地了解一下 tRPC。

所以，本篇博客的目标是实现一个小型 tRPC，包含客户端以及服务端。

由于 tRPC 的源码复杂度比较高，所以需要注意的是，我们不会完全按照 tRPC 源码的思路去做（只是参考），而是以实现功能为目的，以最简单的方式去进行实现，包括类型代码。

主要目的是对于 rpc 远程过程调用的思想进行学习。

最终实现跟官网演示接近的类型提示效果：

![效果](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/tRPC/trpc-show.gif)

&nbsp;

## 实现思路

理清楚实现思路之前，我们先复习一下 rpc 的定义：

> 远程过程调用（Remote Procedure Call，RPC）是一种计算机通信协议。它允许运行在一台计算机上的程序调用另一台计算机上的子程序，就像调用本地程序一样，不需要程序员显式编写这个交互细节。

听起来是不是很神奇，居然可以在本地直接调用一个远端的函数？

搞不清楚就对了，这正是我们为什么要学习 tRPC 的原因。

### 服务端实现思路

rpc 虽然说是可以像本地一样调用函数，但实际上也是需要通过网络通讯的，只不过通常提供 rpc 服务的框架会将这些细节封装起来。

rpc 通讯可以基于 tcp、udp、http 协议进行，取决于框架的开发者，而本次我们的主角 tRPC 是**基于 http 协议**的。

既然是基于 http，那自然也是 nodejs 的强项，服务端的实现思路是比较简单的，总体来说就是像普通后端程序一样，起一个 http 服务，定义路由，就完成 80% 的工作了。

我们提供一个定义路由的方法，用户就跟平常写接口没什么区别，只是用户不需要再考虑路径、处理 http 请求等一系列问题了。

像常见的 req、res 等对象，对用户来说也是不可见的，用户需要实现的就是一个普通函数。

然后，我们将用户定义的路由以一个键值对映射表的方式保存起来，在监听到 http 请求之后，做统一处理：

解析请求的参数，直接通过映射表找到对应的函数去调用，获取返回值，然后返回即可。

### 客户端实现思路

客户端的实现则会比较复杂一些，倒也不是代码复杂，而是思路不好一下子理解和总结。

我们在客户端需要创建一些代理，去代理客户端的函数调用。

用户在调用一个远程的函数的时候，实际上这个函数对于客户端来说是不存在的，所以我们需要借助 TypeScript 的类型提示，让用户在进行开发的过程中，可以得到编辑器提供的代码提示。

然后在用户调用这个函数的时候，我们再创建一个代理，将用户传入的参数进行解析封装，发送 http 请求到服务端，获取返回值，然后将返回值解析，作为函数返回值返回。

文字描述比较难以表达，我们后面用代码实现的时候就会比较清晰了。

&nbsp;

## 项目搭建

本次项目依然是使用 monorepo 搭建，需要注意的是，我们这次不会太注重工程化，也不会对代码进行打包，只会项目内部引用。

先初始化项目：

```bash
npm init -y
git init
```

在项目的跟目录创建工作空间配置文件`pnpm-workspace.yaml`：

```yaml
packages:
  - "packages/*"
```

所有代码文件都会放在 packages 目录下，初始化完成后项目结构如下：

其中 `example` 是一个 Vue 项目，用于引用我们写好的框架来测试的。

```bash
├── node_modules
├── packages
│   ├── client
│   ├── example
│   ├── server
├── pnpm-lock.yaml
├── .gitignore
├── .prettierrc
├── pnpm-workspace.yaml
├── package.json
```

安装 ts

```bash
pnpm install typescript -w
```

&nbsp;

## 服务端实现

服务端实现要依赖 node，所以先安装一下依赖：

```bash
pnpm i @types/node -r --filter @rpc-in-ts/server
```

服务端一共有两个 api 需要提供给调用方：

* `createHttpServer`：用户需要通过这个 api 将服务端启动。
* `defineRouter`：用户通过这个 api 来定义路由

### 路由

在路由模块，我们只需要提供一个定义路由的方法 `defindRouter` 就可以了。

除此之外，服务端提供给客户端的类型，也要从这里推断出来，

`Router` 是一个键值对映射表，由用户作为参数传入，然后通过 `RouterConfig` 这个泛型类型返回出去。

```typescript
// 文件: /packages/server/src/router.ts
export type Router = {
  [key: string]: Function
}

type RouterConfig<T> = {
  [key in keyof T]: T[key]
}

export function defineRouter<R extends Router>(config: R): RouterConfig<typeof config> {
  const router = Object.assign(Object.create(null), config)
  return router
}
```

### 启动 http 服务

用 node 创建一个 http 服务器，然后用刚刚用户定义的路由作为参数传入。

```typescript
// 文件: /packages/server/src/http.ts
import http from 'http'
import { Server } from 'http'
import { Router } from './router'

interface HttpServerOptions {
  router: Router
}

/**
 * 创建 http 服务
 * @param opts 默认只有路由参数
 * @returns http server 实例
 */
export function createHttpServer(opts: HttpServerOptions): Server {
  const handler = createHttpHandler(opts)
  return http.createServer((req, res) => handler(req, res))
}

// 创建请求回调
function createHttpHandler(opts: HttpServerOptions) {
  return async (req: http.IncomingMessage, res: http.ServerResponse) => {
    // 解析 body
    const body = await getBody(req)
    if (!body.status) {
      res.end(JSON.stringify({ status: false }))
      return
    }
    // 获取请求方法
    const path = body.data.method
    const { router } = opts
    // 调用方法，回应请求
    if (router[path] && typeof router[path] === 'function') {
      const result = await router[path](...body.data.args)
      res.end(
        JSON.stringify({
          status: true,
          data: result
        })
      )
    }
  }
}

/**
 * 获取 http 请求体
 * @param req node 请求对象
 * @returns
 */
interface BodyResult {
  status: Boolean
  data: any
}
async function getBody(req: http.IncomingMessage): Promise<BodyResult> {
  return new Promise((resolve) => {
    let body = ''
    req.on('data', (chunk) => {
      body += chunk.toString()
    })
    req.on('end', () => {
      resolve({
        status: true,
        data: JSON.parse(body)
      })
    })
  })
}

```

### 导出 server

将所有需要导出的内容导出。

```typescript
// 文件: /packages/server/src/index.ts
export * from './http'
export * from './router'
```

至此，服务端就实现完成了。

&nbsp;

## 客户端实现

客户端需要安装 server 端作为依赖：

```bash
pnpm i @rpc-in-ts/server -r --filter @rpc-in-ts/server
```

客户端这边，我们只需要提供一个 api `createClientProxy` 即可。

用户通过 `createClientProxy` ，传入服务端的请求地址，就可以获取到我们生成的代理对象，然后直接调用请求了。

```typescript
// 文件: /packages/client/src/index.ts
import { AppRouter } from '@rpc-in-ts/server'

type ProxyCallback = (args: unknown[]) => unknown

export function createClientProxy<R extends AppRouter>(url: string): R {
  const proxy = new Proxy({} as R, {
    get(_, key) {
      return createInnerProxy(async (args) => {
        const result = await fetch(url, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({
            method: key,
            args: args
          })
        })
        const json = await result.json()
        if (json.status) {
          return json.data
        } else {
          return null
        }
      })
    }
  })
  return proxy
}

/**
 * 创建一个新的代理对象返回给调用方
 * 返回值是一个函数，调用该函数会触发代理对象的 apply 方法
 * 获取到传入的参数后，调用 callback
 * @param callback
 * @returns
 */
function createInnerProxy(callback: ProxyCallback) {
  return new Proxy(callback, {
    apply(_1, _2, args) {
      return callback(args)
    }
  })
}
```

这里面包含了两层代理，首先最外面 `createClientProxy`，我们创建了一个空对象，然后通过 `as` 将这个空对象断言为 `AppRouter`。

`AppRouter` 是服务端在定义路由时推断出来的路由对象，这就意味着用户通过这个代理，就可以访问到所有定义的接口。

![调用](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/tRPC/trpc-show-1.gif)

不过现在有个问题，我们这个代理对象中，实际上并没有 `getUserName` 这个方法，我们也没有办法获取到用户在调用方法时所传入的参数。

所以这就需要 `createInnerProxy` 函数，来再创建一个代理，这个代理是一个函数对象。

这样，用户无论调用的是哪个接口我们都可以获取到这个**接口的名称**以及**调用时传入的参数**了。

所以在回调中，我们通过 fetch 发送一个请求到我们的服务端，将需要调用的接口名以及函数参数封装在 body 里面，这样服务端在监听到请求之后，就可以知道自己需要调用哪个接口，以及参数是什么了。

至此，客户端也实现完成了。

&nbsp;

## 创建示例项目进行测试

我们在 example 文件夹里面创建一个 Vue 项目来进行测试：

```bash
cd ./packages
# 选 ts
pnpm create vite
# 安装依赖
pnpm i @rpc-in-ts/client @rpc-in-ts/server -r --filter @rpc-in-ts/example
```

然后直接在 App.vue 引入需要用到的 api：

```vue
<script setup lang="ts">
import { createClientProxy } from '@rpc-in-ts/client'
import { AppRouter } from '@rpc-in-ts/server'

const test = async () => {
  // 传入请求路径
  const trpc = createClientProxy<AppRouter>('/trpc')
  const res = await trpc.getUserName('Hello')
  console.log('res ====>',res)
}
</script>

<template>
  <button @click="test">初始化</button>
</template>
```

注意，在浏览器调用远程服务无论如何也是会有跨域问题的，所以需要配置一下 Vite 的转发：

```typescript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [vue()],
  server: {
    proxy: {
      '/trpc': {
        target: 'http://127.0.0.1:3721',
        changeOrigin: true,
        secure: false,
        rewrite: (path) => path.replace(/^\/trpc/, '')
      }
    }
  }
})

```

定义服务端路由，启动服务端：

```typescript
// 文件： /packages/server/src/index.ts
export * from './http'
export * from './router'
import { defineRouter } from './router'
import { createHttpServer } from './http'

const router = defineRouter({
  getUserId: (id: string) => {
    return id
  },
  getUserName: (name: string) => {
    return name
  }
})

export type AppRouter = typeof router

createHttpServer({
  router: router
}).listen(3721)
```

```bash
ts-node ./packages/server/src/index.ts
```

测试通过：

![测试结果](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/tRPC/trpc-show-2.gif)

&nbsp;

## 总结

[完整项目代码仓库](https://github.com/wumanho/rpc-in-ts)

这次浅尝即止的小实验，是我自己对于 rpc 的实现思路的一次总结。

虽然代码总体来说还是过于简单，很多问题都没有考虑进去，类型的推断也不够完美。

但还是从中学到了不少，tRPC 虽然不是一个非常主流的全栈框架，但对于前端到全栈的领域来说也是一次很有意义的尝试。
