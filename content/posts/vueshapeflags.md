---
weight: 2
title: "Vue3 源码中的位运算"
summary: "位运算在实际工程中的妙用"
date: 2022-04-06T20:21:51+08:00
draft: false
categories: ["前端"]
tags: ["Vue3","源码","位运算","shapeFlag"]
---

## 前言

[位运算操作](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Expressions_and_Operators#%E4%BD%8D%E8%BF%90%E7%AE%97%E7%AC%A6) 在几乎所有编程语言中都被支持，位运算符可以将操作数转化成二进制串，然后在二进制的基础上执行运算，然后返回标准的数值。

由于位运算符属于低级运算操作，所以运算速度也是最快的，并且可以代替最基础的加减乘除运算，因此经常出现在一些条件有限制的场合中，例如算法题，最经典的就是[不使用 + 和 - 计算两个整数之和](https://leetcode-cn.com/problems/sum-of-two-integers/)。

但位运算操作也有很明显的缺点，就是不直观，在业务代码中使用位运算更是不值得推崇的做法。

霍春阳大佬的《Vue.js 设计与实现》中，在第一篇的第一章里面就有提到，Vue 3 在`框架设计里到处都体现了权衡的艺术`，指的是 Vue3 在设计上有大量对于**性能**和**可维护性**之间选择的权衡，所以 Vue3 对于位运算的妙用，是非常值得学习的地方。

&nbsp;

## 渲染器和 VNode

VNode 俗称虚拟 DOM，是一种用 JavaScript 对象描述 DOM 结构的一种方式，而渲染器就是 Vue 用于将 VNode 转译成真实 DOM 的模块。

Vue 提供了一个`h`函数，用户可以利用这个`h`函数来手动编写`render`，`h`接受三个参数：

* type：描述元素的类型
* props：描述元素拥有的属性
* children：子节点

最后，**返回一个 VNode 虚拟 DOM 结点**。

为了方便理解，可以用一个简化版的`createVNode`函数来说明（实际上`h`函数只是 Vue 为用户提供的简化调用方式，`h`函数内部调用的方法正是`createVNode`）：

```typescript
export function createVNode(type, props?, children?) {
    const vnode = {
        type,
        props,
        children
    }
    return vnode
}
```

举个例子，如果需要使用`h()`函数来创建以下这段 HTML 内容：

```vue
<div class="red">
    <div>div text</div>
    <p>Hello</p>
</div>	
```

那么应该这样写：

```typescript
render(){
    return h(
       "div",
       {class:"red"},
       [
        h("div",{},"div text"),
        h("p",{},"Hello")
     ])
}
```

看起来似乎非常完美:sunglasses:，结构清晰层次分明。

不过仔细想一想，如果当前渲染的节点的 children 不是普通 HTML 元素，而是一个`组件`：

```vue
<div>
    <!-- vue 组件 -->
    <Component></Component>
</div>	
```

这时候，Vue 应该做的就是渲染出`Component`组件内的所有元素，而不是一个 HTML 标签，这种情况应该怎样用 VNode 来表示呢？

&nbsp;

## 添加 shapeFlag

本篇博客的重点并不是源码分析，所以就不卖关子了，Vue 在渲染内容时，会在`patch`阶段判断 VNode 类型，根据 VNode 的类型去走不同的渲染逻辑。

而判断类型的依据就是本次的主要学习对象`shapeFlag`，`shapeFlags`是一个枚举对象，它最特殊的点是利用了二进制来表示 VNode 的各个属性：

```typescript
/** packages/shared/src/shapeFlags.ts **/

export const enum ShapeFlags {
  ELEMENT = 1, 
  FUNCTIONAL_COMPONENT = 1 << 1,
  STATEFUL_COMPONENT = 1 << 2,
  TEXT_CHILDREN = 1 << 3,
  ARRAY_CHILDREN = 1 << 4,
  SLOTS_CHILDREN = 1 << 5,
  TELEPORT = 1 << 6,
  SUSPENSE = 1 << 7,
  COMPONENT_SHOULD_KEEP_ALIVE = 1 << 8,
  COMPONENT_KEPT_ALIVE = 1 << 9,
  COMPONENT = ShapeFlags.STATEFUL_COMPONENT | ShapeFlags.FUNCTIONAL_COMPONENT
}

```

对应的值如下表：

|          ShapeFlag          |       对应节点类型       |  位移  |   二进制位图   |
| :-------------------------: | :----------------------: | :----: | :------------: |
|           ELEMENT           |           HTML           |   1    | 000000000**1** |
|    FUNCTIONAL_COMPONENT     |        函数式组件        | 1 << 1 | 00000000**1**0 |
|     STATEFUL_COMPONENT      |         普通组件         | 1 << 2 | 0000000**1**00 |
|        TEXT_CHILDREN        |      子节点是纯文本      | 1 << 3 | 000000**1**000 |
|       ARRAY_CHILDREN        |       子节点是数组       | 1 << 4 | 00000**1**0000 |
|       SLOTS_CHILDREN        |       子节点是插槽       | 1 << 5 | 0000**1**00000 |
|          TELEPORT           |          传送门          | 1 << 6 | 000**1**000000 |
|          SUSPENSE           |         SUSPENSE         | 1 << 7 | 00**1**0000000 |
| COMPONENT_SHOULD_KEEP_ALIVE | 将要被 keep alive 的组件 | 1 << 8 | 0**1**00000000 |
|    COMPONENT_KEPT_ALIVE     | 已经被 keep alive 的组件 | 1 << 9 | **1**000000000 |

### 为 VNode 添加状态

定义了枚举类型`shapeFlag`，就可以对之前的`createVNode`进行一些改造，创建 VNode 时将它标记为某种节点，这是为了对`patch`过程进行优化。

例如，判断`createVNode`函数的第一个参数的类型，如果是字符串类型，就可以将当前节点定义为`ELEMENT` 节点：

```typescript
export function createVNode(type, props?, children?) {
    const vnode = {
        type,
        props,
        children,
        shapeFlag: getShapeFlag(type),
    }
    return vnode
}

function getShapeFlag(type) {
    // 判断是元素或者是组件
    return typeof type === 'string' ? ShapeFlags.ELEMENT : ShapeFlags.STATEFUL_COMPONENT
}
```

然后，如果当前节点**拥有子节点**的话，则可以同时标记子节点的类型。

也就是说这时候需要让一个节点同时拥有两种状态，例如：**当前节点是ELEMENT**并且**子节点是数组**，要实现这一点，只要使用位运算的`或`运算符即可。

`或`运算 `a | b` ，对比每一个比特位，当同一位上的数有一个为1，结果就为1，否则结果为0。

于是，可以通过判断子节点的类型，添加一个 shapeFlag 标记：

```typescript
export function createVNode(type, props?, children?) {
    const vnode = {
        type,
        props,
        children,
        shapeFlag: getShapeFlag(type),
    }
    // 为 children 添加 shapeFlag
    // 通过位运算处理
    if (typeof children === 'string') {
        vnode.shapeFlag = vnode.shapeFlag | ShapeFlags.TEXT_CHILDREN
    } else if (Array.isArray(children)) {
        vnode.shapeFlag = vnode.shapeFlag | ShapeFlags.ARRAY_CHILDREN
    }
    return vnode
}

function getShapeFlag(type) {
    // 判断是元素或者是组件
    return typeof type === 'string' ? ShapeFlags.ELEMENT : ShapeFlags.STATEFUL_COMPONENT
}
```

如果当前 VNode 的 shapeFlag 已经被标记为 ELEMENT，则二进制位图表示为：000000000**1**，

如果当前 VNode 的子节点是数组，则二进制位图表示为：00000**1**0000，

将两个数值进行`|`运算后，便可以得出新的 shapeFlag 的值为：00000**1**000**1**

### 判断 VNode 的 shapeFlag 状态

有了 shapeFlag 之后，渲染器在`patch`的时候就可以根据`与`运算，判断 VNode 类型，再决定渲染逻辑就可以了。

`与`运算 `a & b` ，对比每一个比特位，当同一位上的数都为1，结果才为1，否则结果为0。

```typescript
function patch(vnode, container) {
    //判断 vode 是否 element，element 要单独处理
    //element：{type:'div',props:'hello'}
    //组件：{ type:APP{render()=>{} ,setup()=>{} }}
    const {shapeFlag} = vnode
    if (shapeFlag & ShapeFlags.ELEMENT) {
        //处理 element
        processElement(vnode, container)
    } else if (shapeFlag & ShapeFlags.STATEFUL_COMPONENT) {
        //处理组件
        processComponent(vnode, container)
    }
}
```

处理 element 时，还需要判断子组件的 shapeFlag：

```typescript
function processElement(vnode, container) {
    const {children, props, shapeFlag} = vnode
    //创建元素
    const el = (vnode.el = document.createElement(vnode.type))
    //创建元素内容
    if (shapeFlag & ShapeFlags.TEXT_CHILDREN) {
        el.textContent = children
    } else if (shapeFlag & ShapeFlags.ARRAY_CHILDREN) {  //children 为数组
        vnode.children.forEach(v => {
        //再次调用 patch 处理数组中每个 children
        patch(v, container)
    })
  }
    /**....**/
}


```

接上面的逻辑，假设 VNode 的 shapeFlag 被标记为：00000**1**000**1**，

用 VNode 的 shapeFlag 标记去与 `ShapeFlags.ELEMENT`的值 000000000**1** 进行`与`运算，可以得到一个非0的值，非0的值用于`if`判断，属于`true`值，所以判断有效。

&nbsp;

## 总结

位运算由于在实际开发中并不会经常用到，所以在学习 Vue3 源码时看到 shapeFlag 觉得还是比较有意思的。

但实际上 Vue3 对于 VNode 的操作和定义要比上面所描述的复杂得多，由于 VNode 的设计涉及到最关键的性能部分，可以说每一行代码都是经过仔细优化的，很难用一篇博客去详细展开这些细节，所以这次就仅仅是对 shapeFlag 这个点进行总结和学习，也已经感觉到获益良多。

&nbsp;

（完）

