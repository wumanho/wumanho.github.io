---
title: "CSS总结"
date: 2020-03-30T10:02:12+08:00
draft: false
categories: ["前端笔记"]
tags: ["CSS"]
---


# 浏览器的渲染原理
## 渲染树构建
要解释浏览器的渲染原理，需要先了解三棵树：
* DOM树
* CSSOM树
* Render树  

DOM树通过解析HTML/SVG/XHTML三种文件产生。  

CSSOM树通过解析CSS产生。  

Render树则是由DOM树和CSSOM树合并而成。  

![tree](/images/render-tree-construction.png)

## 布局与绘制
当浏览器生成渲染树以后，就会根据渲染树来进行布局（也可以叫做回流）。这一阶段浏览器要做的事情是要弄清楚各个节点在页面中的确切位置和大小。通常这一行为也被称为“自动重排”。  

布局流程的输出是一个“盒模型”，它会精确地捕获每个元素在视口内的确切位置和尺寸，所有相对测量值都将转换为屏幕上的绝对像素。  

布局完成后，浏览器会立即发出“Paint Setup”和“Paint”事件，将渲染树转换成屏幕上的像素。  

下面简要概述了浏览器完成渲染的步骤：  
1. 处理 HTML 标记并构建 DOM 树。  
2. 处理 CSS 标记并构建 CSSOM 树。  
3. 将 DOM 与 CSSOM 合并成一个渲染树。  
4. 根据渲染树来布局，以计算每个节点的几何信息。  
5. 将各个节点绘制到屏幕上。
## 更新样式
更新样式一般会通过 JS 来操作。  
```JavaScript
div.style.background = "red"; //设置背景色
div.style.display = "none"; //设置display
div.remove(); //删除节点
```

更新方式共有三种，根据更新样式的不同，触发不同的更新方式：  
![update](/images/treeupdate.png)    
<br/><br/>

# CSS 动画的两种做法
## transform
transform四种常用属性
* translate 移动
```CSS
transform: translateX(200px);       /* 向X轴移动200像素 */
transform: translateY(100px);       /* 向Y轴移动100像素 */
transform: translate(50px,50px);  /*同时向X轴移动50像素,向Y轴移动50像素*/
```
* scale 缩放
```CSS
transform: scale(1.5);  /* 放大为原来的1.5倍 */
```

* rotate 旋转
```CSS
transform: rotate(45deg);           /* 顺时针旋转45度 */
transform: rotate(-45deg);          /* 逆时针旋转45度 */
```

* skew 倾斜
```CSS
transform: skew(30deg);             /* 水平方向倾斜30度 */
transform: skew(30deg, 30deg);      /* 水平方向倾斜30度,//垂直方向倾斜30度 */
```

transition: 属性名 | 时长 | 过渡方式 | 延迟
```CSS
transition:right 1ms linear 1ms;    /* 1ms内完成向右移动，动画形式为线性表示，延迟为1ms */
transition:all 5s ease 1ms;         /* 5s内完成所有动画，动画形式为缓速表示，延迟为1ms */
```
## animation
animation可以解决transform的一些痛点，就是使用transform制作多段动画比较麻烦。  

animation语法和属性：  

* animation: 时长 | 过渡方式 | 延迟 | 次数 | 方向 | 填充模式 | 是否暂停 | 动画名  

* 时长: 可以使用s或者ms作为单位，表示动画从开始到结束花费的时长  

* 过渡方式: 动画通过什么方式进行转换，默认ease

* 次数: 有限次数或者infinite(无限)  

* 方向: reverse(从结束向开始)、alternate(开始与结束相互循环)  

* 填充模式: forward(定格在最后一帧)

* 暂停：paused | running  
  

@keyframes  
keyframes是与animation结合使用的模块，常用语法:
```CSS
@keyframes 名称{
 0%{top:0;}     /* 动画开始时的动作 */
 50%{top:5px;}  /* 动画进行到50%的动作 */
 100%{top:5px; top:10px;}   /* 动画结束时的动作 */
}
```

# 其他想写的
    没想到CSS这么难



