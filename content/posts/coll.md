---
weight: 2
title: "【Vue 函数式组件】实现不定高元素折叠动画"
summary: "参考 Element UI 源码实现的函数式组件 "
date: 2021-10-11T20:22:49+08:00
draft: false
categories: ["前端"]
tags: ["JavaScript","Element","函数式组件"]
---

## 背景

前段时间在做一个带有手风琴折叠效果的列表时，由于列表的长度不固定，以至于仅靠过渡动画实现的效果不够丝滑，没有达到预期的标准，于是后来参考了一下 Element UI 的源码，实现了一个简化版的函数式组件，最终效果如下（太懒所以就直接把业务组件搬来展示了:roll_eyes:）：

![效果](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/coll/list.gif)

## Vue 函数式组件简介

### 函数式组件有什么用

在日常开发的过程中，如果我们需要一个**不需要管理任何状态**的组件，也就是说这个组件可能只是简单地接受一些 props 参数，而不需要对数据进行响应式处理，不需要生命周期方法，也不需要 this 上下文，Vue 提供了一个可选的方案，这便是`函数式组件`。

简单来说，函数式组件的存在主要是为了提升性能，因为函数式组件的初始化速度比有状态组件快得多，不过根据 Vue 官方文档描述，由于 Vue `3.x` 版本对于有状态组件的性能已经提高到**可以使这两者之间的区别可以忽略不计**的程度，所以如果项目中使用的是 `3.x`版本，则没有必要使用函数式组件了。

### 如何使用函数式组件

使用函数式组件，只要声明选项`functional: true`即可：

```javascript
//collapseTtansition.js
export default {
  name: 'CollapseTransition',
  functional: true, //声明为函数式组件
  render(h, context) {
   // ...
  },
};
```

&nbsp;

## 实现代码

实现平滑过渡的原理也比较简单，核心思路是通过 `Vue 过渡`的 JavaScript 钩子获取需要折叠的元素的高度，再动态去赋值，完整代码：

```javascript
//collapseTtansition.js

//过渡效果
const transitionStyle = '0.3s height ease-in-out';

//过渡 javascript 钩子实现
const Transition = {
  beforeEnter(el) {
    el.style.transition = transitionStyle;
    el.style.height = 0;
  },

  enter(el) {
    if (el.scrollHeight !== 0) {
      el.style.height = `${el.scrollHeight}px`;
    } else {
      el.style.height = '';
    }
    el.style.overflow = 'hidden';
  },

  afterEnter(el) {
    el.style.transition = '';
    el.style.height = '';
  },

  beforeLeave(el) {
    el.style.height = `${el.scrollHeight}px`;
    el.style.overflow = 'hidden';
  },

  leave(el) {
    if (el.scrollHeight !== 0) {
      el.style.transition = transitionStyle;
      el.style.height = 0;
    }
  },

  afterLeave(el) {
    el.style.transition = '';
    el.style.height = '';
  },
};

export default {
  name: 'CollapseTransition',
  functional: true,
  render(h, { children }) {
    const data = {
      on: Transition,
    };
    return h('transition', data, children);
  },
};
```

在业务组件中引入并使用，使用方法类似 Vue 的 `<transition>`：

```vue
<template>
	<collapse-transition>
    	<ul 
            v-show="item.code === current && item.children" 
            class="li-dropdown-list">
        	<li :class="['dropdown-list-item', plan.id === currentChildren?'active':'']" 
                v-for="plan in item.children"
                :key="plan.id"
                @click="_listItemClickHandle(plan)">
            </li>
        </ul>
	</collapse-transition>
</template>

<script>
//IdtList.vue
import CollapseTransition from './collapseTtansition.js'
export default {
  components: {
    CollapseTransition
  },
}
</script>
```

&nbsp;

（完）

&nbsp;

### 参考

[Vue 过渡](https://cn.vuejs.org/v2/guide/transitions.html#JavaScript-%E9%92%A9%E5%AD%90)

[Vue 函数式组件](https://cn.vuejs.org/v2/guide/render-function.html#%E5%87%BD%E6%95%B0%E5%BC%8F%E7%BB%84%E4%BB%B6)

[ElementUI](https://github.com/ElemeFE/element/blob/dev/src/transitions/collapse-transition.js)
