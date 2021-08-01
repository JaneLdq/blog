---
title: JavaScript中的闭包是什么？
date: 2018-12-18 19:43:32
categories: 技术笔记
tags: JavaScript
---

上周正好在跟同事一起看了一个跟闭包有一点点相关的小问题，忘性太大，赶紧温习一波。
记得最开始听到”闭包“这个词的时候觉得好高深，但是看英文原词”Closure“就感觉还好诶ˊ_>ˋ

---

不过在认识闭包之前，我们先要弄清楚两个概念：**执行环境(execution context)**和**作用域(scope)**。

# 执行环境与作用域是什么？

一个变量或函数（在js中，函数的本质也是一个变量）的执行环境**定义了它当前可以访问的数据（可以读到什么）和它应有的行为（可以做些什么操作）**。每个执行环境都关联了一个**变量对象**(variable object)，环境中定义的所有变量和函数都保存在这个对象中。

全局环境是指最外层的一个执行环境，根据ECMAScript实现的宿主环境不同，表示全局环境的变量对象也不同。在Web浏览器中，window对象都代表了全局执行环境，所有的全局变量和函数都被创建为window对象的属性和方法。
<!--more-->

每个函数都有自己的执行环境，当进入一个函数时，这个函数的执行环境被压入一个专门管理执行环境的栈中，当这个函数执行完成后，这个函数的执行环境从栈中弹出，控制权交还给外层函数的执行环境。

当代码在某个执行环境中时，会为这个环境的变量对象创建一个**作用域链(scope chain)**。既然是个**链**，也就表示它是**有序的**。作用域链的目的就是**保证对执行环境有权访问的所有变量和函数的有序访问**。当前执行的代码所处的执行环境最优先，由内而外直到全局环境。

函数运行时对标识符的解析就是沿着作用域链进行搜索哒。


举个例子：
```javascript
// 最外层一个匿名函数，它的执行环境里面定义了一个变量outerColor和一个函数changeColor
(function() {
    var outerColor = "blue";
    // 内层函数changeColor，它的执行环境里定义了一个变量innerColor和一个函数swapColors
    function changeColor() {
        var innerColor = "red";
        // 最内层函数swapColor，它的执行环境里又定义了一个变量tempColor
        function swapColors() {
            var tempColor = innerColor;
            innerColor = outerColor;
            outerColor = tempColor;
        }
        swapColors();
    }
    changeColor();
})();
```

下图为swapColors函数的作用域链结构，可以看到scope中有四个对象，第一个Local表示当前执行环境的变量对象，第二个是changeColor，第三个是外层匿名函数，最后一个是Global：
![scope chain example][2]

在本例中，当执行到swapColor函数内部时：
1. 先从作用域链的首端（即它自己的执行环境）开始找innerColor，找不到，于是从作用域链的下一个执行环境（即往外一层）中找，即changeColor的执行环境，找到了，执行下一条语句；
2. 又开始从作用域链首端查询outerColor，又没找到，跳到外一层执行环境，还是没有，再往外走一层，在匿名函数的执行环境中找到了outerColor，执行下一条语句；
3. 执行完毕后，当前执行环境从栈中弹出，回到changeColor的执行环境。依次类推，直到代码执行完毕。

**「Note」** 一个函数的执行环境由它的活动对象（activation object）表示。每个活动对象初始时都只包含一个arguments对象。

---
# 闭包又是什么？
了解了上面两个概念，闭包的定义就比较好懂了。**闭包是指有权访问到另一个函数作用域中的变量的函数。**

> Closures are functions that have access to variables from another function’s scope. 
> from *Professional JavaScript for Web Developers*

创建闭包的常见方式就是在一个函数内部创建另一个函数。

举个例子：
```javascript
function createCompareFunc(propertyName) {
    // 返回一个匿名函数，这个函数可以访问包含函数的参数propertyName
    return function(obj1, obj2) {
        var val1 = obj1[propertyName];
        var val2 = obj2[propertyName];
        if (val1 < val2 ) {
            return -1;
        } else if (val1 > val2) {
            return 1;
        } else {
            return 0;
        }
    }
}

var compareNames = createCompareFunc("name");
var result = compareNames({name: "Tom"}, {name: "Jerry"});
```

一般当某个函数执行完毕后，这个函数的局部活动对象会被销毁。然而闭包的情况却不太一样。

在上例中，调用createCompareFunc返回了一个匿名函数，这个匿名函数的作用域链中包含了createCompareFunc函数的活动对象和全局变量对象。因此createCompareFunc函数在执行完毕后，它的执行环境的作用域链被销毁了，但是它的活动对象并不会被销毁，因为匿名函数的作用域链中还保留对它的引用。直到匿名函数也被销毁后，createCompareFunc的活动对象才被销毁。

比如我们在获得result之后，解除对匿名函数的引用，以便释放内存：
```javascript
var compareNames = createCompareFunc("name"); // 创建
var result = compareNames({name: "Tom"}, {name: "Jerry"}); // 调用
compareNames = null; // 释放
```

**「Note」** 由于闭包会携带包含它的函数的作用域，所以会比一般函数占用更多的内存。所以**不要过度使用闭包**哦～最好**只在必要时使用**～

---
## 聊聊循环中的setTimeout

几乎只要是javascript笔面试，就会看到一道经典的setTimeout题目：
问下面这段代码控制台里会打印出来什么呢？会是0，1，2，3，4吗？
```javascript
for (var i = 0; i < 5; i++) {
    setTimeout(() => console.log(i), 1000);
}
```
当然并不会了，你只会看到5，5，5，5，5～

为什么会这样呢？认真想一下，在setTimeout中创建的每个匿名函数在读取i的值时，会去它的作用域链中查询，最终它们都会在全局变量对象找到i，也就是说它们引用的都是同一个变量。而闭包只能取得包含函数中任何变量的最后一个值。所以，打印出来的都是5啦～

那么怎么才能使得代码达到预期效果呢？我们可以通过再创建一个立即执行的匿名函数，由于参数是值传递的，所以每次调用会将i的当前值传给参数num，然后这个匿名函数的内部再创建一个访问num的闭包，这样就能达到预期了。
```javascript
for (var i = 0; i < 5; i++) {
    ((num) => {
        setTimeout(() => console.log(num), 1000);
    })(i);
}
```

上面这个做法相当于使用匿名函数仿造了一个块级作用域。将函数声明包含在一对圆括号中，其后紧跟一对圆括号，以立即调用这个函数。这种函数也被成为[IIFE][3](Immediately Invoked Function Expression)。：
```javascript
(function() {
    // 这里就是块级作用域
})();
```

当然了，这些操作es6之前的做法啦，在ES6之后，有支持块级作用域的let，这个问题就很好解决了，只需要把var换成let就好啦～
```javascript
for (let i = 0; i < 5; i++) {
    setTimeout(() => console.log(i), 1000);
}
```

---

* **参考资料**
* *Professional JavaScript for Web Developers, 3rd*
* [MDN - Closures][4]
* [What is a Closure?][5]


  [1]: https://stackoverflow.com/questions/336859/var-functionname-function-vs-function-functionname#
  [2]: http://static.zybuluo.com/JaneL/ubs2fqmhn2tr4px5w2pogukr/image.png
  [3]: https://developer.mozilla.org/en-US/docs/Glossary/IIFE
  [4]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures
  [5]: https://medium.com/javascript-scene/master-the-javascript-interview-what-is-a-closure-b2f0d2152b36