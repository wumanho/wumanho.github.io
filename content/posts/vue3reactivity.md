---
weight: 2
title: "Vue3 源码中的 Reactivity 学习和思考"
summary: "写一个 Vue3 Reactivity"
date: 2022-03-16T22:55:29+08:00
draft: false
categories: ["前端"]
tags: ["Vue3","源码","响应式"]
---

## 前言

正式开始读 Vue3 源码是从 Vue3 转正成为默认版本开始的，从单元测试入手，得益于 Evan You 良好的编码风格，发现刨除了 TS 的类型声明之后其实并没有想象中复杂。

写这篇博客主要是记录一些学习中的思考，以白话文为主，一方面是源码解读相关的博客和文章教程实在是太多了，另外还有一个更重要的原因就是 Vue Core Team 的霍春阳大佬最近出版的《Vue.js 设计与实现》写得实在是太好了，好到让人放弃了正经写博客的欲望，因为在这本书面前这些努力都显得非常多余:smile: 。

至于读源码到底有没有意义，还是只是单纯是内卷的一种表现。首先我是觉得拒绝内卷并不是拒绝进步，只要你认为对自己有价值的事，那还是非常值得去做一做的。

其次是只要在这行工作久了，就不难发现实现一个**库**（俗称轮子）跟在公司里埋头写**业务代码**是两种完全不一样的概念，大多数人在公司写代码基本上就是用框架提供的 API 实现业务功能，由于代码跟业务关联性很强，你完全可以一本道，想到什么写什么，像写脚本一样一行一行写下去就行了，反正其他事情有框架帮你兜底。

但框架就完全不一样，框架需要适应所有业务功能，所以它是抽象的，它需要开发者有一定程度的抽象能力，这种能力我个人认为长年累月写业务代码是无法锻炼出来的，前端程序员干三年都未必会写一个 Class（ Java 程序员也一样，只是 Java 要求你必须要写类型，这使得你的代码看起来人模人样的，实际上照样是在写脚本而已:expressionless:），所以除了亲自动手造一些轮子以外，阅读框架源码给每一位程序员带来的提升(主要是思想上)是不言而喻的，至少可以带给人一种看起来我也能做的自信心。

话有点多了:frowning:

&nbsp;

## Reactive 的本质

这个话题从 Vue2 讲到 Vue3 ，都快被讲烂了，但我觉得重点不在于如何实现，更重要是是理解它的本质，因为 Vue2 和 Vue3 的响应式核心思想是没有太多出入的，不同的只是实现方式。

不过我觉得如果随便抓一个前端来让他实现一个最简单的 Reactivity ，其实真不一定每个人都能做出来，所以我还是花了一些时间去总结，与其说 Reactivity 是一种技术，更贴切的说法应该是一种思想。

它的本质就是当一个被用户关注的数据发生变化时，触发一系列相对应的动作，在 Vue 中这个动作通常是更新页面，这些动作在源码中被描述为 [effect](https://github.com/vuejs/core/blob/main/packages/reactivity/src/effect.ts) ，官方喜欢用中文`副作用`来称呼，那么如果要自己实现的话，`effect`应该是长这样的：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
    <script type="module" src="./main.js"></script>
</head>
<body>
<div id="app">Hello!</div>
</body>
</html>
```

```javascript
/*=== main.js ===*/

//挂载到 window 上方便演示
window.text = ''
window.effect = effect

function effect() {
    const root = document.getElementById('app')
    root.innerText = window.text
}
```

![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/reactivity/effect.gif)



这就是 Reactivity 的雏形，当我把 text 的值改变后，手动触发了一下`effect()`，页面上的值就更新了，但显然这跟预期有很大出入，这响应式怎么还要手动触发？

于是，一个十分天才的想法诞生了，如果把所有引用了`text`这个值的`effect`看作为`text`的**依赖**收集起来，当我们改变`text`的值的时候，把这些**依赖**都执行一遍，那不就完美解决了这个问题了吗。

&nbsp;

## ref 实现

Vue3 源码中，[ref.ts](https://github.com/vuejs/core/blob/main/packages/reactivity/src/ref.ts) 的核心逻辑除去注释只有不到 300 行代码，不看具体实现（如 track 的过程等）的话，非常容易理解它在试图做些什么。

先看一下它的两个关键的单测：

```typescript
/** 官方仓库： packages/reactivity/__tests__/ref.spec.ts **/

describe('reactivity/ref', () => {
    //给 ref 传一个值，可以通过 .value 获取到
  it('should hold a value', () => {
    const a = ref(1)
    expect(a.value).toBe(1)
    a.value = 2
    expect(a.value).toBe(2)
  })
    //而且这个值会是响应式的
  it('should be reactive', () => {
    const a = ref(1)
    let dummy
    let calls = 0
    effect(() => {
      calls++
      dummy = a.value
    })
    expect(calls).toBe(1)
    expect(dummy).toBe(1)
    a.value = 2
    expect(calls).toBe(2)
    expect(dummy).toBe(2)
    // 如果是值没有改变，那么就 calls 就不变
    a.value = 2
    expect(calls).toBe(2)
  })
})
```

了解了大概的用法之后就可以开始动手实现了，先试试攻克第一个测试吧。

传给 ref 的参数 1 是一个基本的数字类型，数字类型是不可能有 .value 属性的，所以 ref 内部肯定是返回了一个对象:thinking:，我们就称他为`RefImpl`吧（好吧这个名字是直接从源码里抄的，你可以直接在源码里搜索到）

```typescript
/** ref.ts **/
class RefImpl {
    private _value: any
    
    constructor(value) {
        this._value = value
    }
    get value() {
        return this._value
    }
}
```

然后是 ref 方法：

```typescript
/** ref.ts **/
export function ref(value) {
    return new RefImpl(value)
}
```

至此，第一个单测应该是可以通过的，ref 方法将 value 参数传递给`RefImpl`的构造函数，提供一个`get`方法，将这个`RefImpl`返回出去就即可。

不难发现，之所以要封装一个`RefImpl`类，是为了可以实现一些拦截的动作，当用户使用 .value 获取值的时候，可以利用类的`get`方法拦截这个动作，来达到**收集依赖**的目的，而当用户利用`set`方法来试图改变这个值的时候，便可以**触发依赖**，将收集到的依赖执行，来达到响应式的目的。

所以接下来就可以开始动手攻克第二个测试了，试试将数据变成响应式的吧。

在 Vue3 的 Reactive 中，`effect`起到一个非常关键的作用，可以说是 Reactive 的核心，Vue3 提供`reactive`和`ref`这两个响应式的 API，无论用户使用哪个，最终**收集依赖**都是通过`effect`来完成的，`reactive`和`ref`主要是做一些拦截的动作，为`effect`提供入口。

`effect()` 接收一个函数作为参数，并封装到 `ReactiveEffect` 类中：

```typescript
/** effect.ts **/
export function effect(fn) {
    //将依赖封装到 reactiveeffect 类的run方法中
    const _effect = new ReactiveEffect(fn)
    //马上执行一下
    _effect.run()
}
```

```typescript
/** effect.ts **/

//全局容器，用于保存当前的 ReactiveEffect 实例
let activeEffect: any

class ReactiveEffect {
     private readonly _fn: any

    constructor(fn) {
        this._fn = fn
    }
    //封装的 run 方法，执行 effect 传入的回调函数
    run() {
        activeEffect = this
        this._fn()
    }
}
```

将 fn 封装到 `ReactiveEffect` 之后，会立即调用一次 run 方法，这是为了触发一下依赖收集。

「执行一下是为了触发一下依赖收集」这句话可能会让人有点摸不着头脑，其实可以用单测的用例作为例子说明：

```typescript
it('should be reactive', () => {
    //声明了一个响应式的值
    const a = ref(1)
    let dummy
    effect(() => {
      // effect 每次执行的时候，让 dymmy = a.value  
      dummy = a.value
    })
    expect(dummy).toBe(1)
  })
```

可以看到，传给 effect 的回调函数中，引用了响应式数据`a `，当这个回调被执行，a.value 会被触发，这时候必然会进入到之前在 `RefImpl` 事先定义好的 get 拦截器中，所以这就是收集依赖的好时机。

所以，现在来实现一下，在响应式对象的 get 被调用时，执行`track`函数来进行依赖收集：

```typescript {hl_lines=[6,9,12]}
/** ref.ts **/
import {track} from "./effect";

class RefImpl {
    private _value: any
    public deps
    constructor(value) {
        this._value = value
        this.deps = new Set()
    }
    get value() {
        track(this.deps)
        return this._value
    }
}
```

在构造函数中初始化一个用于保存依赖的集合`deps `，之所以是 Set 主要是为了避免重复收集。

```typescript
/** effect.ts **/
export function track(dep) {
    //不再重复收集
    if (dep.has(activeEffect)) return
    dep.add(activeEffect)
}
```

为 `track`传入响应式对象的 `dep` 集合，`track`函数将当前全局保存的 `activeEffect` 对象存到`dep`集合中。

实际上到这一步，依赖收集就已经完成了。所以接下来，只要**当响应式对象的值被改变的时候，触发收集到的所有依赖**，就这个简单的响应式流程就完成了。

所以，在响应式对象的 set 方法中拦截请求，在修改值后，调用 `trigger` 方法：

```typescript {hl_lines=[2,11,"16-21"]}
/** ref.ts **/
import {track,trigger} from "./effect";

class RefImpl {
    private _value: any
    public deps
    constructor(value) {
        this._value = value
        this.deps = new Set()
    }
    //收集依赖
    get value() {
        track(this.deps)
        return this._value
    }
    
    //触发依赖
    set value(newVal) {
       this._value = newVal
       trigger(this.deps)
    }
}
```

```typescript
/** effect.ts **/
export function trigger(dep) {
    for (const effect of dep) {
        effect.run()
    }
}
```

在 set 拦截器中，首先需要将响应式对象的 value 赋值为新的值，然后调用`trigger`方法，将收集到的`deps`全部传进去，而 `trigger` 只需要将这些 effect 全部执行一遍就可以了。

不妨在本地跑一下单测试试看：

![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/reactivity/test.gif)

没有问题。

&nbsp;

## 打包并测试 ref

本次测试的目标，是看看对响应式数据的更改能不能如期反映到页面上。

首先需要将实现的库打包，对于库代码的打包，像 Vue3 一样使用 `rollup`就可以了。

先将本次需要用到的 `ref` 和 `effect` 统一导出：

```typescript
/** src/reactivity/index.ts **/
export {effect} from "./effect"
export {ref} from "./ref"
```

然后在 `rollup` 的入口文件里面导出：

```typescript
/** src/index.ts **/
export * from "./reactivity/index"
```

安装依赖：

```code
yarn add rollup --dev
```

由于用到了 typescript，还需要安装一下 `rollup` 提供的 ts 插件：

```code
yarn add @rollup/plugin-typescript --dev
yarn add tslib --dev
```

添加`rollup.config.js`配置文件：

```javascript
import typescript from "@rollup/plugin-typescript"

export default {
    input: "./src/index.ts", //入口
    output: [{ //出口
        format: "es",
        file: "lib/esm.js"
    }],
    plugins: [typescript()]
}
```

打包：

```code
yarn build
```

最后，还原一开始的那个用例，不过注意看，这个时候`id="app"`的 div 内容是空的 ：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Demo</title>
    <script type="module" src="./main.js"></script>
</head>
<body>
<div id="app"></div>
</body>
</html>
```

在这里，利用 ref 去初始化元素的内容为”Hello Ref !“

```javascript
/** main.js **/
import * as Ref from "../../lib/esm.js"
import * as Effect from "../../lib/esm.js"

const {ref} = Ref
const {effect} = Effect

const text = ref("Hello Ref !")
window.text = text

effect(() => {
    const root = document.getElementById('app')
    root.innerText = text.value
})
```

根据 effect 的特性，需要马上执行一次回调函数，进行依赖收集，所以这个时候页面上应该显示`"Hello Ref !"`，来看看效果：

![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/reactivity/final.gif)

效果跟预期的一致。所以，大功告成！:tada:

&nbsp;

## 优化

记不记得在单测中还有一个小功能，就是 ref 的值被改变时，如果新的值跟原有的值一样，那么是不会触发`trigger`的，优化了一些性能，所以在最后，来把这个功能点也实现一下吧。

```typescript {hl_lines=["14-16"]}
  it('should be reactive', () => {
    const a = ref(1)
    let dummy
    let calls = 0
    effect(() => {
      calls++
      dummy = a.value
    })
    expect(calls).toBe(1)
    expect(dummy).toBe(1)
    a.value = 2
    expect(calls).toBe(2)
    expect(dummy).toBe(2)
    // 如果是值没有改变，那么就 calls 就不变
    a.value = 2
    expect(calls).toBe(2)
  })
```

思路很简单，每次设置新的值的时候，对比一下新旧的值是否相等，如果不相等就执行`trigger`，相等的话就返回

```typescript {hl_lines=[4,8,"22-23",26,30]}
/** ref.ts **/
class RefImpl {
    private _value: any
    private _rawValue: any
    public deps

    constructor(value) {
        this._rawValue = value
        this._value = value
        this.deps = new Set()
    }

    //收集依赖
    get value() {
        track(this.deps)
        return this._value
    }

    //触发依赖
    set value(newVal) {
        //如果数据没有变更，不触发依赖
        if (hasChanged(newVal, this._rawValue)) {
            this._rawValue = newVal
            this._value = newVal
            trigger(this.deps)
        }
    }
}

const hasChanged = (newVal, oldVal) => !Object.is(newVal, oldVal)
```

再来执行一下单测看看是否可以通过：

![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/reactivity/finaltest.gif)

顺利通过。

&nbsp;

## 总结

这次实现的`ref `只是 Reactivity 中的其中一个小部分，`ref`只能对值数据进行拦截，如果需要对一个对象进行拦截，就要实现`reactive`方法，但核心思路没有太大的区别，经常听说的 Vue3 使用了 ES6 的`Proxy`实现了响应式就是从这里来的，不过碍于篇幅，`reactive`可能会在下一篇博客中再尝试实现，敬请期待。

[查看本次演示的代码仓库](https://github.com/wumanho/vue-ref-for-blog)

&nbsp;

（完）
