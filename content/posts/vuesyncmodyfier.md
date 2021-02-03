---
weight: 2
title: "理解 .sync 修饰符"
date: 2020-10-03T11:26:08+08:00
draft: false
categories: ["前端"]
tags: ["Vue",".Sync"]
---

[Vue](https://vuejs.org/) 在 2.3 版本得时候引入了`.sync`修饰符，它在此前的版本曾经被移除，后来又以新的方式重新加入。

<!--more-->

很多时候我们希望使用双向绑定来简化大量业务无关的代码，不过双向绑定虽然方便，但是同时也增加了维护难度，`.sync`修饰符本质上是一个语法糖，它为我们提供了一种新的在 Vue 父子组件之间传递的`props`双向绑定的选择。

我们知道 Vue 组件不能直接修改 props 外部数据，但是：

* `$emit` 可以触发事件，并传递数据；
* `$event` 可以获取 `$emit` 事件总线的参数；

也就是说在此之前，父子组件之间的`props`进行双向绑定，思路大概是这样的：

1. 父子组件之间通过`props`传值；
2. 子组件通过`$emit("event",content)`事件总线传递动作，父组件监听自定义事件*event*，当触发事件时，父组件修改对应的值；

&nbsp;

## 事件总线传值

通过一个实例来解释一下父子组件通过事件总线传值：

```vue
//===ParentComponent.vue===
<template>
  <div class="app">
    ParentComponent.vue 数值：{{total}}
    <hr>
    <Child :counter="total" @update:counter="total = $event"/>
  </div>
</template>

<script>
import Child from "./Child.vue";
export default {
  data() {
    return { total: 0 };
  },
  components: { Child: Child }
};
</script>

<style>
.app {
  border: 2px solid red;
  padding: 10px;
}
</style>
```

```vue
//===Child.vue===
<template>
  <div class="child">
    Child.vue 数值：{{counter}}
    <button @click="$emit('update:counter', counter + 1)">
      <span> + </span>
    </button>
    <button @click="$emit('update:counter', counter - 1)">
      <span> - </span>
    </button>
  </div>
</template>

<script>
export default {
  props: ["counter"]
};
</script>


<style>
.child {
  border: 2px solid blue;
}
</style>
```

![before](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/vuesync/before.gif)

这是 Vue 被普通用于实现 props 双向绑定的方式，但是这样做也有明显的缺点，就是我们在父组件中很难看出 props 是被双向绑定的，当有大量数据相互依赖相互绑定，导致数据问题的源头难以被跟踪到，不利于管理数据源。

&nbsp;

## 引入.sync修饰符

利用`.sync`修饰符来实现跟上面一样的双向绑定

```vue
//===ParentComponent.vue===
<template>
  <div class="app">
    ParentComponent.vue .sync数值：{{total}}
    <hr>
    <Child :counter.sync="total"/>
  </div>
</template>

<script>
import Child from "./Child.vue";
export default {
  data() {
    return { total: 0 };
  },
  components: { Child: Child }
};
</script>

<style>
.app {
  border: 2px solid red;
  padding: 10px;
}
</style>
```



```vue
//===Child.vue===
<template>
  <div class="child">
    Child.vue .sync数值：{{counter}}
    <button @click="$emit('update:counter', counter + 1)">
      <span> + </span>
    </button>
    <button @click="$emit('update:counter', counter - 1)">
      <span> - </span>
    </button>
  </div>
</template>

<script>
export default {
  props: ["counter"]
};
</script>


<style>
.child {
  border: 2px solid blue;
}
</style>
```

![after](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/vuesync/after.gif)

可以看到在 *ParentComponent.vue* 中，舍弃了`@update:counter`事件监听，执行的结果是一样的。

&nbsp;

## 总结

使用 Vue 提供的`.sync`修饰符，带来了一定的便利性，而且可以显式地表示哪些 Props 被子组件双向绑定，哪些 props 不会被更改，对代码的可读性有了一定提高。

而`.sync`语法则其实只是`:counter="total" @update:counter="total = $event"`的缩写，他的本质依然是从父组件利用 props 传值到子组件，然后监听子组件的自定义事件。

&nbsp;

(完)