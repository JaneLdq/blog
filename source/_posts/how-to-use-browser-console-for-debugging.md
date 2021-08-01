---
title: 解锁浏览器里调戏八阿哥新姿势
date: 2018-07-19 14:01:53
categories: 技术笔记
tags: JavaScript
---

作为一个平时只会用`console.log`找八阿哥的伪前端，刚刚看到一篇文章介绍在浏览器控制台debug时不那么常用的几个方法，真的很简单，但是看起来就很有用的样子。

关键是，我真的，从来都没有这么用过（捂脸逃

敲个笔记学习一下~

<!--more-->

---

# 轻松输出多变量
```javascript
const a = 123;
const b = `abc`;
const c = {
    name: 'foo',
    age: 16
};
console.log('This is a log: ', a, b, c);
```

还可以这样：

```javascript
console.log("Number: %d, \nString: %s, \nObject: %o", a, b, c);
```

像上面这两种写法呢，就能完美避开`[Object Object]`这种并没有什么鬼用的信息啦~

![console-log][1]

---
# Log级别记得分呀
根据不同的级别打log应该算在代码习惯里了，在正式的产品中要比较注意这一点才好。但平时真的很容易随便写，好习惯要培养~

```javascript
console.debug("Debug Message");
console.info("Info Message");
console.log("Log Message");
console.warn("Warning Message");
console.error("Error Message");
```

以上四个log level在Chrome的控制台里分别对应Verbose, Info, Warnings, Errors。大家最常用的console.log也被归在info级别。

![console-log-level][2]

---
# Group之后更清晰
除了单挑，还可以组队输出哦~
```javascript
console.group("Group A");
console.log("Message 1");
console.log("Message 2");
console.groupEnd();
```

Group操作还可以嵌套玩耍，组内还可以再分小分队~
```javascript
console.group("Peppa Pig Yeah!");
console.log("Welcome!");
console.group("Playgroup");
console.log("Peppa is here!");
console.log("Suzy is here!");
console.groupEnd();
console.log("Where is George?");
console.groupEnd();
```

![console-group][3]

---
# Time计时更方便
以前如果要在控制台检测性能一般都是如下操作：
```javascript
const start = Date.now();
// do something
console.log("Task A " + (Date.now() - start) + 'ms');
```

但是，其实有更简洁的方法 —— 用`console.time`：
```javascript
console.time("Task A");
for(let i = 0; i < 1000000; i++) { Math.sqrt(i) }
console.timeEnd("Task A");
```
![console-time][4]

---
Console对象其实还有一些其他的更加不常用的方法，可能在遇到某些及其特别的情况会用到，感兴趣的朋友可以到这里[MDN web docs - Console][5]看看啦~

# 参考文章
[How to go beyond console.log and get the most out of your browser’s debugging console][6]


  [1]: http://static.zybuluo.com/JaneL/cqzm62yw5caibij2dlcmzylo/image.png
  [2]: http://static.zybuluo.com/JaneL/7fc2kwpe2s8bxkd0ctq7fpie/image.png
  [3]: http://static.zybuluo.com/JaneL/oe2gsenz7n4aebxl2j862ohh/image.png
  [4]: http://static.zybuluo.com/JaneL/6q3l6rw5suubbvw382ynk59b/image.png
  [5]: https://developer.mozilla.org/en-US/docs/Web/API/Console
  [6]: https://medium.freecodecamp.org/how-to-go-beyond-console-log-and-get-the-most-out-of-your-browsers-debugging-console-e185256a1115