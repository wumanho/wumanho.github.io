---
title: "【TypeScript】类型体操_简单题"
date: 2022-05-07T22:15:39+08:00
draft: false
hiddenFromHomePage: true
categories: ["技术"]
tags: ["typescript","challenges","类型体操"]
---

## 04_pick

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
// 签名
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

### test-case

```typescript
import type { Equal, Expect } from '@type-challenges/utils'

type cases = [
    Expect<Equal<Expected1, MyPick<Todo, 'title'>>>,
    Expect<Equal<Expected2, MyPick<Todo, 'title' | 'completed'>>>,
    // @ts-expect-error
    MyPick<Todo, 'title' | 'completed' | 'invalid'>,
]

interface Todo {
    title: string
    description: string
    completed: boolean
}

interface Expected1 {
    title: string
}

interface Expected2 {
    title: string
    completed: boolean
}

```

大功告成。

&nbsp;

## 07_readonly

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
// 签名
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

### test-case

```typescript
import type { Equal, Expect } from '@type-challenges/utils'

type cases = [
  Expect<Equal<MyReadonly<Todo1>, Readonly<Todo1>>>,
]

interface Todo1 {
  title: string
  description: string
  completed: boolean
  meta: {
    author: string
  }
}

```

大功告成。

&nbsp;

## 11_Tuple_to_Object

### 题目

传入一个元组类型，将这个元组类型转换为对象类型，这个对象类型的键/值都是从元组中遍历出来。

例如：

```typescript
const tuple = ['tesla', 'model 3', 'model X', 'model Y'] as const

type result = TupleToObject<typeof tuple> // expected { tesla: 'tesla', 'model 3': 'model 3', 'model X': 'model X', 'model Y': 'model Y'}

```

### 知识点

* const 断言
* 遍历数组

### 解题

```typescript
// 签名
type TupleToObject<T extends readonly any[]> = any
```

做这道题需要一些前置知识，就是理解 `as const` 所代表的意义，是 ts 在 3.4 版本中新增的功能，称为 `const 断言` ，具体请参考[官方文档](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-4.html#const-assertions)，在这道题中，`as const` 的作用就是声明一个字面量只读数组：

```typescript
const tuple = ['tesla', 'model 3', 'model X', 'model Y'] as const

type r = typeof tuple
const arr: r = ["aba"] // TS2322: Type '"aba"' is not assignable to type '"tesla"'.
```

在知道传入的参数到底是什么玩意之后，我们可以先将给到的签名修改一下，约束传入的泛型 `T` 应该是一个只有字符串的数组，而且应该返回一个对象：

```typescript
type TupleToObject<T extends readonly string[]> = {
    
}
```

然后就需要遍历传入的数组，将提取数组中的值为对象的 key ，value 则是与之相同的值。

在 TS 中遍历一个数组，可以采用 `in T[number]` 的方式：

```typescript
type TupleToObject<T extends readonly string[]> = {
    [F in T[number]]: F
}
```

### test-case

```typescript
import type {Equal, Expect} from '@type-challenges/utils'

// 在 ts 中 as const = '字面量类型'，跟 js 的 const 有一定区别
const tuple = ['tesla', 'model 3', 'model X', 'model Y'] as const

type cases = [
    Expect<Equal<TupleToObject<typeof tuple>, { tesla: 'tesla'; 'model 3': 'model 3'; 'model X': 'model X'; 'model Y': 'model Y' }>>,
]

// @ts-expect-error
type error = TupleToObject<[[1, 2], {}]>
```

在测试用例中，最下面有一个 `@ts-expect-error` 的注解声明，意思是如果下面的语句不报错，TS 就会报错，我们在上面传入参数时已经限制了泛型类型 `T` 为 `readonly string[]` ，所以用例中这样传参肯定会抛出错误的，测试通过。

&nbsp;

## 14_First

### 题目

实现一个通用`First<T>`，它接受一个数组`T`并返回它的第一个元素的类型。

例如：

```typescript
type arr1 = ['a', 'b', 'c']
type arr2 = [3, 2, 1]

type head1 = First<arr1> // expected to be 'a'
type head2 = First<arr2> // expected to be 3
```

### 知识点

* `never`
* 数组下标取值
* `infer 推断`

### 解题

```typescript
// 签名
type First<T extends any[]> = any
```

按照惯例先约束一下泛型类型 `T`，这一题需要注意，在后面的测试用例中，作为参数的数组并没有限制特定的类型，所以应该使用`unknown[]`约束比较合适：

```typescript
type First<T extends unknown[]> = any
```

在 TS 中获取数组的第一个元素方法有很多，这次我将会尝试使用其中两种解法来解这道题，一种是比较简单的，一种是相对进阶的解法。

先来看看最简单的**解法一**：判断传入的参数是否为空数组，如果是空数组，返回`never`，否则返回数组的第一个元素，在 TS 中访问数组下标只需要使用`T[index]`即可

```typescript
type First<T extends unknown[]> = T extends [] ? never : T[0]
```

**解法二**，使用关键字`infer`来对数组进行操作，`infer`关键字官方翻译为推断，`infer`必须要结合`extends`一起使用，可以在条件语句中声明一个变量，接收待推断的类型，例如：

```typescript
// 可以获取函数返回值的类型
type getReturnType<T> = T extends (...args:any[]) => infer R ? R : any

type funType = getReturnType<(str: string) => string> // string
```

所以，只要使用`infer`来推断参数数组中第一个元素，返回就可以了：

```typescript
type First<T extends unknown[]> = T extends [infer First,...unknown[]] ? First : never
```

### test-case

```typescript
import type { Equal, Expect } from '@type-challenges/utils'

type cases = [
  Expect<Equal<First<[3, 2, 1]>, 3>>,
  Expect<Equal<First<[() => 123, { a: string }]>, () => 123>>,
  Expect<Equal<First<[]>, never>>,
  Expect<Equal<First<[undefined]>, undefined>>,
]

type errors = [
  // @ts-expect-error
  First<'notArray'>,
  // @ts-expect-error
  First<{ 0: 'arrayLike' }>,
]

```

&nbsp;

## 18_Length_of_Tuple

### 题目

创建一个通用的`Length`，接受一个`readonly`的数组，返回这个数组的长度。

例如：

```typescript
type tesla = ['tesla', 'model 3', 'model X', 'model Y']
type spaceX = ['FALCON 9', 'FALCON HEAVY', 'DRAGON', 'STARSHIP', 'HUMAN SPACEFLIGHT']

type teslaLength = Length<tesla> // expected 4
type spaceXLength = Length<spaceX> // expected 5
```

### 知识点

* 数组长度
* 元组

### 解题

```typescript
// 签名
type Length<T> = any
```

首先处理一下测试用例中的 `@ts-expect-error` ，只要加上约束即可，参数是字符串元组，元组是只读的，所以：

```typescript
type Length<T extends readonly string[]> = any
```

然后，TS 中取数组长度有一个最简单的方法，就是 `T["length"]` ，所以这题可以直接用这个方法解：

```typescript
type Length<T extends readonly string[]> = T["length"]
```

### test-case

```typescript
import type { Equal, Expect } from '@type-challenges/utils'

const tesla = ['tesla', 'model 3', 'model X', 'model Y'] as const
const spaceX = ['FALCON 9', 'FALCON HEAVY', 'DRAGON', 'STARSHIP', 'HUMAN SPACEFLIGHT'] as const

type cases = [
  Expect<Equal<Length<typeof tesla>, 4>>,
  Expect<Equal<Length<typeof spaceX>, 5>>,
  // @ts-expect-error
  Length<5>,
  // @ts-expect-error
  Length<'hello world'>,
]
```

测试通过。

&nbsp;

（持续更新中 ...）
