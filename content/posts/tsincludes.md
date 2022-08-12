---
weight: 2
title: "【Typescript】Array.includes 函数对于常量数组的处理"
date: 2022-08-12T13:03:12+08:00
draft: false
categories: ["前端"]
tags: ["TypeScript","narrowing","Array"]
---

## 前言

`Array.includes()` 应该大家都很熟悉，在数组中匹配传入的参数，返回一个布尔值，ts 对于这个函数也提供了对应的原生类型声明。

不过最近在开发一个 CLI 库的时候遇到了一个问题，当传入的参数是一个常量数组的时候，ts 会对这个参数报错。

&nbsp;

## 复现

正常来说，`includes`函数对于普通数组的过滤当然是没问题的，例如，我在`checkOptionValue`函数中判断用户输入的参数是否支持：

```typescript
const optionTypes = ['component', 'styled' ,'lib-entry']

export async function checkOptionValue(options: Options) {
  let { type } = options
  // 判断用户输入是否支持
  if(optionTypes.includes(type)){
     // do something
  }
}
```

但是，目前我的库暂时只支持这三种输入，所以我希望对于参数类型做出一些更具体的声明，也就是通过常量对 `optionTypes` 做一个类型收窄，这个时候 ts 就会报错：

`TS2345: Argument of type 'string' is not assignable to parameter of type '"component" | "styled" | "lib-entry"'.`

```typescript {hl_lines=[1,6]}
const optionTypes = ['component', 'styled' ,'lib-entry'] as const

export async function checkOptionValue(options: Options) {
  let { type } = options
  // 判断用户输入是否支持
  if(optionTypes.includes(type)){ // ERROR
     // do something
  }
}
```

&nbsp;

## 原因

看一下 `includes` 的声明就发现问题了，第一个参数 `searchElement` 要求与数组本身的类型一致：

源码位置：`lib/lib.es2016.array.include.d.ts`

```typescript
interface ReadonlyArray<T> {
    /**
     * Determines whether an array includes a certain element, returning true or false as appropriate.
     * @param searchElement The element to search for.
     * @param fromIndex The position in this array at which to begin searching for searchElement.
     */
    includes(searchElement: T, fromIndex?: number): boolean;
}
```

好像也很合理，不过在类型收窄的情况下，如果我可以确定用户输入的参数是我需要的，那我还要 `includes` 干什么呢 :thinking:

退一步想，或许不是 ts 本身的问题，而是应该改变我代码的逻辑，但是既然问题都已经排查到这一步了，那我就继续下去，想想有什么办法可以处理吧。

&nbsp;

## 尝试解决

可以尝试用一个辅助函数来做断言，帮助 ts 推断出正确的信息：

```typescript
export function includesHelper<T extends U, U>(
  source: ReadonlyArray<T>,
  target: U
) {
  return source.includes(target as T)
}

```

将常量数组作为第一个参数，将常量数组通过`extends`约束为 target 的子集，然后在调用`includes`的时候将 target 断言为 `T`，这样做的话无论是代码逻辑还是类型推断都是成立的，因为当这个方法 return true 的时候，target 的断言自然是合法的，当 return false 的时候，类型推断自然也是非法的，是相对合理的做法。

不过，要记得这依然仅仅是为了通过 ts 的类型推断，所以业务逻辑还是需要自己去做 handle 以免报错：

```typescript
async function checkOptionValue(type: string): Promise<string> {
    // 输入了不支持的类型
  if (!includesHelper(OptionTypes, type)) {
    console.log(
      red(`目前支持支持创建类型: ${OptionTypes.join(',')}，请重新选择`)
    )
      // 处理，让用户自行选择一个支持的类型
    type = await handleOptionPrompt()
  }
  return type
}
```

&nbsp;

（完）





