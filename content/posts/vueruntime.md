---
weight: 2
title: "Vue 运行时版本简介"
date: 2020-12-28T15:54:49+08:00
draft: false
categories: ["前端"]
tags: ["JavaScript","Vue"]
---

在刚刚开始使用 Vue 构建项目时，可能会遇到一个疑惑，就是 Vue 的`完整版`与`非完整版`到底有什么区别。

实际上[官方文档](https://cn.vuejs.org/v2/guide/installation.html#%E5%AF%B9%E4%B8%8D%E5%90%8C%E6%9E%84%E5%BB%BA%E7%89%88%E6%9C%AC%E7%9A%84%E8%A7%A3%E9%87%8A)对于各个版本已经有较明确的解析，所以本文着重于对比两个版本的差异。

&nbsp;

## 特点

### 完整版

根据官方的说法，完整版的意思为**同时包含「编译器」和运行时的版本**。编译器主要用于将模板字符串编译成为 JavaScript 渲染函数的代码。

### 运行时版本

运行时版本即**非完整版**，非完整版不包含「编译器」。主要用于创建 Vue 实例、渲染并处理虚拟 DOM 等的代码。基本上就是除去编译器的其它一切。

&nbsp;

## 视图

### 完整版

由于完整版包含「编译器」，所以可以直接写在 HTML 里，或者传入字符串给`template`选项

```javascript
// 需要编译器
new Vue({
  template: '<div>{{ hi }}</div>'
})
```

### 运行时版本

运行时版则只能在 render 函数里用 h 来创建标签，比较麻烦

```javascript
//不需要编译器
new Vue({
  render (h) {
    return h('div', this.hi)
  }
})
```

&nbsp;

## 命名方式

两个版本的命名方式不同，引用的名字也不同

### 完整版

完整版命名是`vue.js`，引用方式：

```html
<script src="https://cdn.bootcdn.net/ajax/libs/vue/2.6.10/vue.min.js"></script>
```

### 运行时版本

运行时版本命名是`vue.runtime.js`，引用方式：

```html
<script src="https://cdn.bootcdn.net/ajax/libs/vue/2.6.10/vue.runtime.min.js"></script>
```

&nbsp;

## Vue Loader

由于完整版包含编译器，所以体积会比运行时版本大，两者的体积大概相差 30% 左右。

所以为了保证性能最优，建议应该尽可能使用`运行时版本`，但是如上述所说，运行时版本只能使用非常难用的 render 函数创建节点，有没有什么办法可以解决呢？

答案就是使用`Vue Loader`，Vue Loader 是一个 [webpack](https://webpack.js.org/) 的 loader，它允许你以一种 Vue 独有的[单文件组件 (SFCs)](https://vue-loader.vuejs.org/zh/spec.html)的格式撰写 Vue 组件：

```vue
<!--test.vue-->
<template>
  <div class="div1">
    {{ number }}
    <button @click="add">+1</button>
  </div>
</template>
```

以上 .vue 文件代码，会被 Vue loader 加载，并自动转译为如下的 render 函数

```javascript
render() {
  var _vm = this
  var _h = _vm.$createElement
  var _c = _vm._self._c || _h
  return _c("div", { staticClass: "div1" }, [
    _vm._v(" " + _vm._s(_vm.number) + " "),
    _c("button", { on: { click: _vm.add } }, [_vm._v("+1")])
  ])
}
```

可以看到，以 .vue 方式组织的代码可读性更高，配合 Vue Loader 自动转译，开发者可以只专注于自己的业务逻辑而不必关心转译结果。

### Vue CLI 自动配置

如果你不想手动设置 webpack，可以使用 [Vue CLI](https://github.com/vuejs/vue-cli) 直接创建一个项目的脚手架，Vue Loader 会自动配置在 webpack 中，开箱即用。

### 手动设置

手动配置 Vue Loader 可以参考如下配置

```javascript
// webpack.config.js
const VueLoaderPlugin = require('vue-loader/lib/plugin')

module.exports = {
  mode: 'development',
  module: {
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader'
      },
      // 它会应用到普通的 `.js` 文件
      // 以及 `.vue` 文件中的 `<script>` 块
      {
        test: /\.js$/,
        loader: 'babel-loader'
      },
      // 它会应用到普通的 `.css` 文件
      // 以及 `.vue` 文件中的 `<style>` 块
      {
        test: /\.css$/,
        use: [
          'vue-style-loader',
          'css-loader'
        ]
      }
    ]
  },
  plugins: [
    // 请确保引入这个插件
    new VueLoaderPlugin()
  ]
}
```

&nbsp;

（完）

### 参考

[vue 官方文档](https://cn.vuejs.org/v2/guide/installation.html)