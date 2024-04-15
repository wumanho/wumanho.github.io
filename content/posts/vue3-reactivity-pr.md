---
weight: 2
title: "分析 Vue3 Reactivity  的一个重构 PR"
summary: "baseHandler.ts 为什么会被重构，结果又是如何呢"
date: 2024-04-15T13:52:24+08:00
draft: false
categories: ["前端"]
tags: ["Vue3","面向对象","类","Reactivity","TypeScript"]
---

## 前言

大概半年前，[vuejs/core]([vuejs/core: 🖖 Vue.js is a progressive, incrementally-adoptable JavaScript framework for building UI on the web. (github.com)](https://github.com/vuejs/core)) 有一个比较有意思的 refactor PR [#8586](https://github.com/vuejs/core/pull/8586)，标题为 "encapsulate reactive handlers in class"，即《通过 class 类封装 reactive handlers》。

这是一次不影响功能的重构行为，但也算得上是大刀阔斧，毕竟 `baseHandler` 是响应式模块中的核心代码。

贡献者并不是 core team 成员，但最终还是被合到了主分支，这就非常有意思了。

所以这篇博客我们尝试简单分析一下这次重构。

&nbsp;

## 动机

`baseHandler.ts` 模块的主要责任是为响应式对象（reactive）创建 `getter` 和 `setter` 的。

在重构之前的代码中，所有 `getter` 均通过同一个函数 `createGetter` 创建，区别是传入的参数不同：

```typescript
const get = /*#__PURE__*/ createGetter()
const shallowGet = /*#__PURE__*/ createGetter(false, true)
const readonlyGet = /*#__PURE__*/ createGetter(true)
const shallowReadonlyGet = /*#__PURE__*/ createGetter(true, true)
```

同样地，setters 也是通过类似的方法创建：

```typescript
const set = /*#__PURE__*/ createSetter()
const shallowSet = /*#__PURE__*/ createSetter(true)
```

Contributor 认为，这部分代码的可读性和可维护性相对较低。

许多不一样的逻辑被放置在同一个函数中，根据传入参数的不同，创建出不同的函数返回值。

而且， `getter` 和 `setter` 也是类本身就支持的行为，所以他觉得，如果使用 class 将这些代码封装起来，可以使这个模块代码的可读性和可维护性提升。

&nbsp;

## 代码实现

代码部分，上面提到的 `createGetter` 和 `createSetter` 两个函数被移除，实现了一个 `BaseReactiveHandle` 类，这个类实现了 `ProxyHandler` 接口。

需要注意的是，这是一次**不影响功能的重构行为**，所以功能性代码跟重构前是一样的，没有改动，只是代码组织方式进行了改变。

`BaseReactiveHandle` 仅包含一个基础 `getter`，以此作为所有其他 handler 的父类：

```typescript
class BaseReactiveHandler implements ProxyHandler<Target> {
  constructor(
    protected readonly _isReadonly = false,
    protected readonly _shallow = false
  ) {}

  get(target: Target, key: string | symbol, receiver: object) {
    if (key === ReactiveFlags.IS_REACTIVE) {
      return !this._isReadonly
    } else if (key === ReactiveFlags.IS_READONLY) {
      return this._isReadonly
    } else if (key === ReactiveFlags.IS_SHALLOW) {
      return this._shallow
    } else if (
      key === ReactiveFlags.RAW &&
      receiver ===
        (this._isReadonly
          ? this._shallow
            ? shallowReadonlyMap
            : readonlyMap
          : this._shallow
          ? shallowReactiveMap
          : reactiveMap
        ).get(target)
    ) {
      return target
    }

    const targetIsArray = isArray(target)

    if (!this._isReadonly) {
      if (targetIsArray && hasOwn(arrayInstrumentations, key)) {
        return Reflect.get(arrayInstrumentations, key, receiver)
      }
      if (key === 'hasOwnProperty') {
        return hasOwnProperty
      }
    }

    const res = Reflect.get(target, key, receiver)

    if (isSymbol(key) ? builtInSymbols.has(key) : isNonTrackableKeys(key)) {
      return res
    }

    if (!this._isReadonly) {
      track(target, TrackOpTypes.GET, key)
    }

    if (this._shallow) {
      return res
    }

    if (isRef(res)) {
      // ref unwrapping - skip unwrap for Array + integer key.
      return targetIsArray && isIntegerKey(key) ? res : res.value
    }

    if (isObject(res)) {
      // Convert returned value into a proxy as well. we do the isObject check
      // here to avoid invalid value warning. Also need to lazy access readonly
      // and reactive here to avoid circular dependency.
      return this._isReadonly ? readonly(res) : reactive(res)
    }

    return res
  }
}
```

同时，`BaseReactiveHandle` 因为只有一个 `getter`，所以他也是只读的，所以 `ReadonlyReactiveHandler` 的实现就非常简单，只需继承前者，并在用户操作数据的时候，给予适当的提示即可：

```typescript
class ReadonlyReactiveHandler extends BaseReactiveHandler {
  constructor(shallow = false) {
    super(true, shallow)
  }

  set(target: object, key: string | symbol) {
    if (__DEV__) {
      warn(
        `Set operation on key "${String(key)}" failed: target is readonly.`,
        target
      )
    }
    return true
  }

  deleteProperty(target: object, key: string | symbol) {
    if (__DEV__) {
      warn(
        `Delete operation on key "${String(key)}" failed: target is readonly.`,
        target
      )
    }
    return true
  }
}
```

最后，响应式对象没有 `setter` 是不行的，所以只需要再创建一个 `MutableReactiveHandler` 继承 `BaseReactiveHandle`，将 `setter` 逻辑完善一下：

```typescript
class MutableReactiveHandler extends BaseReactiveHandler {
  constructor(shallow = false) {
    super(false, shallow)
  }

  set(
    target: object,
    key: string | symbol,
    value: unknown,
    receiver: object
  ): boolean {}

  deleteProperty(target: object, key: string | symbol): boolean {
	// ...
  }

  has(target: object, key: string | symbol): boolean {
   // ...
  }

  ownKeys(target: object): (string | symbol)[] {
   // ...
  }
}
```

导出时，将对应的类进行实例化，这次的重构就大功告成了：

```typescript
export const mutableHandlers: ProxyHandler<object> =
  /*#__PURE__*/ new MutableReactiveHandler()

export const readonlyHandlers: ProxyHandler<object> =
  /*#__PURE__*/ new ReadonlyReactiveHandler()

export const shallowReactiveHandlers = /*#__PURE__*/ new MutableReactiveHandler(
  true
)
export const shallowReadonlyHandlers =
  /*#__PURE__*/ new ReadonlyReactiveHandler(true)
```

&nbsp;

## 改进和影响

首先相信大家都认同，这次重构对于代码的可读性无疑是有显著提升的，所以 core team 成员才会对这次提交有更进一步的兴趣。

但是代码组织结构改变之后，除了可读性外，还产生了哪些影响呢？

### 代码体积

实际上 Contributor 在进行重构时，就已经考虑到了代码体积问题，所以这次一共实现了三个类，分别是 `BaseReactiveHandle`、`ReadonlyReactiveHandler` 以及 `MutableReactiveHandler`。

分成三个类的好处是可以方便地进行摇树操作，假如用户在代码中没有用到 `MutableReactiveHandler`，那这个类就会被 tree shaking 摇掉。

最终，代码体积仅比重构前增加了 0.05 KB 大小，在进行 gzip 压缩之后，几乎可以忽略不计。

重构后：

    reactivity.global.prod.js min:12.01kb / gzip:4.42kb / brotli:4.08kb
    runtime-dom.global.prod.js min:83.82kb / gzip:31.80kb / brotli:28.71kb
    vue.global.prod.js min:128.50kb / gzip:48.14kb / brotli:43.18kb
    vue.runtime.global.prod.js min:83.82kb / gzip:31.80kb / brotli:28.73kb

重构前：

    reactivity.global.prod.js min:11.96kb / gzip:4.44kb / brotli:4.09kb
    runtime-dom.global.prod.js min:83.76kb / gzip:31.84kb / brotli:28.72kb
    vue.global.prod.js min:128.45kb / gzip:48.17kb / brotli:43.27kb
    vue.runtime.global.prod.js min:83.77kb / gzip:31.83kb / brotli:28.77kb

### 性能

经过 core team 成员和 Contributor 进行 benchmark 测试，最终结果如下

重构前：

    Use Vue3: https://cdnjs.cloudflare.com/ajax/libs/vue/3.3.4/vue.global.js
    Benchmarks: reactiveObject
    Available: ref,computed,watch,watchEffect,mix,reactiveObject,reactiveMap,reactiveArray
    vue.global.js:10552 You are running a development build of Vue.
    Make sure to use the production build (*.prod.js) when deploying for production.
    Benchmark: reactiveObject
    -create reactive obj x 183,736 ops/sec ±164.46% (5 runs sampled)
    write reactive obj property x 4,866,484 ops/sec ±0.42% (68 runs sampled)
    write reactive obj, don't read computed (never invoked) x 4,872,878 ops/sec ±0.26% (69 runs sampled)
    write reactive obj, don't read computed (invoked) x 3,021,506 ops/sec ±0.25% (66 runs sampled)
    write reactive obj, read computed x 2,183,469 ops/sec ±0.18% (67 runs sampled)
    write reactive obj, don't read 1000 computeds (never invoked) x 4,812,272 ops/sec ±0.20% (67 runs sampled)
    write reactive obj, don't read 1000 computeds (invoked) x 42,809 ops/sec ±0.24% (69 runs sampled)
    write reactive obj, read 1000 computeds x 6,859 ops/sec ±0.34% (66 runs sampled)
    1000 reactive objs, 1 computed x 11,690 ops/sec ±0.58% (63 runs sampled)

&#x20;重构后：

    Use Vue3: ./vue.global.js
    Benchmarks: reactiveObject
    Available: ref,computed,watch,watchEffect,mix,reactiveObject,reactiveMap,reactiveArray
    vue.global.js:10552 You are running a development build of Vue.
    Make sure to use the production build (*.prod.js) when deploying for production.
    Benchmark: reactiveObject
    +create reactive obj x 574,722 ops/sec ±51.86% (13 runs sampled)
    write reactive obj property x 4,636,881 ops/sec ±0.25% (69 runs sampled)
    write reactive obj, don't read computed (never invoked) x 4,620,616 ops/sec ±0.24% (69 runs sampled)
    write reactive obj, don't read computed (invoked) x 2,687,439 ops/sec ±3.70% (61 runs sampled)
    write reactive obj, read computed x 2,051,574 ops/sec ±1.19% (68 runs sampled)
    write reactive obj, don't read 1000 computeds (never invoked) x 4,495,859 ops/sec ±0.34% (67 runs sampled)
    write reactive obj, don't read 1000 computeds (invoked) x 41,264 ops/sec ±2.62% (66 runs sampled)
    write reactive obj, read 1000 computeds x 6,645 ops/sec ±2.37% (38 runs sampled)
    1000 reactive objs, 1 computed x 11,758 ops/sec ±0.32% (68 runs sampled)

*   创建响应式对象，**性能有显著提升（200% - 700%）**

*   对响应式对象进行读写，**性能有些微下降（1% - 5%）**

对于这个结果，vue 团队应该是比较认可的，所以测试没问题之后，就选择合并了。

&nbsp;

## 总结

对于重构后有这么巨大的性能提升，感觉有点好奇，所以就简单推测一下。

我猜大概是因为在重构前，每当创建一个响应式对象时，都需要调用两个函数 `createGetter` 和 `createSetter`。

所以一旦到了创建多个响应式对象的场景，这两个函数被无意义地连续执行了多次。

而重构后，创建响应式对象需要做的就仅仅是执行一次 `new` 操作，不再需要执行 create 函数调用，同时也达到了一摸一样的效果。

但是，对于造成响应式对象读写性能下降这一点，只能说庆幸只有不到 5%，一旦超过了 5%，对于总体性能来说其实也是一个较大的影响了，对于是否合并也是需要取舍的。

（完）

