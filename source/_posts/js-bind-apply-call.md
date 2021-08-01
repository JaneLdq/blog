---
title: JavaScript - bind, apply, call
date: 2018-01-05 15:20:27
categories: 技术笔记
tags: JavaScript
---

Function.prototyp.bind
```javascript
func.bind(thisArg[, arg1[, arg2[, ...]]]);
```
bind()方法可以指定函数运行的上下文，bind的第一个参数用于绑定this的指向，其后的参数在函数真正调用时会被添加在实际调用的输入参数之前。
举个例子
<!--more-->
```javascript
var apple = {
    name: "apple",
    sayHi: function(a, b) {
        var friends = a + ", " + b;
        console.log("Hello " + friends + ", I am " + this.name);
    }
}
apple.sayHi("Jane", "Charles"); // "Hello Jane, Charles, I am apple"
var orange = {
    name: "orange"
}
var orangeSayHi = apple.sayHi.bind(orange, "Jane", "Charles"); // this指向orange， 同时"Jane", "Charles"将作为a, b传入
orangeSayHi(); // 所以在调用时即使不传参也会得到"Hello Jane, Charles, I am orange"
```

bind()函数到底做了什么呢？参考MDN的文档描述，bind()函数创建了一个新的Bound Funcion(BF), 它相当于在原始函数外又进行了一次封装。之后的每次调用都是执行的这个封装过的函数。
一个BF新增了一些内部属性：

* `[[BoundTargetFunction]]` - 被封装的原始函数
* `[[BoundThis]` - 绑定的this
* `[[BoundArguments]]` - 在绑定时除this外的其他参数，它们在之后的任意调用中都会被加到所有传入参数的最前面
* `[[Call]]` - executes code associated with this object. Invoked via a function call expression. The arguments to the internal method are a this value and a list containing the arguments passed to the function by a call expression.

当一个BF被调用时，它会使用它内部的`[[Call]]`来调用`[[BoundTargetFunction]]`，以`[[BoundThis]]`作`为thisArg`, `[[BoundArguments]]`加上函数调用时传入的其他参数`arg1`, `arg2`, ...作为参数，如下所示：
`BoundTargetFunction.Call(BoundThis, BoundArguments[, arg1[, arg2[, ...]]])`

下图就是上面的例子中orangeSayHi这个BF的内部结构，可以直观感受一下~
![bound function example][1]

[1]: http://static.zybuluo.com/JaneL/c78mkeq3ad5vlvck6e1cub8k/image.png
