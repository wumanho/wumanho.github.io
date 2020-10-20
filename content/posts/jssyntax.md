---
title: "JS 基本语法"
date: 2020-10-11T15:47:04+08:00
draft: false
---

# 什么是表达式和语句
## 表达式
表达式一般都会有“值”，例如：  
* 表达式 1 + 2 的值为3
* 表达式add(1,2)的值为函数add的**返回值**
* console.log 表达式的值为函数本身  
## 语句
语句一般用于声明变量、赋值变量，但不是绝对的，例如：  
var a = 1 是一个语句  
<br/>

# 标识符的规则
最常见的标识符就是变量名：  
* var _ = 1  
* var $ = 2
* var 你好 = "hello"  

## 规则  
标识符的第一个字符可以是Unicode字母或者$或者下划线或者中文  
后面的字符除了上面说的，还可以使用数字  

# if else 语句
## 语法
* if(表达式){语句1}else{语句2}
* 花括号{}如果语句只有一句的情况下，可以省略，不过不建议这么做
```JavaScript
var a = 5
if(a < 10){
    console.log("a小于10")
}else{
    console.log("a大于10")
}
```
多个判断逻辑可以使用if else if组合
```JavaScript
var a = 5
if(a < 10){
    console.log("a小于10")
}else if(a > 10){
    console.log("a大于10")
}else{
    console.log("a等于10")
}
```
# while for语句  
    while和for语句主要用于循环  

## while循环语句
语法：while(表达式){语句}

1. 判断表达式的真假
2. 当表达式为真，执行语句，执行完再判断表达式的真假
3. 当表达式为假，跳过代码块语句，执行后面的逻辑
```JavaScript
var a = 0
while(a < 5){
    console.log(a)
    a++
}
```  
## for循环语句
for语句可以视为while语句的语法糖，因为同样的循环逻辑，for语句写起来更加方便  
语法：for(语句1;表达式2;语句3){循环体}  
1. 先执行语句1
2. 然后判断表达式2
3. 如果为真，执行循环体，然后执行语句3
4. 如果为假，直接退出循环，执行后面的语句
```JavaScript
for(let i=0; i<10; i++){
    console.log(i)
}
```
## break continue
* break  
    break主要用于退出整个循环，如果是嵌套循环的话，则只会退出离break最近的那个循环  

* continue  
    continue主要用于退出当前一次循环，会继续进入下一次的循环  

# label
    label语句,又称标记语句.作用是在语句前面加个可以引用的标识符。相当于将一条语句存储在一个变量里面,类似函数的函数名  

## 语法
```JavaScript
foo : {
    console.log(1);
    break foo; //退出foo标签
    console.log("这一行不会输出")
}
```