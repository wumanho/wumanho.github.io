---
weight: 2
title: "【JavaScript数据结构】链表和双向链表"
date: 2021-11-14T10:03:57+08:00
draft: false
categories: ["技术"]
tags: ["JavaScript","数据结构","链表"]
---

由于最近在复习一些数据结构的知识，作为练习，这篇博客会尝试利用 JavaScript 以最简单的方式实现两个数据结构，`链表`和`双向链表`。

&nbsp;

# 链表

---

## 定义

`链表`是一种线性的数据结构，用于表示一组元素的集合。

其中，每一个元素都会指向下一个元素，链表里面第一个元素称作链表的`头部`，最后一个元素称作链表的`尾部`

![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/jsdatastructures/linklist.png)

### 在链表中，每个元素必须具有以下两个属性：

- `value`：当前元素的值
- `next`：指向下一个元素的指针（如果没有下一个元素，则为 `null`）

### 链表的关键属性：

- `size`：用于表示当前链表里面元素的数量
- `head`：指向当前链表中的第一个元素
- `tail`：指向当前链表中的最后一个元素

### 链表的关键操作：

- `insertAt`：在指定的索引中插入一个元素
- `removeAt`：在指定的索引中移除一个元素
- `getAt`：检索指定的索引中的元素
- `clear`：清空链表
- `reverse`：反转链表中的元素顺序

&nbsp;

## 实现

创建一个带有构造函数的类，为每个实例初始化一个空数组

```javascript
class LinkedList {
  constructor() {
    this.nodes = []
  }
}
```

### 创建关键属性：

- 将`size`通过 getter 语法定义为计算属性，`size`返回 nodes 数组长度以表示链表中元素的个数
- 将`head`通过 getter 语法定义为计算属性，`head`返回 nodes 数组中第一个元素，如果没有则返回 null
- 将`tail`通过 getter 语法定义为计算属性，`tail`返回 nodes 数组中最后一个元素，如果没有则返回 null

```javascript
class LinkedList {
  constructor() {
    this.nodes = []
  }
  //关键属性 size
  get size() {
    return this.nodes.length
  }
  //关键属性 head,利用 size 来判断 nodes 是否为空
  get head() {
    return this.size ? this.nodes[0] : null
  }
  //关键属性 tail,利用 size 来判断 nodes 是否为空
  get tail() {
    return this.size ? this.nodes[this.size - 1] : null
  }
}
```

### 创建操作方法：

- 定义`insertAt()`方法，该方法利用`Array.prototype.splice()`为 nodes 数组插入一个新的元素，**并且更新上一个元素的 next 值**
- 定义`insertFirst()`和`insertLast()`，这两个方法不是必要的，不过由于有了`insertAt()`，可以通过复用它来增加一些更加方便的操作，例如在头部插入一个元素以及在尾部插入一个元素
- 定义`getAt()`方法，用于检索参数中给定的索引所对应的值
- 定义`removeAt()`方法，该方法利用`Array.prototype.splice()`为 nodes 数组移除一个元素，**并且更新上一个元素的 next 值**
- 定义`clear()`方法，用于清空 nodes 数组
- 定义`reverse()`方法，该方法利用`Array.prototype.reduce()`和扩展运算符使 nodes 数组反转，**并且重新设置每一个元素的 next 值**
- 通过`Symbol.iterator`定义一个生成器方法，使链表成为可迭代对象

```javascript
class LinkedList {
  constructor() {
    this.nodes = []
  }

  get size() {
    return this.nodes.length
  }

  get head() {
    return this.size ? this.nodes[0] : null
  }

  get tail() {
    return this.size ? this.nodes[this.size - 1] : null
  }

  insertAt(index, value) {
    const previousNode = this.nodes[index - 1] || null
    const nextNode = this.nodes[index] || null
    //定义元素
    const node = { value, next: nextNode }
	//更新 next
    if (previousNode) previousNode.next = node
    this.nodes.splice(index, 0, node)
  }

  insertFirst(value) {
    this.insertAt(0, value)
  }

  insertLast(value) {
    this.insertAt(this.size, value)
  }

  getAt(index) {
    return this.nodes[index]
  }

  removeAt(index) {
    const previousNode = this.nodes[index - 1]
    const nextNode = this.nodes[index + 1] || null
	//更新 next
    if (previousNode) previousNode.next = nextNode

    return this.nodes.splice(index, 1)
  }

  clear() {
    this.nodes = []
  }

  reverse() {
    this.nodes = this.nodes.reduce(
      (acc, { value }) => [{ value, next: acc[0] || null }, ...acc],
      []
    )
  }

  *[Symbol.iterator]() {
    yield* this.nodes
  }
}
```

&nbsp;

# 双向链表

---

## 定义

`双向链表`和`链表`类似，都属于线性数据结构。

不同的是，`双向链表`中，每一个元素除了拥有指向下一个元素的指针，还具有指向前一个元素的指针。

![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/jsdatastructures/doublelinklist.png)

### 在链表中，每个元素必须具有以下三个属性：

- `value`：当前元素的值
- `next`：指向下一个元素的指针（如果没有下一个元素，则为 `null`）
- `previous`：指向上一个元素的指针（如果没有上一个元素，则为 `null`）

### 双向链表的关键属性：

- `size`：用于表示当前双向链表里面元素的数量
- `head`：指向当前双向链表中的第一个元素
- `tail`：指向当前双向链表中的最后一个元素

### 双向链表的关键操作：

- `insertAt`：在指定的索引中插入一个元素
- `removeAt`：在指定的索引中移除一个元素
- `getAt`：检索指定的索引中的元素
- `clear`：清空双向链表
- `reverse`：反转双向链表中的元素顺序

&nbsp;

## 实现

创建一个带有构造函数的类，为每个实例初始化一个空数组

```js
class DoublyLinkedList {
  constructor() {
    this.nodes = []
  }
}
```

### 创建关键属性：

- 将`size`通过 getter 语法定义为计算属性，`size`返回 nodes 数组长度以表示链表中元素的个数
- 将`head`通过 getter 语法定义为计算属性，`head`返回 nodes 数组中第一个元素，如果没有则返回 null
- 将`tail`通过 getter 语法定义为计算属性，`tail`返回 nodes 数组中最后一个元素，如果没有则返回 null

```javascript
class DoublyLinkedList {
  constructor() {
    this.nodes = []
  }
  //关键属性 size  
  get size() {
    return this.nodes.length
  }
  //关键属性 head,利用 size 来判断 nodes 是否为空
  get head() {
    return this.size ? this.nodes[0] : null
  }
  //关键属性 tail,利用 size 来判断 nodes 是否为空
  get tail() {
    return this.size ? this.nodes[this.size - 1] : null
  }
}
```

### 创建操作方法：

- 定义`insertAt()`方法，该方法利用`Array.prototype.splice()`为 nodes 数组插入一个新的元素，**并且分别为上一个元素以及下一个元素更新 next 值和 previous 值**
- 定义`insertFirst()`和`insertLast()`，这两个方法不是必要的，不过由于有了`insertAt()`，可以通过复用它来增加一些更加方便的操作，例如在头部插入一个元素以及在尾部插入一个元素
- 定义`getAt()`方法，用于检索参数中给定的索引所对应的值
- 定义`removeAt()`方法，该方法利用`Array.prototype.splice()`为 nodes 数组移除一个元素，**并且分别为上一个元素以及下一个元素更新 next 值和 previous 值**
- 定义`clear()`方法，用于清空 nodes 数组
- 定义`reverse()`方法，该方法利用`Array.prototype.reduce()`和扩展运算符使 nodes 数组反转，**并且重新设置每一个元素的 next 值以及 previous 值**
- 通过`Symbol.iterator`定义一个生成器方法，使双向链表成为可迭代对象

```javascript
class DoublyLinkedList {
  constructor() {
    this.nodes = []
  }

  get size() {
    return this.nodes.length
  }

  get head() {
    return this.size ? this.nodes[0] : null
  }

  get tail() {
    return this.size ? this.nodes[this.size - 1] : null
  }

  insertAt(index, value) {
    const previousNode = this.nodes[index - 1] || null
    const nextNode = this.nodes[index] || null
    //定义元素
    const node = { value, next: nextNode, previous: previousNode }
	//更新 next 和 previous
    if (previousNode) previousNode.next = node
    if (nextNode) nextNode.previous = node
    this.nodes.splice(index, 0, node)
  }

  insertFirst(value) {
    this.insertAt(0, value)
  }

  insertLast(value) {
    this.insertAt(this.size, value)
  }

  getAt(index) {
    return this.nodes[index]
  }

  removeAt(index) {
    const previousNode = this.nodes[index - 1] || null
    const nextNode = this.nodes[index + 1] || null
	//更新 next 和 previous
    if (previousNode) previousNode.next = nextNode
    if (nextNode) nextNode.previous = previousNode

    return this.nodes.splice(index, 1)
  }

  clear() {
    this.nodes = []
  }

  reverse() {
    this.nodes = this.nodes.reduce((acc, { value }) => {
      const nextNode = acc[0] || null
      const node = { value, next: nextNode, previous: null }
      if (nextNode) nextNode.previous = node
      return [node, ...acc]
    }, [])
  }

  *[Symbol.iterator]() {
    yield* this.nodes
  }
}
```

&nbsp;

（完）

&nbsp;
