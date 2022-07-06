---
weight: 2
title: "【CAC】源码阅读笔记 01 - 工程化"
date: 2022-07-04T20:17:08+08:00
draft: false
categories: ["前端"]
tags: ["CAC","TypeScript","Node.js"]
---

## 前言

本系列博客主要用于记录和分享我自己学习 Node.js 库 [CAC](https://github.com/cacjs/cac) 的笔记，CAC 是一个开源工具，用于帮助用户构建基于命令行的应用程序，基于 Typescript 和 OOP 思想是我认为值得学习的地方。

除了源码本身之外，工程化所使用的技术和配置，也会有详细或者简单介绍。



## 目录结构

```shell
.
├── LICENSE
├── README.md
├── circle.yml								#circle.yml: CircleCI 配置文件
├── examples   						      	#examples: 演示代码目录
│   ├── basic-usage.js
│   ├── command-examples.js
│   ├── command-options.js
│   ├── dot-nested-options.js
│   ├── help.js
│   ├── ignore-default-value.js
│   ├── negated-option.js
│   ├── sub-command.js
│   └── variadic-arguments.js
├── .editorconfig							#.editorconfig: EditorConfig 配置文件
├── .gitattributes
├── .gitignore
├── .prettierrc	
├── index-compat.js							#index-compat.js: 处理兼容性问题
├── jest.config.js							#jest.config.js: jest 配置文件
├── mod.js									#mod.js: 兼容 deno
├── mod.ts									#mod.js: 兼容 deno
├── mod_test.ts
├── package.json							#package.json: package.json
├── rollup.config.js						#rollup.config.js: rollup 打包配置
├── scripts									#scripts: js 脚本目录
│   └── build-deno.ts
├── src										#src: 源码目录
│   ├── CAC.ts
│   ├── Command.ts
│   ├── Option.ts
│   ├── __test__							#__test__: 测试目录
│   │   ├── __snapshots__					#__snapshots__:测试快照目录
│   │   │   └── index.test.ts.snap
│   │   └── index.test.ts
│   ├── deno.ts
│   ├── index.ts
│   ├── node.ts
│   └── utils.ts
├── tsconfig.json							#tsconfig.json: ts 配置
└── yarn.lock
```

&nbsp;

## scripts 目录详解

在阅读源码之前，不妨先看一看 cac 的工程化，这个项目的工程化运用的技术算是比较新的，所以估计有不少值得学习的地方，先从 script 目录入手。

scripts 目录用于存放 js 脚本，在本项目中仅有一个，用于兼容和构建 deno 工程，构建后会将文件输出在`deno `目录，`deno/index.ts` 会被 `mod.ts`指定为 deno 工程入口文件：

```typescript
/* mod.ts */
export * from './deno/index.ts'
```

`build-deno.ts` 脚本主要做了两件事，读取`src`目录下的源码文件，借助`bable`的 API 完成对于 deno 工程的转换：

```typescript {hl_lines=[15]}
/* scripts/build-deno.ts */

// 入口函数
async function main() {
    // 读取 src 目录下除了测试代码之外的所有源码文件
  const files = await globby(['**/*.ts', '!**/__test__/**'], { cwd: 'src' })
  await Promise.all(
      // 遍历读取
    files.map(async (file) => {
      if (file === 'node.ts') return
        // 读取代码内容，格式化为 urf-8
      const content = await fs.readFile(path.join('src', file), 'utf8')
      // 借助插件，完成代码转换
      const transformed = await transformAsync(content, {
          // 注意，node2deno 是自定义插件
        plugins: [tsSyntax, node2deno],
      })
      // 输出到根下的 deno 目录
      await fs.outputFile(path.join('deno', file), transformed.code, 'utf8')
    })
  )
}

main().catch((error) => {
  console.error(error)
  process.exitCode = 1
})
```

值得注意的是，这里使用了一个自定义 bable 插件来完成代码转换工作：

```typescript
function node2deno(options: { types: typeof Types }): PluginObj {
  return {
    name: 'node2deno',

    visitor: {
        // ImportDeclaration 函数主要用于处理文件引用
      ImportDeclaration(path) {
        const source = path.node.source
        if (source.value.startsWith('.')) {
          if (source.value.endsWith('/node')) {
            source.value = source.value.replace('node', 'deno')
          }
          source.value += '.ts'
        } else if (source.value === 'events') {
          source.value = `https://deno.land/std@0.114.0/node/events.ts`
        } else if (source.value === 'mri') {
          source.value = `https://cdn.skypack.dev/mri`
        }
      },
    },
  }
}
```

来看一看这个插件做了什么，`ImportDeclaration` 函数是 bable 提供用于处理 import 语句的，`path.node.source.value` 这个变量所获取到的是每个 import 语句的值，例如：

```typescript
import { EventEmitter } from 'events' // path.node.source.value = ‘events’
import mri from 'mri' // path.node.source.value = ‘mri’
import CAC from './CAC' // path.node.source.value = ‘./CAC’
```

理解了这一点之后，就很容易明白这个函数在处理什么

* 当文件中引用了 `events` 的时候，改变引用路径为 `https://deno.land/std@0.114.0/node/events.ts`
* 当文件中引用了 `mri` 的时候，改变引用路径为 `https://cdn.skypack.dev/mri`
* 当文件中引用了其他文件的时候，后缀加上 .ts
* 当文件中引用了 node.ts 的时候，将 node 改为 deno

&nbsp;

## circle.yml 配置简介

`circle.yml` 用于配置 `CircleCI` 与 github 继承的自动化测试和构建任务。

```yaml
version: 2
jobs:
  build:
    docker:
      - image: circleci/node:12
    branches:
      ignore:
        - gh-pages # list of branches to ignore
        - /release\/.*/ # or ignore regexes
    steps:
      - checkout
      - restore_cache:
          key: dependency-cache-{{ checksum "yarn.lock" }}
      - run:
          name: install dependences
          command: yarn
      - save_cache:
          key: dependency-cache-{{ checksum "yarn.lock" }}
          paths:
            - ./node_modules
      - run:
          name: test
          command: yarn test:cov
      - run:
          name: upload coverage
          command: bash <(curl -s https://codecov.io/bash)
      - run:
          name: Release
          command: yarn semantic-release

```

其实这个配置文件写得已经比较清晰了，可以仅从语义上判断整个构建过程。

### docker

指定用户构建构建的 docker 环境

### branches - ignore

配置需要忽略的分支或者目录

### steps-checkout

checkout 指令用于签出代码到配置好的路径

### steps-restore_cache

通过 key 按顺序恢复对应的缓存，checksum 会对给定的文件名生成哈希，用于更新缓存。

### run-install dependences

安装依赖

### save_cache

保存缓存到对应的 key

### run-test

执行命令 `jest --coverage`，--coverage 参数会在输出中生成测试覆盖率报告。

### run-upload coverage

Linux 中 `<` 的意思**将后面文件作为前面命令的输入**，所以这一步在做的是从 codecov.io 获取脚本，上传测试报告

### run-Release

发布版本，具体可以参考 `semantic-release` 这个库

&nbsp;

## rollup.config.js 打包配置简介

```javascript
import nodeResolvePlugin from '@rollup/plugin-node-resolve'
import esbuildPlugin from 'rollup-plugin-esbuild'
import dtsPlugin from 'rollup-plugin-dts'

// 辅助函数
function createConfig({ dts, esm } = {}) {
    // file 是输出结果
  let file = 'dist/index.js'
   // 根据参数判断输出结果的后缀
  if (dts) {
    file = file.replace('.js', '.d.ts')
  }
  if (esm) {
    file = file.replace('.js', '.mjs')
  }
  return {
      // 入口文件
    input: 'src/index.ts',
      // 输出文件配置
    output: {
        // 指定输出格式
      format: dts || esm ? 'esm' : 'cjs',
      file,
      exports: 'named',
    },
    plugins: [
      nodeResolvePlugin({
        mainFields: dts ? ['types', 'typings'] : ['module', 'main'],
        extensions: dts ? ['.d.ts', '.ts'] : ['.js', '.json', '.mjs'],
        customResolveOptions: {
          moduleDirectories: dts
            ? ['node_modules/@types', 'node_modules']
            : ['node_modules'],
        },
      }),
      !dts && require('@rollup/plugin-commonjs')(),
      !dts &&
        esbuildPlugin({
          target: 'es2017',
        }),
      dts && dtsPlugin(),
    ].filter(Boolean),
  }
}

// 输出配置
export default [
  createConfig(),
  createConfig({ dts: true }),
  createConfig({ esm: true }),
]
```

打包配置也是非常简单，通过一个辅助函数`createConfig`，输出三种不同的结果

* dts（ts 声明文件）
* cjs（cjs 格式）
* mjs（esm 格式）

plugins 中有个比较不常见的写法，借助`Array.filter`，过滤出需要的插件。

```typescript
  plugins: [
      nodeResolvePlugin({
        mainFields: dts ? ['types', 'typings'] : ['module', 'main'],
        extensions: dts ? ['.d.ts', '.ts'] : ['.js', '.json', '.mjs'],
        customResolveOptions: {
          moduleDirectories: dts
            ? ['node_modules/@types', 'node_modules']
            : ['node_modules'],
        },
      }),
      !dts && require('@rollup/plugin-commonjs')(),
      !dts &&
        esbuildPlugin({
          target: 'es2017',
        }),
      dts && dtsPlugin(),
    ].filter(Boolean), // 返回为 true 的结果
```

&nbsp;

## jest 配置

```javascript
module.exports = {
    // 用于测试的环境，这里是 node 环境，如果需要测试 web 环境，可以配置为 jsdom
  testEnvironment: 'node',
    // 如果需要测试的不是原生 js 代码，则需要指定解析器
  transform: {
    '^.+\\.tsx?$': 'ts-jest'
  },
    //匹配测试文件
  testRegex: '(/__test__/.*|(\\.|/)(test|spec))\\.tsx?$',
    //配置忽略的目录
  testPathIgnorePatterns: ['/node_modules/', '/dist/', '/types/'],
    //指定文件类型
  moduleFileExtensions: ['ts', 'tsx', 'js', 'jsx', 'json', 'node']
}

```

### moduleFileExtensions

jest 官方建议将最常用的格式配置到最左侧，本项目几乎是纯 ts 编码的，所以将 ts 配置在第一位，是一个可以学习的最佳实践

&nbsp;

## index-compat.js

```javascript
const { cac, CAC, Command } = require('./dist/index')

// 兼容性处理
module.exports = cac

Object.assign(module.exports, {
  default: cac,
  cac,
  CAC,
  Command,
})
```

注意，`index-conpat.js`这个文件被作为`package.json` 中的 `exports` 字段的导出方式之一：

```json
/** package.json **/

  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./index-compat.js"
    },
    "./package.json": "./package.json",
    "./": "./"
  },
```

exports 字段可以根据不用的导入方式返回不同的结果，所以一下这两种写法都是可以的：

```javascript
// CommonJS 写法，从 ./index-compat.js 中导出
const cac = require('cac')

// ESModule 写法，从 ./dist/index.mjs 中导出
import cac from 'cac'
```

&nbsp;

（本篇完）



