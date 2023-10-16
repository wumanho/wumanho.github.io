---
weight: 2
title: "tRPC 配合 Vue 构建端到端类型安全的全栈方案简介"
summary: "用 tRPC 和 Vue 搭建一个 monorepo 全栈项目"
date: 2023-10-07T14:55:23+08:00
draft: false
categories: ["技术"]
tags: ["TypeScript","RPC","远程过程调用","tRPC","全栈","Vue"]
---

## tRPC 简介

[tRPC](https://github.com/trpc/trpc) 是一个基于 TypeScript 的全栈框架，相对其他全栈框架例如 next 等不算特别流行。

但随着 TypeScript 的应用越来越广泛， `tRPC` 的优势就越来越明显。



### 什么是 RPC

要了解 `tRPC`，就需要先理解什么是 RPC，RPC（Remote Procedure Call）即远程过程调用，通常被应用于后端分布式系统中。

RPC 与 HTTP 一样，都是一种服务端/客户端通讯的手段。

在 HTTP 中，我们通过请求一个 URL 来获取结果，而在 RPC 中，我们可以直接像调用本地程序一样直接调用一个函数，来获取这个函数的结果，这样的好处有很多，例如省去了数据序列化和反序列化的步骤，还有提高了安全性。

当然，RPC 调用也是基于网络请求的，只是像 tRPC 这种框架会对调用过程进行封装，对用户来说是无感知的。



### tRPC 的关键优势

如上文所说，RPC 可以使客户端直接调用服务端的函数，基于这一点，前后端不仅可以共享一套类型，实现**端到端的类型安全**，还可以利用**更全面的智能类型推断**来驱动生产力。

效果可以参考 tRPC 官网视频：

![官网视频](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/tRPC/trpcdemo.gif)

&nbsp;

## 初始化 monorepo 项目

本次实验将会使用 pnpm 管理 monorepo ，先初始化项目：

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

```bash
├── node_modules
├── packages
│   ├── client
│   ├── server
├── pnpm-lock.yaml
├── .gitignore
├── .prettierrc
├── pnpm-workspace.yaml
├── package.json
```

安装 trpc 依赖：

```bash
pnpm install @trpc/client @trpc/server -w
```

&nbsp;

## 搭建服务端

进入 server 目录

```bash
cd ./packages/server
```

初始化 npm 项目，安装必要依赖：

```bash
npm init --name="@trpc-vue/server"
pnpm i filter zod -r --filter @trpc-vue/server
```



### 初始化 tRPC 服务端

```typescript
// file: packages/server/trpc.ts

import { initTRPC } from '@trpc/server'
const t = initTRPC.create()
export const router = t.router
export const publicProcedure = t.procedure
```



### 数据库操作模拟

```typescript
// file: packages/server/db.ts
export type User = { id: string; name: string }

const users: User[] = []
export const db = {
  user: {
    findMany: async () => users,
    findById: async (id: string) => users.find((user) => user.id === id),
    create: async (data: { name: string }) => {
      const user = { id: String(users.length + 1), ...data }
      users.push(user)
      console.log(users, 'usrs')
      return user
    }
  }
}

```



### 定义接口路由 & 启动服务

我们先对以下代码会用到的两个关于 tRPC 的关键概念做个简单介绍：

* publicProcedure：`Procedure(过程)` 是 RPC 中的一个概念，是指在远程服务器上执行的特定功能或操作，可以理解为一个 API 端点，在 tRPC 中，一个过程可以有三种定义：`query`指一次查询操作，`mutation`指一次增删改操作，`subscription`指一次订阅，订阅模式本篇博客不作介绍。
* Validation：校验，tRPC 支持对输入参数进行校验，以下代码使用`zod`库进行校验

```typescript
// file: packages/server/index.ts

import { router, publicProcedure } from './trpc'
import { createHTTPServer } from '@trpc/server/adapters/standalone'
import { db } from './db'
import { z } from 'zod'

/**
 *  接口路由定义
 */
const appRouter = router({
  userList: publicProcedure.query(async () => {
    const users = await db.user.findMany()
    return users
  }),
  userById: publicProcedure.input(z.string()).query(async (opts) => {
    const { input } = opts
    const user = await db.user.findById(input)
    return user
  }),
  userCreate: publicProcedure.input(z.object({ name: z.string() })).mutation(async (opts) => {
    const { input } = opts
    const user = await db.user.create(input)
    return user
  })
})

// 创建服务
const server = createHTTPServer({
  router: appRouter
})

// 注意这里导出的是类型，用于自动类型推导的
export type AppRouter = typeof appRouter

// 监听端口
server.listen(3721)

```

&nbsp;

## 搭建客户端

进入 client 目录

```bash
cd ./packages/client
# 创建 vite 项目，选 typescript
pnpm create vite
```

安装一下依赖：

```bash
pnpm i @trpc-vue/server -r --filter @trpc-vue/client
```

别的就可以不用动了。



### 配置代理

转发配置一下，不然也会有跨域问题

```typescript
// file: packages/client/vite.config.ts

import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  server: {
    proxy: {
      '/trpc': {
        target: 'http://localhost:3721',
        changeOrigin: true,
        secure: false,
        rewrite: (path) => path.replace(/^\/trpc/, '')
      }
    }
  }
})

```



### 封装 useTRPC hook

方便起见，将客户端连接也封装在 hook 里面

```typescript
// file: packages/client/src/hooks/useTRPC.ts

import { httpLink, createTRPCProxyClient } from '@trpc/client'
import { AnyRouter } from '@trpc/server'

/**
 *  url：服务端地址
 *  headers：允许添加请求头，如携带令牌
 */
export function useTRPC<Router extends AnyRouter>(
  url: string,
  headers?: Parameters<typeof httpLink>[0]['headers'],
  transformer?: Parameters<typeof createTRPCProxyClient>[0]['transformer']
) {
  const httpLinkConfig = httpLink({ url: url, headers })
	// 创建 trpc 客户端
  const trpc = createTRPCProxyClient<Router>({
    transformer,
    links: [httpLinkConfig]
  })
  return { trpc }
}

```



### 页面调用 trpc 服务端

```vue
<script setup lang="ts">
// file: packages/client/src/App.vue
  
import { ref } from 'vue'
import type { User } from '@trpc-vue/server/types'
import { useTRPC } from './hooks/useTRPC'
  
// 这里导入服务端的路由定义
import { AppRouter } from '@trpc-vue/server'
const { trpc } = useTRPC<AppRouter>('/trpc')

// 新增用户
const addUser = async () => {
  await trpc.userCreate.mutate({
    name: new Date().toISOString() + '用户'
  })
  await initPage()
}

const users = ref<User[]>([])

// 查询用户
async function initPage() {
  users.value = await trpc.userList.query()
}
initPage()
</script>

<template>
  <ul>
    <li v-for="item in users" :key="item.id">{{ item.name }}</li>
  </ul>
  <button @click="addUser">添加用户</button>
</template>

<style lang="scss" scoped></style>
```

&nbsp;

## 测试

启动服务端：

```bash
cd ./packages/server/

# 用 ts-node 执行 index.ts
ts-node ./index.ts
```

启动客户端：

```bash
cd ./packages/client/

npm run dev
```

新增和查询：

![新增和查询](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/tRPC/trpc-add-query.gif)

新增和查询调用成功，测试完成。

&nbsp;

## 总结

[完整代码仓库](https://github.com/wumanho/vue-trpc-demo-repo)

虽然本次实验是比较简单的，但已经可以充分展现出 tRPC 强大的地方。

实际上，还有更强大的类型推导这一点在博客中无法展示，tRPC 是一个完全可以在生产环境应用的成熟的库，推荐各位有兴趣的读者亲自动手试一试。

&nbsp;

（完）
