---
title: JavaScript 中的单例模式
date: 2017-03-24 20:49:17
categories: 设计模式
tags: 
- JavaScript
- 单例模式
---
之前也学过一些设计模式，很久不用也忘得差不多了，最近为了做毕设准备复习一下设计模式，用的语言是JavaScript，正好把二者结合起来，一边看*Pro JavaScript Design Patterns*、*Design Patterns*， 一边写笔记。

又挖新坑啦，果真是要用的时候才想起来学啊_(:з」∠)_

<!--more-->
---

## 单例模式（Singleton）
单例模式的意图在于确保一个类有且仅有一个实例，并且为它提供一个全局访问点。

由于JavaScript的灵活性，单例模式在JS中的实现有多种形式，接下来我们逐个介绍。

### 1. Using Object Literal（对象字面量）
对象字面量是最简单的一种单例实现方式，使用它就可以把一批有关联的方法和属性组织到一起，并且它提供了访问接口。

```javascript
var Singleton = {
    name: 'Singleton',
    otherAttr: false,
    method1: function() {},
    method2: function() {}
}
```

在面向对象的语言中，类是可以扩展但不能被修改的。而使用这种方式定义的单例是十分不严谨的，它不是可以被实例化的类，它还可以被随意修改。在ES6之前，JavaScript中没有显式支持class这个关键字，因此类的概念也不是那么明确，那么我们在这里就把单例模式的定义拓宽一些：**单例是一个对象，用来划分命名空间并且负责将一组相关联的方法和属性组织起来，并为其提供一个全局访问的入口**。

对象字面量只是创建单例的方法之一，而且，如果一个对象字面量只是用于容纳数据或者只是用于模仿关联数组（associative aary），那就显然不能称之为单例。如果它是一组县关联的方法和属性的集合，那么它有可能是单例。这之间的差别一般在于设计者的意图。

### 2.Using Function Wrapper（包装函数）
看下面这段代码：

```javascript
// 在这里使用DPNS作为命名空间，为什么要这样做在后面的小节有提到
var DPNS = {}; // Design Pattern Namespace
DPNS.Singleton = (function() {
    return {
        name: 'Singleton',
        otherAttr: false,
        method1: function() {},
        method2: function() {}
    }
})();
```

这段代码调用了一个立即执行的匿名函数，返回值是一个对象字面量，赋值给了变量Singleton。这段代码的结果跟直接使用字面常量的方式没有区别，那么为什么要这样写呢？

再来看一段代码：

```javascript
DPNS.Singleton = (function() {
    var privateAttr1 = 1;
    function privateFunc() {
        console.log('I am private'）;
    };
    return {
        name: 'Singleton',
        publicAttr1: false,
        publicFunc1: function() {
            if (privateAttr1 % 2) {
                privateFunc();
            }
        },
        publicFunc2: function() {
            privateAttr1++;
        }
    }
})();
```

从上面代码中就可以看出包装函数(function wrapper)的作用就是**创建用来添加私有变量的闭包**。

*在函数定义外再套上一堆圆括号算是一些程序猿的习惯，以表示该匿名函数会在声明后立即执行，这对于创建函数体很长的单例时很有用，因为只要看到第一眼的左括号就知道这个函数是用于创建闭包的。（之前看到括号花括号扎堆就头大，现在知道原因后只想说：前辈们真的很机智啊！豁然开朗再也不怕啦~*

PS：通常我们说私有变量的使用要谨慎，因为每个实例都持有一个新副本，是很消耗内存的。但是！我们现在讨论的是单例模式，本来就只会实例化一次，所以完全不用担心私有变量的复制带来的内存浪费，就是这么自信！


### 3. Lazy Instantiation（惰性实例化）
以上提到的两种方式，都是在脚本加载时就完成了创建。但是如果单例实例化的资源开销、配置开销很大，例如要加载大量数据的单例。可以考虑类似懒加载的思想，采用Lazy Instantiation（惰性实例化）。即在真正调用到单例时或者空闲时再进行实例化，这样可以避免在脚本刚刚加载时跟其他初始化过程抢资源。

对Lazy Instantiation单例的访问与直接创建的不同之处在于它要借助静态方法，看代码：

```javascript
DPNS.LazySingleton = (function() {
    var instance;
    function constructor() {
        var privateAttr = false;
        function privateFunc() {};
        return {
            publicAttr: true,
            publicFunc: function() {}
        }
    };
    return {
        getInstance: function() {
            // 写到这里终于感觉跟以前用Java写的单例模式长得像起来了
            if (!instance) {
                instance = constructor();
            }
            return instance;
        }
    }
})();
// 调用方式
DPNS.LazySingleton.getInstance().publicFunc();
```

Lazy Instantiation的缺点就在于增加了复杂度，代码不直观，所以要用的话要写认真写文档，最好注释为什么要这样做。

### 4. Branching（分支）
如果在生成单例的时候，要根据当时的运行环境对实例化过程进行动态设置，可以考虑使用分支技术。一个比较常见的例子就是在浏览器环境下创建XHR对象，写过AJAX的小伙伴都知道IE这朵奇葩总是要做特例处理的。每次创建XHR对象都要在当前浏览器下重复判断的效率明显不如在脚本加载时一次性针确定这对特定浏览器的代码。
解释看多了容易绕，还是看代码更直观，来吧：

```javascript
// 简单例子
DPNS.BranchSingleton = (function(){
    var objectA = {
        method1: function() { ... },
        method2: function() { ... }
    };
    var objectB = {
        method1: function() { ... },
        method2: function() { ...}
    };
    return (someCondition) ? objectA : objectB;
})();
// XHR创建例子
DPNS.SimpleXhrFactory = (function(){
    var standard = {
        createXhrObj: function() { return new XMLHttpRequest(); }
    };
    var activeXNew = {
        createXhrObj: function() { return new ActiveXObject('Msxml2.XMLHTTP'); }
    };
    var activeXOld = {
        createXhrObj: function() { return new ActiveXObject('Microsoft.XMLHTTP'); }
    };
    var testObj;
    try {
        testObj = standard.createXhrObj();
        return standard; // return this if no error was thrown
    } catch(e) {
        try {
            testObj = activeXNew.createXhrObj();
            return activeXNew; // return this if no error was thrown
        } catch(e) {
            try {
                testObj = activeXOld.createXhrObj();
                return activeXOld; // return this if no error was thrown
            } catch(e) {
                throw new Error('No XHR object found in this environment');
            }
        }
    }
})();
// 使用SimpleXhrFactory.createXhrObj()就可以得到特定运行环境下的XHR对象了，上面那一段复杂的判断只会在初始化实例时调用，之后每次生成对象都不需要判断了
```

分支技术利弊权衡：使用分支缩短了计算时间（判断使用哪个对象的代码只会在初始化时执行一次），但是它也占用了更多内存（多个分支对象会被创建并保存在内存中）

### 5.ES6引入class之后的单例模式的实现
```javascript
let instance;
const Singleton = class {
	constructor() {
	    console.log('I am Singleton!');
		if (!instance) {
			instance = this;
		}
		return instance;
	}
	publicFunc() {
		console.log('hello, world!');
	}
}
var obj1 = new Singleton(); // 'I am Singleton!'
var obj2 = new Singleton(); // 'I am Singleton!'
console.log(obj1 === obj2); // true
```
上面这种方式使用了全局变量，而且在两次实例化时都调用了Singleton的构造函数，只是第二次调用返回的仍然是第一次初始化的实例罢了，改进之后我们使用静态方法getInstance()获取单例，代码如下：

```javascript
const Singleton = class {
	constructor() {
		console.log('I am Singleton!');
	}
	publicFunc() {
		console.log('hello, world!');
	}
	static getInstance() {
	    console.log(this);
		if (!this.instance) {
			this.instance = new Singleton();
		}
		return this.instance;
	}
}
let obj1 = Singleton.getInstance(); //[Function: Singleton] 'I am Singleton!'
let obj2 = Singleton.getInstance(); // {[Function: Singleton], instance: Singleton{}}
let obj3 = Singleton.getInstance(); // {[Function: Singleton], instance: Singleton{}}
console.log(obj1 === obj2); // true
```


## 单例模式的适用场景
单例模式的适用场景十分广泛，结合不同的实现形式有不同的作用。
1. 为代码提供命名空间，减少全局变量的书目
**一个单例对象 = 对象本身（包含属性和方法）+ 一个可以访问它的变量（这个变量一般是全局的）**

* 命名空间
由于JavaScript的灵活性，命名空间是js编程中必须重视的一点。在JS中几乎一切都是可以被重写的，因此很容易在无意中就抹掉了某个变量、函数甚至整个类。
为了防止无意改写变量的最佳方式之一，就是**使用单例模式来划分命名空间**。

2. 增强模块性，把自己的代码组织在一个全局变量名下，放在单一位置，便于维护
3. 大型复杂的项目，使用Lazy Instantiation优化性能
4. 等等

## 单例模式的优缺点
### 优点
1. 优化代码结构：相关的方法和属性集中在一个地方，且只会实例化一次。简化了代码的调试和维护
2. 使用自描述的命名空间可增强代码的可读性（所以命名空间也要起得有意义哦）
3. 可防止代码被重写
4. 保持全局命名空间的整洁
5. 单例模式的一些高级变体可以在开发后期用于脚本的优化，提升性能（比如惰性实例化和分支）

### 缺点
单例模式使用单点访问，可能导致模块间的强耦合，从而不利于单元测试。无法单独测试一个调用了来自单例的方法的类，而只能把它与那个单例作为一个单元一起测试。

单例最好用于划分命名空间等耦合不会构成严重问题的情形。其他情况可以使用其他设计模式，例如分支技术就可以采用工厂模式代替呀~

那么下一篇就介绍工厂模式吧~

<del>偷偷告诉你，不会有下一篇了（这是一条来自三年后的吐槽～</del>
