---
title: JS中的递归函数怎么写？
date: 2018-12-19 20:55:15
categories: 技术笔记
tags: JavaScript
---

前面聊闭包的时候提到了不少JavaScript函数表达式的用法。这里补充一个，就是借助函数表达式构建递归函数，因为有一丢丢小坑值得注意一下。

我们先来看一个大家都会写的递归函数：
```javascript
function factorial(num) {
    if (num <= 1) {
        return 1;
    } else {
        return num * factorial(num - 1);
    }
}
```
看起来这是一个很普通的递归函数，普通的递归函数普通地使用是完全没有问题的。
然而，如果有个人如此这般操作一波：
<!--more-->
```javascript
var anotherFactorial = factorial;
factorial = "I Just want to give it another value";
anotherFactorial(3); // 报错啦~
```
这样调用会报错的，因为anotherFactorial内部还是会去调factorial，然而factorial现在被赋给其他值了。
怎么才能避免这种可能出现的充满恶趣味的操作呢？请使用**Named Function Expression**~

```javascript
var factorial = (function f(num) {
    if (num <= 1) {
        return 1;
    } else {
        return num * f(num - 1);
    }
});
```

**「温故知新」**：更多关于JavaScript中的函数声明(Function Declaration)、匿名函数表达式(Anonymous Function Expression)和命名函数表达式(Named Function Expression)的内容，在Stack Overflow上有一个回答挺全面的问题[var functionName = function() {} vs function functionName() {}?][1]*

**「考古时间」**：关于递归函数的实现，其实还有一种**被弃用**的方式——使用`arguments.callee`：
```javascript
function factorial(num) {
    if (num <= 1) {
        return 1;
    } else {
        return num * arguments.callee(num - 1);
    }
}
```
一方面`arguments.callee`在strict模式下是禁用的，另一方面这种方式因为要访问`arguments`对象，每次访问都要创建一个新的，代价高昂，影响性能。所以现在已经不推荐使用啦~

* **参考资料**
* *Professional JavaScript for Web Developer*

