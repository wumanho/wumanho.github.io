---
title: "DOM 事件与事件委托"
date: 2020-11-08T16:08:51+08:00
draft: false
categories: ["前端"]
tags: ["JavaScript","DOM事件"]
---

## 什么是 DOM 事件 :question:

DOM 事件是指用户在与浏览器交互时，或者浏览器自身运行时执行的某一个动作。  

发送 DOM 事件是为了将发生的相关事情通知代码。每个事件都是继承自 Event 接口的对象，可以包括自定义的成员属性及函数用于获取事件发生时相关的更多信息。JS 与 html 之间的交互是通过事件实现的，DOM 支持大量的事件，列举一些常用的事件：

| 事件名称                                                     | 何时触发                           |
| ------------------------------------------------------------ | ---------------------------------- |
| [error](https://developer.mozilla.org/zh-CN/docs/Web/Reference/Events/error) | 资源加载失败时。                   |
| [load](https://developer.mozilla.org/zh-CN/docs/Web/Reference/Events/load) | 资源及其相关资源已完成加载。       |
| [focus](https://developer.mozilla.org/zh-CN/docs/Web/Reference/Events/focus) | 元素获得焦点（不会冒泡）。         |
| [blur](https://developer.mozilla.org/zh-CN/docs/Web/Reference/Events/blur) | 元素失去焦点（不会冒泡）。         |
| [keydown](https://developer.mozilla.org/zh-CN/docs/Web/Reference/Events/keydown) | 按下任意按键。                     |
| [click](https://developer.mozilla.org/zh-CN/docs/Web/Reference/Events/click) | 在元素上按下并释放任意鼠标按键。   |
| [contextmenu](https://developer.mozilla.org/zh-CN/docs/Web/Reference/Events/contextmenu) | 右键点击（在右键菜单显示前触发）。 |
| [mousemove](https://developer.mozilla.org/zh-CN/docs/Web/Reference/Events/mousemove) | 指针在元素内移动时持续触发。       |
| [submit](https://developer.mozilla.org/zh-CN/docs/Web/Reference/Events/submit) | 点击提交按钮                       |

&nbsp;

## DOM 事件流 :raising_hand_man:

DOM 事件模型分为两种，一种是捕获，一种是冒泡，至于为什么是两种，这里其实涉及了多年前的浏览器厂商之间的战争和 W3C 委员会的介入，这里不多赘述。  

一个事件发生后，会在子元素和父元素之间传播，这种传播分为三个阶段。

1. 捕获阶段：事件从 document 对象开始由上往下传播；（网景提出）
2. 目标阶段：真正的目标节点正在处理事件的阶段；
3. 冒泡阶段：事件从目标节点由下往上向 document 对象传播的阶段；（微软提出）

![示意图](/images/event.png)

&nbsp;

开发者可以自己选择事件绑定到**捕获过程**还是**冒泡过程**。  

事件绑定 API：

```javascript
.attachEvent('onclick',fn) // 冒泡
.addEventListener('click',fn) //捕获
.addEventListener('click', fn, bool)  //如果第三个参数不为true，或者为空，就是冒泡，否则就是捕获
```

&nbsp;

## 事件委托 :handshake:

事件委托简单来说就是将一个元素的响应事件，委托到另一个元素上，为什么需要事件委托？可以想象一下以下场景：

- 场景一

  - 当要为多个按钮添加点击事件，怎么办？

    答：监听这多个按钮的祖先元素，等冒泡发生的时候判断 target 是不是这些元素中的一个。

    ```html
    <body>
        <div id="div1">
            <button>click 1</button>
            <button>click 2</button>
            <button>click 3</button>
            ...
        </div>
    </body>
    ```

    ```javascript
    //绑定事件到父元素 div1
    div1.addEventListener('click',(e)=>{
        //获取被点击的元素
        let target = e.target
        //如果被点击的元素的 tagName 是button,就...
        if(target.tagName.toLowerCase() === "button"){
           console.log("button 被点击了") 
        }
    })
    ```

&nbsp;

- 场景二

  - 需要监听目前不存在的元素的点击事件，怎么办？

    答：监听祖先元素，等点击的时候看看是不是我想要监听的元素即可。

    ```html
    <body>
        <div id="div1">
           <!-- 空的 -->
        </div>
    </body>
    ```

    ```javascript
    //一秒钟后，往 div1 中插入一个子元素 button
    setTimeout(()=>{
       const btn = document.createElement("button")
       btn.textContent = "click 1"
       div1.appendChild(btn)
    },1000)
    
    //在 div1 中添加监听，一秒钟后 button 被插入，点击 button 依然可以成功执行函数
    div1.addEventListener('click',(e)=>{
        let target = e.target
        if(target.tagName.toLowerCase() === "button"){
           console.log("button 被点击了") 
        }
    })
    ```

&nbsp;

:smirk:从上述演示中可以看出，事件委托的好处主要有两点：

- 省内存（只需要设置一个监听）
- 可以监听动态生成的元素

&nbsp;

（完）