---
weight: 2
title: "【JavaScript数据结构】二叉树实现"
summary: "用 JavaScript 实现二叉树结构"
date: 2021-12-20T20:05:57+08:00
draft: false
categories: ["技术"]
tags: ["JavaScript","数据结构","二叉树"]
---

## 二叉树定义

二叉树由一组相互链接的结点组合而成，最终表示一种具有层次的树形结构。

二叉树中每个结点通过`父-子`关系互相链接，每一个结点最多拥有两个子结点（即左结点与右结点）。

二叉树中的第一个结点被称为`根结点`，而没有子结点的结点称为`叶子结点`。

![二叉树](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/jsdatastructures/btree.png)

&nbsp;

### 在二叉树中，每个结点必须具有以下属性：

- `key`：结点的 key
- `value`：结点的值
- `parent`：结点的父节点（如果没有，用`null`表示）
- `left`：指向结点左子树的指针（如果没有，用`null`表示）
- `right`：指向结点右子树的指针（如果没有，用`null`表示）

### 二叉树的关键操作：

- `insert`：在给定的父结点中插入一个子结点
- `remove`：从二叉树中移除一个结点以及它的子结点
- `find`：检索给定的结点
- `preOrderTraversal`：前序遍历，通过递归遍历每个结点及其子结点来遍历二叉树
- `postOrderTraversal`：后序遍历，从子树最左侧的结点开始，先遍历完叶子结点，再访问父结点，最后访问根结点
- `inOrderTraversal`：中序遍历，最先访问左子树，然后访问根结点，最后访问右子树

&nbsp;

## 实现

### 结点

创建一个带有构造函数的类表示`结点`，为每个实例初始化必要属性

```javascript
class BinaryTreeNode {
  constructor(key, value = key, parent = null) {
    this.key = key
    this.value = value
    this.parent = parent
    this.left = null
    this.right = null
  }
}
```

定义一个 getter `isLeaf`，用于检查`left`和`right`属性是否为空

```javascript
class BinaryTreeNode {
  constructor(key, value = key, parent = null) {
    this.key = key
    this.value = value
    this.parent = parent
    this.left = null
    this.right = null
  }
    
  get isLeaf() {
    return this.left === null && this.right === null
  }
}
```

定义一个 getter `hasChildren`，与`isLeaf`正好相反，所以可以利用`isLeaf`来判断是否拥有至少一个子结点

```javascript
class BinaryTreeNode {
  constructor(key, value = key, parent = null) {
    this.key = key
    this.value = value
    this.parent = parent
    this.left = null
    this.right = null
  }

  get isLeaf() {
    return this.left === null && this.right === null
  }

  get hasChildren() {
    return !this.isLeaf
  }
}
```

&nbsp;

### 树

创建一个带有构造函数的类表示`二叉树`，为每个实例初始化根节点

```javascript
class BinaryTree {
  constructor(key, value = key) {
    this.root = new BinaryTreeNode(key, value)
  }
}
```

### 创建生成器方法：

生成器方法主要用于三种遍历的实现，通过`yield*`语法便可以简单实现递归

```javascript
class BinaryTree {
  constructor(key, value = key) {
    this.root = new BinaryTreeNode(key, value)
  }
    
  *inOrderTraversal(node = this.root) {
    if (node.left) yield* this.inOrderTraversal(node.left)
    yield node
    if (node.right) yield* this.inOrderTraversal(node.right)
  }

  *postOrderTraversal(node = this.root) {
    if (node.left) yield* this.postOrderTraversal(node.left)
    if (node.right) yield* this.postOrderTraversal(node.right)
    yield node
  }

  *preOrderTraversal(node = this.root) {
    yield node
    if (node.left) yield* this.preOrderTraversal(node.left)
    if (node.right) yield* this.preOrderTraversal(node.right)
  }
    
}
```

### 创建操作方法：

- 定义`insert()`方法，可以复用刚刚完成的构造器方法来遍历查找给定的父结点，并插入一个新的`BinaryTreeNode`作为左子结点或者右子结点，可以通过参数中传递的`option`对象来决定
- 定义`remove()`方法，可以复用刚刚完成的构造器方法来遍历查找给定的父结点，从二叉树中移除一个`BinaryTreeNode`
- 定义`find()`方法，可以复用刚刚完成的构造器方法来遍历检索给定的结点

```javascript
class BinaryTree {
  constructor(key, value = key) {
    this.root = new BinaryTreeNode(key, value)
  }

  *inOrderTraversal(node = this.root) {
    if (node.left) yield* this.inOrderTraversal(node.left)
    yield node
    if (node.right) yield* this.inOrderTraversal(node.right)
  }

  *postOrderTraversal(node = this.root) {
    if (node.left) yield* this.postOrderTraversal(node.left)
    if (node.right) yield* this.postOrderTraversal(node.right)
    yield node
  }

  *preOrderTraversal(node = this.root) {
    yield node
    if (node.left) yield* this.preOrderTraversal(node.left)
    if (node.right) yield* this.preOrderTraversal(node.right)
  }

  insert(
    parentNodeKey,key,value = key,
    { left, right } = { left: true, right: true }) {
    for (let node of this.preOrderTraversal()) {
      if (node.key === parentNodeKey) {
        const canInsertLeft = left && node.left === null
        const canInsertRight = right && node.right === null
        if (!canInsertLeft && !canInsertRight) return false
        if (canInsertLeft) {
          node.left = new BinaryTreeNode(key, value, node)
          return true
        }
        if (canInsertRight) {
          node.right = new BinaryTreeNode(key, value, node)
          return true
        }
      }
    }
    return false
  }

  remove(key) {
    for (let node of this.preOrderTraversal()) {
      if (node.left.key === key) {
        node.left = null
        return true
      }
      if (node.right.key === key) {
        node.right = null
        return true
      }
    }
    return false
  }

  find(key) {
    for (let node of this.preOrderTraversal()) {
      if (node.key === key) return node
    }
    return undefined
  }
}
```

&nbsp;

（完）



