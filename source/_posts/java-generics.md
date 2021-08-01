---
title: 浅析Java泛型
date: 2019-06-10 20:16:45
categories: 技术笔记
tags: Java
---

围观面试，正好有聊到Java泛型，自己的记忆也有点模糊，于是翻出了很久之前的零散笔记，重新整理了一波。

---

在引入泛型之前，在Java中使用Collections是一种非常容易出错的操作。举个例子：

```java
List numbers = new ArrayList();
numbers.add(new Integer(10));
numbers.add("foo");    // 在不使用泛型的情况下，List中每个元素的类型都是Object，这意味着你可以往里面扔任何类型的对象
Integer sum = 0;
for (int i = 0; i < numbers.size(); i++) {
    Integer n = (Integer) numbers.get(i); // 必须使用强制转换将元素转换为我们想要的类型
    sum++;
}
System.out.println(sum);
```

那么问题来了，上面这段代码在编译时是不会报错的，然而，因为我们在List中插入了`String`类型，而在循环中我们默认每个元素都应该为`Integer`类型并做了强制转换，在运行时，程序就会抛出`java.lang.ClassCastException`异常。

我们都知道，错误越早发现，修复所需的代价越小，能在编译时发现错误就不要让它潜伏到运行时；而且，在没有泛型时，但凡使用Collections都逃不开强制转换，这也是一件非常痛苦的事情。

所幸，Java在5.0版本就引入了泛型。
<!--more-->
简单总结一下，泛型的出现，主要为了如下几个目的：

* 在编译时可以进行更严格的类型检查
* 消除强制转换，编译器会自动进行强转
* 用于实现更通用的算法，例如Collections的实现

那么接下来，我们就一起来看看泛型的定义和使用吧～

---
# 什么是泛型
泛型的本质是参数化类型(Parameterized Type)的应用，也就是说所操作的数据类型被指定为一个参数。这种参数类型可以用在类、接口和方法的创建中，分别称为泛型类、泛型接口和泛型方法。

我们就先从泛型类说起。

## 泛型类
泛型，又被称为参数化类型，相比于一般的非泛型类，它具有更多的通用性和灵活性，允许我们在声明时再传入一些参数进行定制。

想必现在在大家的日常开发中都没少接触`List`这样的集合类，放在这个语境下，泛型就非常好理解了。相比于一个大杂烩的List，我们通常更希望拥有的是只包含某个类型的List，那为什么不针对每个类型分别实现自己的List呢？比如来一个IntegerList，再来一个StringList，那以后说不定还会有AppleList，OrangeList...而且，我们对这些List实现的操作都是一样的，Integer，String，Apple，Orange只是表明操作针对的对象类型不同而已。那我们为什么不能把它们作为List的一个参数，在声明时设置一下就好了呢？这样我想要什么类型的List都很方便，不需要自己再单独封装或者用风险很高的强制转换。

泛型就是这么个意思。


泛型类/接口的使用也非常简单，还是以`java.util.List`为例：
```java
List<Integer> numbers = new ArrayList<>();
```
跟开头的那段代码相比，这里的List的声明多了一些尖括号，`<>`里的值被称为**类型参数**，类型参数的值一般都为Class或Interface类型。

`List<Integer>`的意思就是*这是一个只能放Integer的List，别的类型一概不收*。有了这个声明，编译器就能在编译时进行类型检查，如果我们强行往里塞`String`，编译器就会报错啦。

值得注意的一点是，后面的`new ArrayList<>()`的`<>`不需要再填一次类型参数了，编译器能通过上下文完成**类型推导（type inference），编译器可以通过检查泛型类声明中的类型参数或泛型方法的参数的类型来计算类型参数的值**。
空的尖括号对`<>`被称为**Diamond**。（小“钻石”，还挺可爱的～）

那么，如何定义泛型类或接口呢？很简单，以`java.util.List`为例，它长得大致如下：

```java
public interface List<E> {
    boolean add(E e);
    Iterator<E> iterator();
}
```

每个泛型类型通过在名字后面加一对`<>`来定义了一组参数化的类型，比如List是这个泛型接口的名字，后面的`<>`里面包含一个类型参数`E`，E只是一个代号，跟一般的方法声明中的形参是一个意思，你可以把它换成T，V，K，都随意。

类型参数可以有一个或多个，比如，我们可以再看个`java.util.Map`的定义：

```java
public interface Map<K,V> {
    Set<K> keySet();
    Collection<V> values()；
}
```
那么在Map泛型接口定义中就包含了两个类型参数，`K`用于指定Key的类型，`V`用于指定Value的类型。那么我们在声明一个Map类时通常会这样定义：
```java
Map<String, Integer> myMap = new HashMap<>();
```
这个Map的键都是`String`类型，对应的值为`Integer`类型，你若是想往里塞其他类型的键值对，编译器不会放过你的。

那么问题来了，既然Java中List接口已经时泛型接口了，那么为什么我在开头写的那段不带类型参数的代码还能编译通过呢？直接使用List为什么还是合法的呢？这里就要提到Java引入泛型之后留下的一个大坑——**泛型类的原生类型(Raw Type)**了。

---
### 原生类型 Raw Types
```java
List raw = new ArrayList();

List<Integer> foo = new ArrayList<>();
List bar  = foo; // this is ok

List<Integer> baz = raw; // warning: unchecked conversion

```
像上面这样写，List就是 List<E> 的一个原生类型(raw type)，即不带任何实际类型参数。
值得注意的是，**非泛型类或接口并不是谁的原生类型，原生类型这个概念只是针对泛型类/接口才存在的**。
如上所示，若将参数化的泛型类赋值给它的原生类型是没问题的，反过来操作则会有警告，因为这意味着在运行时很可能会发生错误。

**使用原生类型就意味着放弃了泛型所提供的安全性和类型检查等优势，所以要在新代码中要避免使用它们**。

既然原生类型这么不好用，那么为什么Java的新版本还要留着它呢？答案显而易见，就是为了与引入泛型之前的遗留代码兼容。

**🌟提问：原生类型`List`和参数化的`List<Object>`有什么区别呢？**
*首先，使用原生类型`List`就失去了泛型提供的类型检查和隐式强转的好处；其次，这里还涉及到泛型的继承，举个例子，`List<String>`可以赋值给`List`，而不能赋值给`List<Object>`，因为`List<String>`是`List`的子类却不是`List<Object>`的子类。关于泛型的继承关系，我们接下来还会进一步介绍。*

---
## 泛型方法
除了定义整个泛型之外，我们也可以只针对某个方法设置它的类型参数，这类方法就被称为泛型方法。
举个例子，比如我们想写一个工具类方法，把某个数组中的所有元素添加到一个Collection中，那么可以定义这样一个泛型方法:
```java
public static <T> void fromArrayToCollection(T[] a, Collection<T> c) {
    for (T o : a) {
        c.add(o); // Correct
    }
}
```

我们又见到了熟悉的尖括号对`<T>`，同泛型类定义一样，这里的`T`表示类型参数，在方法声明的形参中，我们就可以使用`T`来指定形参的类型。当我们使用这个函数时，T的值就根据传入的参数类型来决定。
再看个例子就懂啦：
```java
String[] strArr = new String[10];
List<String> strList = new ArrayList<>();
fromArrayToCollection(strArr, strList); // 调用时传入的参数是String数组和列表，这时类型参数T的具体值就是String类

Integer[] intArr = new Integer[10];
List<Integer> intList = new ArrayList<>();
fromArrayToCollection(intArr, intList); // 调用时传入的参数是Integer数组和列表，这时类型参数T的具体值就是Integer类

fromArrayToCollection(strArr, intList); // error， 如果根据输入变量的推断出的参数类型不一样，编译器就会报错啦
```

---
## Bounded类型参数

除了像上面的例子中展示的定义一个普通的类型参数之外，有时候我们可能会想要限制类型参数的类型，比如限定它只能是某个接口或类的子类。这时我们就需要用到Bounded类型参数(Bounded Type Parameters)。
Bounded类型参数的定义如下例所示，即类型参数名，接上`extends`关键词，后面紧跟上界(upper bound)，表示这个泛型类/接口或泛型方法接受的类型参数必须为upper bound的子类/接口。
```java
public class NaturalNumber<T extends Integer> {
    private T n;
    public NaturalNumber(T n)  { this.n = n; }
    public boolean isEven() {
        return n.intValue() % 2 == 0;
    }
}
```

类型参数还可以定义多个上界，表示类型参数必须同时是这几个类或接口的子类。写法如下所示：
```java
class A { /* ... */ }
interface B { /* ... */ }
interface C { /* ... */ }
class D <T extends A & B & C> { /* ... */ }
```

值得注意的一点，所有的上界中最多有一个是Class，其余皆为Interface，并且这个Class要放在第一位。否则会报错。
```java
class D <T extends B & A & C> { /* ... */ } // 这样就不行哦
```

---
# 神奇的通配符
在上面的几节里我们介绍了关于泛型的定义相关的内容，在这一节，我们将介绍一个使用泛型时会见到的东西——通配符`?`。

通配符的使用场景通常有如下几种：
* **as the type of a parameter, field, or local variable**
* sometimes **as a return type** (though it is better programming practice to be more specific)

另一方面，
> *The wildcard is never used as a type argument for a generic method invocation, a generic class instance creation, or a supertype.*

---
## Unbounded通配符
单独使用的通配符 `?` 被称为无界(unbounded)通配符，例如`List<?>`，这时这个`List`被称为“一个未知类型的`List`”。
无界通配符适用于以下两种场景：

* If you are writing a method that can be implemented using functionality provided in the Object class.
* When the code is using methods in the generic class that don't depend on the type parameter. For example, List.size or List.clear. In fact, Class<?> is so often used because most of the methods in Class<T> do not depend on T.

还是举个例子：
```java
public static void print(List<Object> list) {
    for(Object o: list) {
        System.out.println(o);
    }
}
```
上面这个方法呢，本意是想能打印任意类型的`List`，然而这样做并不能达到目的。因为像`List<String>`, `List<Integer>`, `List<MyClass>` 并不是`List<Object>`的子类（更多关于*泛型的继承关系*请前往下面的章节往下看）。
这时候Unbounded Wildcard就派上用场了，把`List<Object>`换成`List<?>`问题就解决啦。

> *It's important to note that `List<Object>` and `List<?>` are not the same. You can insert an Object, or any subtype of Object, into a `List<Object>`. But you can **only insert null** into a `List<?>`.*

---
## Bounded通配符
跟无界通配符相对应的，还有Bounded通配符。Bounded通配符又分为如下两类：

### Upper Bounded - <? extends supertype>

当希望类型变量的值是某个类（接口）以及其子类时，就可以使用Upper Bounded Wildcards - `<? extends supertype>`。

举个例子：
```java
public static void sum(List<? extends Number> list) {
    double s = 0.0;
    for (Number n : list)
        s += n.doubleValue();
    return s;
}
```
上面的sum方法就对`Number`及其子类的`List`进行处理，所以`List<Integer>`,  `List<Double>`,  `List<Float>`,  `List<Number>`都是合法参数。如果使用`List<Number>`而不是`List<? extends Number>`那么将只有`List<Number>`是合法的。

---
### Lower Bounded - <? super subtype>

与Upper Bounded Wildcards相反，Lower Bounded Wildcards - `<? super subtype>`限制的是下限，即用于指定参数可以是一个具体的类型以及它的父类。

举个例子：
```java
public static void addNumbers(List<? super Integer> list) {
    for (int i = 1; i <= 10; i++) {
        list.add(i);
    }
}
```

上面这个方法希望任何一个能存储`Integer`值的类型都可以作为参数，那么相比较于只允许`Integer`类型作为元素的`List<Integer>`，`List<? super Integer>`使得`List<Number>`, `List<Object>`类型都可以作为参数使用。

---
## 通配符的使用指南
在考虑何时该使用哪种通配符之前，我们先将函数的变量分个类：

* **输入变量(Producer)** - 输入变量为代码提供数据。举个栗子，在拷贝方法`copy(src, dest)`中，source提供数据源，所以它是输入变量
* **输出变量(Consumer)** - 输出变量用于存储提供给其他用途的数据。还是原来的栗子，在拷贝方法`copy(src, dest)`中的,dest用于接收数据，所以它是输出变量。

当然啦，也有即作为输入又作为输出的变量，我们在具体的指导规则中再讨论。

通过输入/输出原则我们就可以确定不同通配符的适用情形了：
* **对于Producer，使用`<? extends supertype>`**。对于内部代码而言，只要输入变量有它要调用的接口就可以了，至于其子类自己添加的那些并不关心。
* **对于Consumer，使用`<? super subtype>`**。对于输出而言，使用下界通配符才能保证要输出的字段都可以被赋值。
* 当作为输入变量是可以通过`Object`类中定义的方法访问时，使用Unbounded wildcard(`?`)
* 当一个变量既作为输入变量又作为输出变量时，就不要使用通配符啦

> PECS stands for producer-extends, consumer-super. (from *Effective Java*)

以上这些原则并不适用与返回值，**应该尽量避免在返回值中使用通配符**，因为这种写法就是在强制方法的调用者来处理通配符。如果调用者在使用一个类或方法时需要考虑通配符，那么你就要思考一下是不是这个类或方法的设计有问题啦。

再举个特殊的例子，`Comparable`和`Comparator`总是扮演consumer的角色，因此在使用时都推荐使用`Comparable<? super T>`和`Comparator<? super T>`
(A comparable of T consumes T instances (and produces integers indicating order relations))。
```java
public static <E extends Comparable<? super E>> E max(Collection<? extends E> c) { 
    if (c.isEmpty())
        throw new IllegalArgumentException("Empty collection");
    E result = null;
    for (E e : c)
        if (result == null || e.compareTo(result) > 0)
            result = Objects.requireNonNull(e);
    return result;
}
```

### 通配符与类型参数的对偶性（duality）
很多方法既可以使用通配符声明也可以使用普通类型参数声明。比如下面的`swap`方法：
```java
public static <E> void swap(List<E> list, int i, int j);
public static void swap(List<?> list, int i, int j); 
```

那么这两种方式那种更好呢？对于公共API，推荐使用第二种方式，因为更简单。毕竟swap方法只是交换list中两个元素的位置，实际上并不需要考虑类型参数。
**一般来说，如果一个类型参数只在方法声明中出现，那么应该用通配符替换它**。(if a type parameter appears only once in a method declaration, replace it with a wildcard.)

那么使用通配符声明的方式，内部要如何实现呢？像下面这样写吗？
```java
public static void swap(List<?> list, int i, int j) {
    list.set(i, list.set(j, list.get(i)));
}
```
这样写是通不过编译的，编译器会报如下错误：
```java
error: incompatible types: Object cannot be converted to CAP#1
        list.set(i, list.set(j, list.get(i)));
                                        ^
  where CAP#1 is a fresh type-variable:
    CAP#1 extends Object from capture of ?
```

问题就出在`List<?>`，除了`null`你是不能忘`List<?>`类型的实例中塞任何东西的。那怎么办呢？答案就是把两种声明方式结合起来使用：
```java
public static void swap(List<?> list, int i, int j) {
    swapHelper(list, i, j);
}
// Private helper method for wildcard capture
private static <E> void swapHelper(List<E> list, int i, int j) { 
    list.set(i, list.set(j, list.get(i)));
}
```
通过引入一个私有的`swapHelper`方法，我们既对外提供了使用通配符的便利性，又在内部利用类型参数保证了类型安全。

---
# 泛型中的继承关系

## 一般泛型的继承关系
废话不多说，直接看图：
![java generics.png-17.8kB][2]

上图中MyList接口的定义如下：
```java
interface MyList<E,T> extends List<E> {
  void setValue(int index, T val);
  // ...
}
```

## 通配符的继承关系
照样先上图：
![java generic wildcard subtyping.png-19.1kB][3]
关于上图中左边的关系，以`Integer`和`Number`为例，`Integer`是`Number`的子类。这里再说明一下，在使用一般泛型类型参数的泛型类继承关系中`List<Integer>`并不是`List<Number>`的子类，但它们都是`List<?>`的子类。
这样写的代码是会报错的：
```java
List<Integer> intList = new ArrayList<>();
List<Number> numList = intList; // compile error
```
那如果我们想List<Integer>的元素能够以Number的方法访问要怎么写呢？根据右边展示的关系，可以使用下面这段代码来实现：
```java
List<? extends Integer> intList = new ArrayList<>();
List<? extends Number> numList = intList;
```

---

关于泛型的一些介绍就先这样吧，把大概的框架梳理了一下，没想到断断续续每天一两个小时的，也花了小一周的时间。
另外还有一些相关的话题，比如Java中的类型擦除，就没有放在这里，准备之后单独新开一篇做整理。
下次见啦~

---

**参考资料**
* [The Java Tutorial - Generics][1]
* *Core Java: Volumn 1*
* *Effective Java (3rd Edition) - Chapter 5 Generics*
* *深入理解Java虚拟机（第二版）*

 [1]: https://docs.oracle.com/javase/tutorial/java/generics/index.html
 [2]: http://static.zybuluo.com/JaneL/w21v266p6vpcn5ek2twqiir3/java%20generics.png
 [3]: http://static.zybuluo.com/JaneL/6xydzj9b0o6o8npu9v07sfu4/java%20generic%20wildcard%20subtyping.png