---
weight: 2
title: "前端 MVC"
date: 2020-09-05T17:19:01+08:00
draft: false
categories: ["前端"]
tags: ["JavaScript","MVC"]
---

&emsp;前端 MVC 其实早在 2014 年前后已经引起过大量讨论，在当下已经不是什么非常新鲜的概念。  

由于前端领域起步和发展本身比较晚，所以 MVC 本身是从传统软件工程开发中借鉴而来的思想，彻底改变了前端领域在初期时那种不受约束的意大利面条式代码组织方式，同时为前端带来了更多的模块化的思想。  

随着前端领域的发展，如今已经延申出许多如「MVP」、「MVVM」等其他类似的概念，不过本篇博客还是着重于介绍最经典的 MVC 模式。

&nbsp;

## Model 模型层 :dancer:

**It simply means data**.  

实际上任何与**数据**相关的对象或者数据结构都可以封装到 Model ，除此之外，对于数据的增删改查或其他用于操作数据的方法也将会被封装到 Model  里面。  

以下示例演示了一个最简单的 model 模型，他包含了数据本身和对数据的增删改查操作。

```javascript
let model = {
    //数据对象
    data:{
        /*从数据源获取数据*/
    },
    
    //操作数据的方法
    create(){
        /*对数据的增加操作*/
    },
    delete(){
        /*对数据的删除操作*/
    },
    update(){
        /*对数据的修改操作*/
    },
    query(){
        /*对数据的查询操作*/
    }
}
```

&nbsp;

## View 视图层 :tv:

顾名思义，视图层是 MVC 三层中用于被展示图到用户屏幕的一部分，视图层的数据来源于 model 当模型的数据发生变化，视图将会刷新自己。

以下示例展示了一个最基础的视图层对象：

```javascript
let view = {
    template: /*页面内容*/,
    
    render(data){
        /*获取数据，并刷新页面的操作*/
    }
}
```

&nbsp;

## Controller 控制器 :video_game:

Controller 通常扮演着视图层和模型层的纽带这么一种角色，所以通常来说例如全局的初始化以及与事件相关的操作等等都是写在 Controller 中的。  

不过由于 Controller 与 View 有着密不可分的关系，例如 Controller 执行全局初始化的时候，可能需要先从 View 中获取某些元素，必须先拿到这些元素，才能执行全局初始化，所以后来 View 和 Controller 有时候会被合并，因此 MVC 中的「C」是一个值得讨论的问题。

以下示例展示了一个最基础的控制器对象：

```javascript
let Controller = {
    init(){
        /*执行初始化操作*/
        view.render.call(view,model.data) //第一次渲染页面
        this.autoBindEvents() 			  //自动绑定事件
        eventBus.on(/*监听传递事件*/)     //很多时候需要使用 eventBus 处理一些组件间传递的事件
    },
    method(){
        /*事件操作*/
    },
    autoBindEvents(){
        /*事件绑定*/
    }
}
```

&nbsp;

### eventBus 事件总线 :bus:

说到这里顺便提一下事件总线`eventBus`，它主要被用于解决模块以及组件之间通信的问题，例如当某一个数据被修改时，可能涉及到其他多处信息的展示更新，这时候如果每一次都要重复这种调用就会让代码显得繁琐而且不可靠，所以利用`eventBus`承担起一个事件中心的角色。

一般来说`eventBus`只需要包含几个关键的 API：

```javascript
on()        //用于事件订阅
trigger()   //用于事件发布
off()       //用于取消事件订阅
```

实际上 jquery 和 vue 中所构建的对象都包含以上三个方法，所以如果想要初始化一个事件总线，其实只要创建一个全局变量，例如：

```javascript
const eventBus = $(window)
```

{{< admonition note "小提示" >}}
用于事件发布的 API，在 jquery 对象中称为 *trigger*，而在 vue 对象中则称为 *$emit*
{{< /admonition >}}

然后就可以利用这个全局变量的三个 API `on()`，`trigger()`，`off()`来进行事件传递了。

&nbsp;

## 关于模块化 :microphone:

JavaScript 和其他常见的高级语言不同，JS 在设计初期是没有考虑到包、模块这种概念的，不过随着前端项目的复杂度增加，前端开发者必然需要通过一些手段来组织代码。

这些手段就是模块化，其实模块化可以简单理解为对于复杂代码的隔离、封装等操作。

其实 MVC 就是模块化的体现之一，MVC 将一个页面上的不同部分分离，降低代码耦合度，减少重复代码，提高代码重用性，并且在项目结构上更加清晰，便于维护。

其实在 JavaScript 中还有很多其他对模块化的体现，例如：`CommonJS`、`AMD`,`CMD`，还有 ES6 的 `import/export`等。

