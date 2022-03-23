---
weight: 2
title: "前端路由学习 - 实现一个简易 Vue-Router"
summary: "参考 Vue-Router 源码实现一个具有基本路由功能的插件"
date: 2020-02-12T10:52:44+08:00
draft: false
categories: ["前端"]
tags: ["JavaScript","Vue","Vue-Router"]
---

## 简介

前端路由应该是近几年随着单页应用（SPA）的兴起才开始被广泛讨论的概念，SPA 应用由于其特殊性，一个 WEB 项目只有一个页面，一旦页面加载完成，当前应用不会因为用户的操作而进行页面的重新加载或跳转，取而代之的是利用 JavaScript 动态的变换 HTML 的内容，从而来模拟多个视图间跳转。

SPA 大大提高了 WEB 应用的交互体验，在与用户的交互过程中，不再需要重新刷新页面，获取数据也是通过 Ajax 异步获取，页面显示变的更加流畅。

而实现 SPA 的关键概念之一，便是「路由」。

Vue-Router 便是 Vue 用来实现路由的插件，它的功能非常完善，且与 Vue 的核心深度集成。

&nbsp;

## 实现思路

出于学习的目的，在一次浅尝辄止的源码阅读以及参考了一些优秀文章之后，打算也尝试着自己写一个简单的 Vue-Router 插件，完成一些基本的路由跳转功能。

一开始感觉无从下手，先回忆一下平时是如何使用 Vue-Router 的：

先创建一个 router.js 配置文件，引入 Vue-Router，通过 Vue 定义的全局方法 `use` 注册插件，然后导出一个 Router 实例，将路由配置以参数的方式传入。

```javascript
import Vue from 'vue';
import Router from 'vue-router';
import C from '@/components/component.vue';
import Home from '@/components/home.vue';

Vue.use(Router)

export default new Router({
  routes: [{
    path: '/',
    component: Home
  }, {
    path: '/c',
    component: C
  }]
});
```

在 main.js 中初始化并挂载 router，

```javascript
/* main.js */
import Vue from 'vue'
import App from './App.vue'
import router from './router.js'

new Vue({
  router,
  render: h => h(App)
}).$mount('#app')
```

在 App.vue 中使用`router-link`以及`router-view`组件实现路由跳转和路由出口功能

```vue
<template>
<!-- APP.vue  -->
  <div id="app">
    <div id="nav">
       <!-- 路由跳转 -->
      <router-link to="/">Home</router-link>|
      <router-link to="/c">Component</router-link>
    </div>

    	<!-- 路由占位 -->
    	<router-view />
  </div>
</template>
```

于是，在开始之前，先站在小白的角度问自己几个问题：

1. 如何开发一个插件？
2. router-view 和 router-link 哪来的？
3. router-view 如何做到替换？

&nbsp;

「如何开发一个插件」这个问题，参考 [Vue 官方文档](https://cn.vuejs.org/v2/guide/plugins.html) 的说明：

> 如果插件是一个对象，必须提供 install 方法。如果插件是一个函数，它会被作为 install 方法。install 方法调用时，会将 Vue 作为参数传入。 该方法需要在调用 `new Vue()` 之前被调用。 当 install 方法被同一个插件多次调用，插件将只会被安装一次。

所以开发一个 Vue 插件，其实就是实现一个 install 方法，然后将方法暴露，用户便可以使用 `Vue.use()`注册插件了。



「router-view 和 router-link 哪来的」：

参考 [Vue-Router 源码](https://github.com/vuejs/vue-router/blob/HEAD/src/install.js)，这两个组件实际上是在 install 方法中注册的，所以后面需要实现这两个组件，分别用于路由跳转和路由出口。



「router-view 如何做到替换」：

根据 Vue-Router 原理，实现路由跳转组件替换，需要通过监听 `hashchange`事件，并响应最新的 url，根据 url 获取对应的组件并显示到界面上。

&nbsp;

有了思路，就可以开始动手了。:hammer:

&nbsp;

## 写一个「插件」

```javascript
/*my-vue-router.js*/
let Vue
class MyVueRouter{
    constructor(options){
        //先获取一下参数
        this.$options = options
    }
}

/**
 * 按照官方的做法，定义 install 方法
 * Vue规定第一个参数必须接受一个Vue实例
 * options 就是 use(xxx，options) 传入的第二个参数 options
 */
MyVueRouter.install = function(_Vue,options){
    Vue = _Vue //使用宿主环境的Vue实例
}
```

&nbsp;

## 挂载 $router

在日常开发的过程中，我们通常会在 main.js 里面，往根实例上挂载 router 选项，以便在组件中使用` $router` 来获取全局路由实例，所以接下来需要实现的便是挂载 $router。

不过如何获取根实例中的 router 选项是个问题，如果没有头绪的话，可以参考 Vue-Router 的源码，利用全局混入来实现。

由于 install 方法会在 `new Vue({})`之前执行，所以实际上只需要全局混入一个生命周期钩子，在钩子里面便可以实现全局挂载：

```javascript
/*my-vue-router.js*/
let Vue
class MyVueRouter{
    constructor(options){
        this.$options = options
    }
}

MyVueRouter.install = function(_Vue,options){
    Vue = _Vue 
    //全局混入
    Vue.mixin({
        //这个钩子会在所有组件中都执行一遍
        beforeCreate() {
            //需要判断当前实例是否根实例
            if (this.$options.router) {
                //实际上这里只会进来一次
                Vue.prototype.$router = this.$options.router
            }
        }
    })
}
```

&nbsp;

## router-view 组件以及路由跳转功能实现

根据之前的思路分析，我们需要通过 `hashchange`监听 url 的变化，然后将对应的 Vue 组件显示到页面上

```javascript
/*my-vue-router.js*/
let Vue
class MyVueRouter{
    constructor(options){
        this.$options = options
        
        //声明一个变量代表当前 url
        this.curl = "/"
    
        //监控url变化，赋值到当前 url
        window.addEventListener("hashchange",()=>{
            this.curl = window.location.hash.slice(1)
        })
        
        //遍历用户传入的路由配置，创建路由映射表
        this.routeMap = {}
        options.routes.forEach(route=>{
            this.routeMap[route.path] = route
        })
    }
}
```

有了`routeMap`以及`curl`当前路径，就可以实现 router-view 组件了：

```javascript
/*my-vue-router.js*/
let Vue
class MyVueRouter{
     /* ... */
}

MyVueRouter.install = function(_Vue,options){
    /* ... */
    
    Vue.component("router-view", {
        //在 runtime-only 环境下，需要利用 render 函数来描述组件
        render(h){
            //获取 path 对应的 component
            let {routeMap,curl} = this.$router
            // component = 当前 url 对应的组件名
            let component = routeMap[curl].component || null
            //将 component 用 h 函数渲染即可
            return h(component)
        }
    })
}
```

到目前位置，貌似一切顺利，但是当进行测试的时候会发现，**hash 发生变化的时候，页面并没有重新进行渲染**。

主要原因是，当前的变量`curl`，并非一个响应式的数据，当他发生改变时，页面并不会重新渲染，如何做到这一点？

这个时候可以利用 Vue 提供的工具`Vue.util.defineReactive`，对数据进行响应式处理，`defineReactive`是一个内部方法，Vue 源码中有时候会用这个方法来对数据做一些响应式处理，所以优化后的代码应该是：

```javascript
/*my-vue-router.js*/
let Vue
class MyVueRouter{
    constructor(options){
        this.$options = options
        
        //创建响应式的 curl 属性
        Vue.util.defineReactive(this,"curl",window.location.hash.slice(1) || "/")
    
        //监控url变化，赋值到当前 url
        window.addEventListener("hashchange",()=>{
            this.curl = window.location.hash.slice(1)
        })
        
        //遍历用户传入的路由配置，创建路由映射表
        this.routeMap = {}
        options.routes.forEach(route=>{
            this.routeMap[route.path] = route
        })
    }
}
```

&nbsp;

## router-link 组件实现

router-link 组件用于实现编程导航，通过组件中的`to`属性指定目标地址，router-link 最终会被渲染成带有正确链接的 `<a>` 标签。

```javascript
/*my-vue-router.js*/
let Vue
class MyVueRouter{
     /* ... */
}

MyVueRouter.install = function(_Vue,options){
    /* ... */
    
    Vue.component("router-link", {
        props: {
            to: {
                type: String,
                required: true
            }
        },
        render(h) {
            //渲染一个 a 标签: <a href="/c">component</a>
            return h("a", { attrs: { href: '#' + this.to } }, this.$slots.default)
        }
    })
}
```

&nbsp;

## 测试

至此，路由插件已经基本实现完成，将配置修改成自己的路由插件，测试看看效果：

![效果图](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/simplevuerouter/test.gif)

（完）