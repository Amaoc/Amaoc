---
title: new 具体做了什么事？
categories:
  - 前端开发
tags:
  - js
toc: true # 是否启用内容索引
---

通俗来说，我们都知道new一般做了下面四件事：

>　　1、创建一个空对象；
>
>　　2、将空对象的原型，指向于构造函数的原型；
>
>　　3、将空对象作为构造函数的上下文（改变this指向）；
>
>　　4、对有返回值的构造函数做判断处理

代码实现：
```js
//定义构造函数
function Fun(age, name) {
    this.age = age
    this.name = name
    return 1
}

function myNew(fn, ...args) {
    //1、先创造空对象
    //其实等于var obj = Object.create({})  
    var obj = {}
    //2、obj的__proto__指向原型
    Object.setPrototypeOf(obj, fn.prototype)
    //3、改变this指向，执行构造函数内部函数
    var result = fn.apply(obj, args)
    //4、判断return
    return result instanceof Object ? result : obj
}
```