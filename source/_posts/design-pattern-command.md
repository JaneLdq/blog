---
title: 设计模式之命令模式
date: 2018-08-01 14:35:35
categories: 设计模式
tags: 命令模式
---

最近在看Hystrix的应用，整个HystrixCommand的概念都是基于Command Pattern（命令模式），正好对我来说是一个比较陌生的设计模式，借此机会深入了解一下。

# 命令模式的定义和动机

* 定义
> Encapsulate a request as an object, thereby letting you parameterize clients with
different requests, queue or log requests, and support undoable operations.
> *将一个请求封装成对象，使得你可以为客户定制化不同的请求，把请求纳入队列或记录请求日志，以及支持可撤销的操作。*

<!--more-->

* 动机
> Sometimes it's necessary to issue requests to objects without knowing anything about the operation being requested or the receiver of the request.
> *在软件设计中，我们经常需要向某些对象发送请求，但是并不知道请求的接收者是谁，也不知道被请求的操作是哪个。我们只需在程序运行时指定具体的请求接收者即可，此时，可以使用命令模式来进行设计，使得请求发送者与请求接收者消除彼此之间的耦合，让对象之间的调用关系更加灵活。*

# 命令模式的结构
命令模式包含如下几类角色：

* **Command** - 声明执行操作的接口
* **ConcreteCommand** - 将一个接收者绑定到一个动作，调用接收者相应的操作已实现Execute
* **Invoker** - 要求命令执行这个请求
* **Receiver** - 知道如何实施某个请求的具体操作
* **Client** - 创建ConcreteCommand，并为它设定接收者

## 类图
![Command Pattern][1]

## 时序图
![commandpattern-seq][2]

命令模式的一大特点就是**对请求的发送者和接收者进行解耦，使得发送者和接收者之间没有直接引用关系，发送请求的一方只需要知道如何发送请求，而不必知道请求如何完成。**

---
# 举个栗子更好懂
看了一堆概念，并不知道在说些什么。

我们还是结合书中给出的例子来看。
假设我们现在要设计一个通用菜单类库，它会显示一个下拉菜单，当用户点击每个菜单按钮的时候一定会触发某个操作，但是具体的操作要做什么是由使用菜单的应用程序来定义的。作为通用类库，我们只需要实现“点击触发请求”这个工作流就可以了。

这就是一个适用命令模式的典型场景了。

下面几段代码简单模拟了一个类似的场景（稍有出入，简化了一下菜单按钮被点击的过程，这里就用一个list简单粗暴的遍历意思一下啦~）

## 类图
![Command Pattern Demo][3]

## 代码
* Command
```java
public interface Command {

    void execute();

}
```

* CopyCommand & PastCommand
```java
public class CopyCommand implements Command {

    private Document doc;

    public CopyCommand(Document doc) {
        this.doc = doc;
    }

    @Override
    public void execute() {
        this.doc.copy();
    }

}

public class PasteCommand implements Command {

    private Document doc;

    public PasteCommand(Document doc) {
        this.doc = doc;
    }

    @Override
    public void execute() {
        this.doc.paste();
    }

}
```

* Document
```java
public class Document {

    private String name;

    public Document(String name) {
        this.name = name;
    }

    public void copy() {
        System.out.println(this.name + " Copied!");
    }

    public void paste() {
        System.out.println(this.name + " Pasted!");
    }

}
```

* DocOperationExecutor
```java
public class DocOperationExecutor {

    private final List<Command> commands = new ArrayList<>();

    public void executeCommand(Command command) {
        commands.add(command);
        command.execute();
    }

}
```

* Client
```java
public class Client {

    public static void main(String[] args) {
        Document doc = new Document("Demo Doc");
        DocOperationExecutor executor = new DocOperationExecutor();
        executor.executeCommand(new CopyCommand(doc));
        executor.executeCommand(new PasteCommand(doc));
    }

}
```

---
# 命令模式的优缺点
* 优点
    * 解耦了请求发送者和请求接受者之间的依赖
    * 新增命令很容易，不会影响已有的命令
    * 可以将命令进行组合变成组合命令（宏？）
    * 可以比较方便的实现对请求的撤销和重做


* 缺点
    * 可能会导致大量的ConcreteCommand类 =皿=

适用场景嘛，能充分发挥这个模式的优点的场景都是合适的啦~

---
# 参考资料
* GoF 《设计模式》
* [Command Pattern Wiki][4]
* [Graphic Design Patterns - Command][5]
* [The Command Pattern in Java][6]


  [1]: http://static.zybuluo.com/JaneL/zbvh1te6ufe0dzupo84vg6b2/Command%20Pattern%20%282%29.png
  [2]: http://static.zybuluo.com/JaneL/plagu0fv5qvmy3j0xf5e2oc9/commandpattern-seq.png
  [3]: http://static.zybuluo.com/JaneL/cbk1ndcknkhqo54cxstoz4xb/Command%20Pattern%20Demo.png
  [4]: https://en.wikipedia.org/wiki/Command_pattern
  [5]: http://design-patterns.readthedocs.io/zh_CN/latest/behavioral_patterns/command.html
  [6]: http://www.baeldung.com/java-command-pattern