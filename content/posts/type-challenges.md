---
weight: 2
title: "【TypeScript】类型体操通关挑战（持续更新）"
summary: "从 easy 到 hard 通关 typescript 类型体操"
date: 2022-04-27T22:27:18+08:00
draft: false
categories: ["技术"]
tags: ["typescript","challenges","类型体操"]
---

## :bookmark:背景

**TS 类型体操**通俗来说是一系列关于 TS 类型的刷题挑战，本篇博客的挑战主要基于 [Github Type Challenges](https://github.com/type-challenges/type-challenges) 项目。

由于 TypeScript 是图灵完备的 JavaScript 的超集，也就是说 JS 能写的他都可以写。

而且跟传统的静态类型语言如 Java 不同，由于 JavaScript 本身是动态类型语言，而 TypeScript 是没有对 JavaScript 语法做任何改变的，可以看成是对 JS 的扩展，因为 TS 的类型系统需要考虑到这种动态性，所以 TS 允许开发者对类型进行编程，简单来说就是**对传入的类型参数（泛型）做各种逻辑运算，产生新的类型。** 总的来说是比单纯的泛型系统要复杂的。

所以，对于 TS 的类型编程，各位 Geek 精神满满的大佬就开始跃跃欲试，提出了类型编程挑战，戏称为 TS 类型体操。

&nbsp;

## 04_pick （简单）

### 题目

实现 TS 内置的 `Pick<T,K>`，不过不可以使用它本身。

**从类型 `T` 中选择出属性 `K`，构造成一个新的类型**。

例如：

```typescript
interface Todo {
  title: string
  description: string
  completed: boolean
}

type TodoPreview = MyPick<Todo, 'title' | 'completed'>

const todo: TodoPreview = {
    title: 'Clean room',
    completed: false,
}
```

### 知识点

* 联合类型 `|`
* 映射类型 `in、keyof`
* 约束 `extends`

### 解题

```typescript
type MyPick<T, K> = any
```

看题目可知泛型类型`T`实际上是 TODO 接口，`K`是一个联合类型

由于联合类型`K`必须是`T`的 key 值，所以这里需要使用约束，需要注意的是如果仅仅用 `K extenfs T`表达是不够的，因为`K`必须是`T`的**key值**，所以应该这样写：

```typescript
type MyPick<T,K extends keyof T> = {
    
}
```

然后在声明的新的类型中，key 对应的是传入的`K`值，value 对应的是 TODO 中对应的属性，所以只需要遍历生成即可：

```typescript
type MyPick<T,K extends keyof T> = {
    [P in K] : T[P]
}
```

大功告成。

&nbsp;

（持续更新中 ...）

