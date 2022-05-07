---
title: "【TypeScript】类型体操_简单题"
date: 2022-05-07T22:15:39+08:00
draft: false
hiddenFromHomePage: true
categories: ["技术"]
tags: ["typescript","challenges","类型体操"]
---

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
    [F in K] : T[F]
}
```

大功告成。

&nbsp;

## 07_readonly （简单）

### 题目

实现`Readonly<T>`，接收一个 **泛型参数**，并返回一个完全一样的类型，只是所有属性都会被 `readonly` 所修饰。

例如：

```typescript
interface Todo {
  title: string
  description: string
}

const todo: MyReadonly<Todo> = {
  title: "Hey",
  description: "foobar"
}

todo.title = "Hello" // Error: cannot reassign a readonly property
todo.description = "barFoo" // Error: cannot reassign a readonly property
```

### 知识点

* readonly
* 映射类型 `in、keyof`

### 解题

```typescript
type MyReadonly<T> = any
```

题目要求接受一个泛型类型`T`，`T`是一个接口，需要做的就是将接口内的所有字段添加上`readonly`属性。

在 ts 中添加 readonly 属性很简单，只要在字段名称添加 readonly 声明即可，例如：

```typescript
interface Todo {
  readonly title: string // title属性只读
  description: string
}
```

所以，解这道题只需要遍历泛型类型`T`的所有字段，并在字段前添加上`readonly`属性：

```typescript
type MyReadonly<T> = {
   readonly [F in keyof T]:T[F]
}
```

大功告成。

&nbsp;
