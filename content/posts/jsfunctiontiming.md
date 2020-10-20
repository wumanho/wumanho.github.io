---
title: "JavaScript 函数的执行时机"
date: 2020-10-16T11:59:41+08:00
draft: false
---

# 解释为什么如下代码会打印 6 个 6
```JavaScript
let i = 0
for(i = 0; i<6; i++){
  setTimeout(()=>{
    console.log(i)
  },0)
}
```  
setTimeout 函数的主要用于在一段时间之后，执行相应的延迟操作，且只会运行一次，不会重复执行，所以上面代码执行过程如下：  
1. 语句 let i = 0 声明了一个变量 i  
2. 变量 i 的作用域在 for 循环之外。  
3. for 循环开始，i 循环 6 次，每一次循环 i 自增1，执行 setTimeout，但由于setTimeout的延迟特性，不会立即打印出 i。
4. ，**循环结束后，i 的值为6**，setTimeout 执行回调内容，打印出 6 个值为 6 的变量 i


&nbsp;
&nbsp;

# 写出让上面代码打印 0、1、2、3、4、5 的方法
只要配合 ES6 的 **let** 关键字即可  

```JavaScript
for(let i = 0; i<6; i++){
  setTimeout(()=>{
    console.log(i)
  },0)
}
```
&nbsp;
&nbsp;

# 除了使用 for let 配合，还有什么其他方法可以
# 打印出 0、1、2、3、4、5
解决方法是要确定 setTimeout 函数的每一个“i”都是独立的副本  
参考：[最高票回答](https://stackoverflow.com/questions/5226285/settimeout-in-for-loop-does-not-print-consecutive-values)

```JavaScript
function doSetTimeout(i) {
  setTimeout(()=>{
    console.log(i)
  },0)
}

for (var i = 0; i < 6; ++i)
  doSetTimeout(i)
```