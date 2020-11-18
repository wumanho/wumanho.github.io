---
title: "JavaScript 对象基本用法"
date: 2020-04-13T11:42:26+08:00
draft: false
categories: ["前端"]
tags: ["JavaScript"]
---

# 如何声明一个对象  

方法一： 
```JavaScript
let obj = {'name':'wu','age':18}
```
方法二： 
```JavaScript
let obj = new Object({'name':'wu','age':18})
```
&nbsp;
&nbsp;  

# 如何删除对象的属性  
使用 **delete** 关键字即可删除对象属性
```JavaScript
let obj = {
    name:"wumanho"
}

delete obj.name //删除obj的name属性

console.dir(obj)
```
&nbsp;
&nbsp;  

# 如何查看对象的属性   
```JavaScript
let obj = {
    name:"wumanho",
    age:18
}
//查看对象自身拥有的属性
Object.keys(obj)
//查看对象所有属性，包括共有属性
console.dir(obj)
//判断一个属性是对象自身的还是公有属性
obj.hasOwnProperty("toString")
```
&nbsp;
&nbsp;  

# 如何修改或增加对象的属性  
```JavaScript
let obj = {
    name:"wumanho",
    age:18
}
/*
直接使用 对象.属性名 的方式
如果属性存在，则修改，属性不存在，则新增
*/
obj.gender = "male"  //新增gender属性
console.dir(obj)
```
&nbsp;
&nbsp;  

# “name” in obj和obj.hasOwnProperty('name') 

* 共同点   
'name' in obj  以及  obj.hasOwnProperty('name')都用于判断对象 obj 是否存在某个属性值。

* 不同点  
'name' in obj 可以判断对象的**公有属性**以及对象自身的**私有属性**是否包含"name"  
obj.hasOwnProperty('name') 顾名思义，只能判断对象自身的**私有属性**是否包含"name"