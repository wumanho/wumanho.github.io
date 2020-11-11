---
title: "Flex 布局常用属性值"
date: 2019-08-27T18:33:57+08:00
draft: false
categories: ["前端笔记"]
tags: ["CSS"]
---

## 前言

记录一些 Flex 常用属性值，主要给自己看的，方便自己查找。

Flex 布局有两个关键字：

1. 容器
2. 元素

&nbsp;

## Container

让一个元素变成一个 flex 容器：

```css
.container{
    display:flex;
}
```

## 改变 items 的流动方向（控制主轴）

```css
flex-direction: column;  			/*从上往下排列*/
flex-direction: column-reverse;      /*从下往上排列*/
flex-direction: row;     			/*从左往右排列，默认值*/
flex-direction: row-reverse;     	/*从右往左排列*/
```

## 改变折行

如果不做特殊的判断，弹性盒一行有多少空间就挤多少元素。

```css
flex-wrap: nowrap;			/*默认值，不折行*/
flex-wrap: wrap;			/*自动折行，且不会拆开元素*/
```

## 主轴对齐方式

```css
justify-content: flex-start;		/*默认值，往主轴开始的地方挤*/
justify-content: flex-end;			/*往主轴结束的地方挤*/
justify-content: center;			/*居中，常用*/
justify-content: space-between;		/*把空间放到元素的间隙中*/
justify-content: space-around;		/*把空间放到元素的周围*/
```

## 次轴对齐方式

次轴又叫交叉轴，如果主轴是从左向右，次轴就是从上往下

```css
align-items: flex-start;		/*默认值，往次轴开始的地方挤*/
align-items: center;			/*居中*/
align-items: flex-end;			/*往次轴结束的地方挤*/
```

&nbsp;

## items

flex 容器的子元素就是 items。

## item 上面加 order

用于对 items 排序，默认是  0

```css
order: 1;    /*靠前*/
order: 100;  /*靠后*/
order: -3;   /*负数排最前面，比 0 小*/
```

## item 上面加 flex-grow

当元素没有宽度时，控制宽度如何分配

```css
flex-grow: 0;		/*默认值*/
flex-grwo: 1;		/*分配一份*/
flex-grow: auto;	/*只要有多余的，全给这个元素*/
```

