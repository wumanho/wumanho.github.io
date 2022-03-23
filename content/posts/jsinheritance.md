---
weight: 2
title: "JavaScript 继承介绍与应用"
date: 2020-03-11T15:03:57+08:00
draft: false
categories: ["前端"]
tags: ["JavaScript"]
---

&emsp;&emsp;「继承」在面向对象编程中属于核心概念，不过 JavaScript 并不是严格意义上的面向对象语言，JS 不像传统的面向对象编程语言例如 Java 那样，倾向于通过先定义类，定义继承关系，再创建对象，而是使用了另一种概念： 「原型」和「原型链」实现了类似继承的特性。   

## 创建构造函数 :man_mechanic:

在 ES6 以前，由于没有`class`关键字，我们会使用**构造函数**来创建对象。

{{< admonition tip "小提示" >}}按照规范，构造函数的函数名通常会*首字母大写*{{< /admonition >}}

```javascript
//声明一个构造函数，用于创建人类
function Person(firstName, lastName) {
    this.FirstName = firstName || "unknown"
    this.LastName = lastName || "unknown"
};

Person.prototype.getFullName = function () {
    return this.FirstName + " " + this.LastName
}
```

在上面的例子中，我们定义了一个用于构造`Person`对象的构造函数，包含了私有属性`FirstName`、`LastName`以及**共有**的函数`getFullName`。  

&nbsp;

## 创建 Person 对象 :pouting_man:

定义了构造函数之后，我们可以使用 JS 提供的`new`关键字来创建出 Person 对象。

```javascript
let student1 = new Person("zhang","san")
let student2 = new Person("li","si")
```

实际上，`new`关键字在背后偷偷做了几件事：

1. 自动创建空对象
2. 自动为空对象关联原型，原型地址指定为 Person.prototype
3. 将 this 指定为刚刚创建的空对象
4. 自动 return this

所以，*student1*和*student2*这两个对象的原型都是 Person.prototype ，来证明一下：

```javascript
student1.getFullName === student2.getFullName //true
```

&nbsp;

## 继承 Person 对象 :man_student:

现在我们可以考虑一下，有没有办法可以定义一个对象拥有 Person 属性的同时又可以拥有自己的属性，例如 Student 对象，除了姓名外，还有学校，成绩等属性。实现这个过程就叫做「继承」。

```javascript
function Student(firstName, lastName, schoolName, grade){
    //构造函数绑定，将父类的构造函数绑定在子类上
    Person.call(this, firstName, lastName)
	//定义子类私有属性
    this.SchoolName = schoolName || "unknown"
    this.Grade = grade || 0
}
//Student.prototype = Person.prototype
Student.prototype = new Person()
Student.prototype.constructor = Student
```

这种继承方式一般叫做「组合继承」，即通过调用父类构造，继承父类的属性并保留传参的优点，然后通过将父类实例作为子类原型，实现了函数复用。

创建对象：

```javascript
let student1 = new Student("zhang","san","MIT","100")
let student2 = new Student("li","si","MIT","60")

console.log(student1.getFullName()) //zhangsan
console.log(student1 instanceof Student) //true
console.log(student1 instanceof Person) //true
```

&nbsp;

## ES6 面向对象 :angel:

在 ES6 之后，JavaScript 引入了`class`和`extends`关键字，变得跟 Java 更像了:dizzy_face:，使用这两个关键字可以更加简洁地用 JavaScript 实现类的定义和继承，不过这仅仅是一个`语法糖`，JavaScript 的面向对象今天依然是基于 Prototype 实现的。

### 定义 Person 类

```javascript
class Person{
    constructor(firstName, lastName) {
    this.firstName = firstName || "unknown"
    this.lastName = lastName || "unknown"
  }
    getFullName(){
        return this.firstName + " " + this.lastName
    }
}
```

### 继承 Person 类

```javascript
class Student extends Person{
    constructor(firstName, lastName, schoolName, grade) {
    	super(firstName, lastName)
    	this.schoolName = schoolName
    	this.grade = grade
  }
    getFullName(){
        return this.firstName + " " + this.lastName + " " + this.schoolName + " " + this.grade
    }
}
```

创建 Student 对象：

```javascript
let student1 = new Student("zhang","san","MIT","100")
let student2 = new Student("li","si","MIT","60")

console.log(student1.getFullName()) //zhang san MIT 100
console.log(student1 instanceof Student) //true
console.log(student1 instanceof Person) //true
```

结果跟使用原型链实现的继承是一样的。

&nbsp;

（完）

&nbsp;

### 参考

[JavaScript 继承机制的设计思想 - 阮一峰](http://www.ruanyifeng.com/blog/2011/06/designing_ideas_of_inheritance_mechanism_in_javascript.html)

