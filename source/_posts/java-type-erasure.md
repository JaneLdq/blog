---
title: Java泛型与类型擦除
date: 2019-06-16 08:36:24
categories: 技术笔记
tags: Java
---

之前我们聊到了Java泛型的定义和应用，本篇笔记就来介绍一下Java中实现泛型的机制——类型擦除。

类型擦除主要做了以下几件事：

* 将所有泛型的类型参数替换成它们的bounds，如果是参数是unbounded，那么就替换成Object。这样生成的二进制码中就只有一般的类、接口和方法了
* 在必要时插入强制类型转换来保证类型安全
* 生成桥接方法来保证多态性

实际上，通过类型擦除，在编译过后的字节码文件中就不存在泛型类了，它们都被替换为原生类型(Raw Type)，并在相应的地方插入了强制转换。因此，对于Java而言在运行期`ArrayList<Integer>`和`ArrayList<String>`就是同一个类。

可以说，Java中的泛型更像一颗语法糖，而基于类型擦除实现的泛型又被称为“伪泛型”。
<!--more-->

---
# 泛型类和泛型方法中的类型擦除
下面我们来看一些例子，更直观地感受一下类型擦除吧～
首先看个unbounded的例子：
```java
public class Box<T> {
    private T t;
    public Box(T t) {this.t = t;}
    public T get() { return t; }
}
```
上面的`Box<T>`中的T是unbounded的，所以编译器会将`Box<T>`中的T替换成`Object`。
我们可以调用如下命令来查看反编译字节码文件：
```
$ javap -c Box
```

得到的反编译结果如下所示，可以看到注释中对应的field的类型都为`Object`了：
```java
public class Box<T> {
  public Box(T);
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: aload_0
       5: aload_1
       6: putfield      #2                  // Field t:Ljava/lang/Object;
       9: return

  public T get();
    Code:
       0: aload_0
       1: getfield      #2                  // Field t:Ljava/lang/Object;
       4: areturn
}
```

再看一个bounded的例子：
```java
public class Box<T extends Comparable<T>> {
    private T t;
    public Box(T t) {this.t = t;}
    public T get() { return t; }
}
```
编译再反编译后就变成了下面这样，原来的`<T extends Comparable>`就变成了`Comparable`：
```java
public class Box<T extends java.lang.Comparable<T>> {
  public Box(T);
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: aload_0
       5: aload_1
       6: putfield      #2                  // Field t:Ljava/lang/Comparable;
       9: return

  public T get();
    Code:
       0: aload_0
       1: getfield      #2                  // Field t:Ljava/lang/Comparable;
       4: areturn
}
```

【Note】如果Bounds有多个，那么类型擦除会将类型参数替换为第一个bound，比如`<T extends Comparable & Serializable>`在编译过后就会变成`Comparable`，而后面的bounds编译器会在必要时插入强制转换。因此，因尽量将`tagging interface`(不包含方法的接口)放在后面。

对于泛型方法的处理，同理：
```java
public class Util {
    public static <T extends Number> int add(T a,  T b) {
        return a.intValue() + b.intValue();
    }
}
```
编译过后：
```java
public class Util {
  public Util();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static <T extends java.lang.Number> int add(T, T);
    Code:
       0: aload_0
       1: invokevirtual #2                  // Method java/lang/Number.intValue:()I
       4: aload_1
       5: invokevirtual #2                  // Method java/lang/Number.intValue:()I
       8: iadd
       9: ireturn
}
```

【Note】*值得注意的一点是，我们在反编译字节码的结果中可以看到方法的参数类型并不是原生类型，而是包括了参数化类型的信息。这一点是由Java虚拟机规范中引入的Signature属性决定的，它的作用就是存储一个方法在字节码层面的特征签名。从这一点中，我们也可以看出，所谓类型擦除只是从Code属性的字节码进行擦数，而元数据还是保存了泛型信息。因此，我们还可以通过反射手段取得参数化类型。*

---
# 类型擦除与方法重载
由于类型擦除的存在，我们在试图重载包含了泛型类参数的方法时可能会遇到问题。
举个例子：
```java
public static void print(List<String> list) {
    System.out.println("invoke print(List<String> list)");
}

public static void print(List<Integer> list) {
    System.out.println("invoke print(List<integer> list)");
}
```
上面的`print`方法的重载是不是看起来很正常，`List<String>`和`List<Integer>`是不同的参数类型。然而，这段代码是不能通过编译的。编译器会告诉你：
```java
Util.java:8: error: name clash: print(List<Integer>) and print(List<String>) have the same erasure
    public static void print(List<Integer> list) {
                       ^
1 error
```
因为类型擦除的原因，编译过后`List<String>`和`List<Integer>`的类型都变为原生类型`List`，这时两个方法的Signature就完全长一样了。ˊ_>ˋ

---
# 桥接方法与多态性
还是看代码说话，给定一个泛型类`Node`和它的子类`IntNode`：
```java
public class Node<T> {
    public T data;
    public Node(T data) { this.data = data; }
    public void setData(T data) { this.data = data; }
}

public class IntNode extends Node<Integer> {
    public IntNode(Integer data) { super(data); }
    public void setData(Integer data) {
        // do something elses
        super.setData(data);
    }
}
```
`Node`和`IntNode`编译之后的结果如下所示：
* `Node`类
```java
public class Node<T> {
  public T data;

  public Node(T);
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: aload_0
       5: aload_1
       6: putfield      #2                  // Field data:Ljava/lang/Object;
       9: return

  public void setData(T);
    Code:
       0: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #4                  // String Node.setData
       5: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       8: aload_0
       9: aload_1
      10: putfield      #2                  // Field data:Ljava/lang/Object;
      13: return
}
```
* `IntNode`类：
```java
public class IntNode extends Node<java.lang.Integer> {
  public IntNode(java.lang.Integer);
    Code:
       0: aload_0
       1: aload_1
       2: invokespecial #1                  // Method Node."<init>":(Ljava/lang/Object;)V
       5: return

  public void setData(java.lang.Integer);
    Code:
       0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #3                  // String IntNode.setData
       5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       8: aload_0
       9: aload_1
      10: invokespecial #5                  // Method Node.setData:(Ljava/lang/Object;)V
      13: return

  public void setData(java.lang.Object);
    Code:
       0: aload_0
       1: aload_1
       2: checkcast     #6                  // class java/lang/Integer
       5: invokevirtual #7                  // Method setData:(Ljava/lang/Integer;)V
       8: return
}
```
经过类型擦除后，`IntNode.setData(Integer data)`和`Node.setData(Object data)`由于参数不同变成了两个方法，也就是说，父类`Node.setData`方法并不会被Override，由此失去了多态性，这并不是我们希望看到的结果。
因此，为了解决这个问题，编译器会在`IntNode`中生成一个**桥接方法**。如上所示，在`IntNode`的反编译结果中包含了两个`setData`方法，其中有一个是`setData(java.lang.Object);`，这个方法就是由编译器生成的桥接方法，转换为代码看得更明白一点：
```java
public class IntNode extends Node<Integer>{
    // ...
    // 由编译器生成的桥接方法
    public void setData(Object data) {
        setData((Integer) data);
    }
}
```

由于编译器实际上是做了类型擦除和添加桥接方法两步，在我们实际运行下面这段代码时，在调用`node.setData("hello")`时就会报异常啦。
```java
IntNode intNode = new IntNode(19);
Node node = (IntNode)intNode;
node.setData("hello"); // 在这一步就会抛出错误了，java.lang.ClassCastException: java.lang.String cannot be cast to java.lang.Integer
```

---

PS：关于Java泛型和类型擦除另外还有一些值得注意的点和容易踩的坑，待我消化好了再见～

---

**参考资料**
* [The Java Tutorial - Generics][1]
* *Core Java Volumn 1 - Fundamentals*
* *深入理解Java虚拟机（第二版）*

 [1]: https://docs.oracle.com/javase/tutorial/java/generics/erasure.html