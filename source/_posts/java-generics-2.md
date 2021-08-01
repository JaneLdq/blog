---
title: Java泛型与数组
date: 2019-06-22 07:35:23
categories: 技术笔记
tags: Java
---

作为Java泛型系列的最后一篇笔记，我们来看看泛型(Generics)与数组(Array)和可变长参数(Varargs)之间的“恩怨情仇”～

关于三者，我们可以先列出如下事实：
* Java中不可以创建泛型数组
* Java可变长参数底层采用数组实现
* Java可变长参数可以为泛型类型

咦，好像有哪里怪怪的？

要弄清楚三者的关系，我们还得从泛型与数组的关系入手～
<!--more-->
---
# “相性不合”的数组与泛型
Java中不能创建泛型数组的原因主要在于泛型与数组的两组特性差异：

## Covariant VS. Invariant
先来看几个定义：
* **Covariance(协变)** - 能在使用父类型的场景中改用子类型的被称为协变。
* **Contravariance(逆变)** - 能在使用子类型的场景中改用父类型的被称为逆变。
* **Invariance(不变)** - 不能做到协变或逆变的就被称为不变。

那么数组和泛型分属于什么呢？
* 数组是Covariant - 如果`Sub`是`Super`的子类，那么`Sub[]`也是`Super[]`的子类。
* 泛型是Invariant - 回想一下一般泛型的继承关系，对于任意两个类型`A`和`B`，无论它们之间有没有继承关系，`List<A>`都既不是`List<B>`的父类也不会是`List<B>`的子类。

举个例子：
```java
// 可以通过编译，但是运行时会报错
Object[] arr = new Long[1];
arr[0] = "This will cause runtime error!"; // throws java.lang.ArrayStoreException

// 不能通过编译
List<Object> list = new ArrayList<Long>(); // incompatible types: ArrayList<Long> cannot be converted to List<Object>
list.add("This won't compile!");
```

【Note】*什么场景下会看到逆变(Contravariance)呢？思考一下泛型中通配符使用时的继承关系，举个例子，`List<? extends T>`是协变的，而`List<? super T>`就是逆变的了。你可能会想，既然`List<? extends T>`是协变的，那为什么不能创建一个`List<? extends T>[]`呢？因为还有类型擦除的存在呀～*

---
## Reifiable VS. Non-reifiable
* **Reifiable(具体化的)** - 一个Reifiable的类型，在运行时也可以获取它的完整的类型信息。**数组就是reifiable类型**
    *  primitives, non-generic types, raw types, and invocations of unbound wildcards都是reifiable类型
* **Non-reifiable(非具体化的)** - Non-reifiable类型的运行时描述(representation)包含的信息要少于编译时的描述。**泛型是Non-reifiable，类型擦除抹掉了部分类型信息**
    * 唯一的reifiable的类型参数如上面提到的就只有无界通配符`?`，比如创建`List<?>`和`Map<?, ?>`的数组就是合法的。


由于以上两组“相性不合”，数组和泛型就不能很愉快地在一起玩耍了。
我们可以假设一下允许创建泛型数组的场景，假设有如下这段代码：
```java
List<String>[] stringLists = new List<String>[1]; // 虽然这行代码并不能真实地通过编译，现在请你动用你聪明的小脑瓜假设一下它通过编译的场景
List<Integer> intList = Arrays.asList(12); // (1)
Object[] objects = stringLists; // (2)
objects[0] = intList; // (3)
String s = stringLists[0].get(0); // (4)

```

1. 我们先创建了一个Integer列表准备搞事情
2. 这行赋值是合法的，因为数组是Covariant，而`List<String>`是`Object`的子类没毛病
3. 我们能成功将`intList`塞到`objects`中，因为类型擦数后，运行时`objects`指向的`stringLists`的类型为`List[]`，`intList`的类型为`List`
4. 这里就要出错啦，因为编译器会将拿出来的元素强转为String，这时就会抛出`ClassCastException`了，因为实际的元素类型为`Integer`。

以上的示例很好地说明了什么叫——“强扭的瓜不甜”。
所以，为了避免不愉快地合作，Java中创建泛型数组时不合法的～（特殊的`?`除外～）

所以，如果当你产生创建泛型数组的念头时，最好的方式是用List替换它，虽然这样做会丧失一部分性能和简洁性，但能保证类型安全和更高的互操作性。

---
## 如何绕过创建泛型数组的编译错误来使用泛型数组呢？
虽然Java不允许创建泛型数组，我们也不推荐使用泛型数组。但是，在某些场景下，我们真的非常需要这样的数组结构来提供更好的性能（HashMap）或者解决其他问题，像的支持，那怎么办呢？

我们通过一个例子来探讨如何解决：
```java
public class Stack<E> {
    private E[] elements;
    private int size;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;
    @SuppressWarnings("unchecked")
    public Stack() {
        elements = (E[]) new Object[DEFAULT_INITIAL_CAPACITY];
    }
    public void push(E e) { 
        ensureCapacity(); 
        elements[size++] = e;
    }
}

```
如上所示，我们定义了一个泛型类`Stack<T>`，在构造函数中创建了一个`Object[]`，然后把它强制转换成了`E[]`。因为我们能够保证所有存入这个数组的元素都是`E`类型的实例(`elements`为私有变量，并且不能被外界获取到，而且元素只能通过`push`方法存)，所以可以在构造函数上加上`SuppressWarnings("unchecked")`注解，这样其他使用这个类的用户就会再遇到unchecked warning了。

另一种方式就是把将`E[]`改为`Object[]`：
```java
public class Stack<E> {
    private Object[] elements;
    private int size;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;
    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }
    public E pop() {
        if (size == 0)
            throw new EmptyStackException();
        @SuppressWarnings("unchecked")
        E result = (E) elements[--size];
        elements[size] = null;
        return result;
    }
}
```
如上所示，我们在构造函数中可以直接创建`Object[]`，然后在取数组元素时进行强制转换。可以看到我们在取元素的那一行代码上添加了`@SuppressWarnings("unchecked")`注解，因为只能从`push`方法向数组中存值，而push方法能保证传入参数都为`E`类型，由此我们能保证类型安全。同时这行注解作用的范围也遵循了最小范围原则，这样才能避免我们把其他潜在的unchecked warning一起suppress掉。

总结一下，有两种解决不能创建泛型数组的方法：
1. **数组声明为泛型数组，比如`E[]`，然后创建一个`Object[]`，然后把它强制转换为`E[]`**。这样编译器不会报错，但是会抛出warning，因为通常不是类型安全的操作。这种方式的优缺点如下
    * 可读性更好 - 数组声明为`E[]`，非常清晰的表明了它所包含的元素类型
    * 简洁性更高 - 只需要在创建数组时进行一次强制转换
    * 引入了**堆污染(Heap Pollution)** - 当一个泛型变量指向的对象并不是它所定义的类型时，就是堆污染。

> Heap pollution occurs when a variable of a parameterized type refers to an object that is not of that type. It can cause the compiler’s automatically generated
casts to fail, violating the fundamental guarantee of the generic type system.

2. **直接将数组声明为`Object[]`，然后在每次从数组中读取数据时强制转换为`E`**。这种方法的繁琐之处就在就是每次从数组中取元素时都要做一次强制转换。

---
# 泛型可变长参数
由于Java中的可变长参数的底层实现实际上是用了一个数组来保存的，而可变长参数又允许使用泛型，所以在使用可变长参数时，我们就无法避免数组与泛型之间这种非常别扭的相处模式了。

一个最容易遇到的问题就是我们在声明泛型可变长参数时，编译器总会抛warning。比如我们把上面那段代码拿过来声明为一个使用可变长参数的方法：
```java
public static void dangerous(List<String>... stringLists) {
    List<Integer> intList = Arrays.asList(12);
    Object[] objects = stringLists;
    objects[0] = intList;
    String s = stringLists[0].get(0);
}
```
编译器要报警啦：
```java
 warning: [unchecked] Possible heap pollution from parameterized vararg type List<String>
    public static void dangerous(List<String>... stringLists) 
```

既然在可变长参数里使用泛型这么危险，为什么Java不禁用呢？因为在某些场景下，在可变长参数重使用泛型时很方便的，只要作为开发者的你能保证方法内部的实现是类型安全的。比如我们上面用到的工具类方法`Arrays.asList`实际上就采用了泛型可变长参数。实际上，很多Collections的方法都用到了泛型可变长参数。

考虑到了使用可变长泛型参数的合理性，为了使开发者能更愉快地编码，Java 7引入了`@SafeVarargs`注解。如果一个方法加上了`@SafeVarargs`，那么编译器就默认方法的作者能保证这个方法即使使用了泛型可变长参数也能保证类型安全，它不再抛出相关的warning。

**那么，如何保证一个加了`@SafeVarargs`的方法是安全的呢？**
* 这个方法没有存任何东西到泛型可变长参数数组中
* 这个方法没有对外暴露这个数组的引用（比如作为返回值传递给外部）
    * 把一个泛型的可变长参数数组传递给另一个方法是不安全的，两种情况除外：
        * 接收数组的方法有`@SafeVarargs`注解
        * 接收数组的方法是一个没有采用可变长参数的方法，that merely computes some function of the contents of the array. （翻译无能）

综上，记得对每一个是用了泛型可变长参数的方法加上`@SafeVarargs`哦～

---

最后，因为自己写泛型类和泛型方法的机会并不是很多，用的也是比较基础的，这组笔记中的例子基本都来自Java官方的tutorial和《Effective Java》这本书，毕竟自己凭空想场景还是有点难 / \_ \\

我想关于Java泛型的笔记到这里就结束啦～

Java泛型系列：

* {% post_link java-generics [浅析Java泛型] %}
* {% post_link java-type-erasure [Java泛型与类型擦除] %}

---

**参考资料**
* *Effective Java (3rd Edition) - Chapter 5 Generics*



























