---
title: 在 Java 中实现单例模式
date: 2020-05-05 16:05:45
categories: 设计模式
tags: 
- Java
- 单例模式
---

虽然单例模式很好理解，在具体实现的时候，还是有很多细节需要注意。几年前，我写过一篇在 Javascript 中实现单例模式的笔记，但其实在不同语言中实现单例模式还是有不少的区别，尤其像 Java、C++ 这类多线程语言。

在本文中，我们一起来看看在 Java 中实现单例模式都有哪些方式，要考虑哪些问题。
<!--more-->

---
# 单线程中的单例模式
一说到单例模式，大家的第一反应可能都是：那还不简单，看我一下给你写好几个版本出来：

* **公共静态类变量**
```java
public class EagerHelper {
    public static final EagerHelper INSTANCE = new EagerHelper();
    private EagerHelper() {}
}
```

* **私有静态类变量 + 工厂方法**
```java
public class EagerFactoryHelper {
    private static final EagerFactoryHelper INSTANCE = new EagerFactoryHelper();
    private EagerFactoryHelper() {}
    public static EagerFactoryHelper getInstance() {
        return INSTANCE;
    }
}
```

* **工厂方法 + Lazy Initialization**
```java
public class LazyFactoryHelper {
    private static LazyFactoryHelper INSTANCE = null;
    private LazyFactoryHelper() {}
    public static LazyFactoryHelper getInstance() {
        if (INSTANCE == null) {
            INSTANCE = new LazyFactoryHelper();
        }
        return INSTANCE;
    }
}
```
乍一看，上面这三种都实现了单例模式：
* 使用类变量共享唯一实例
* 私有构造函数防止外部创建实例
* 还用到了 Lazy Initialzation 降低在加载类时的开销，将初始化推迟到第一次访问实例时

然而，上面的实现都只能在单线程环境中使用，一旦涉及了多线程，就会出问题了。如果有多个线程同时访问，就可能出现初始化多个实例，或者某个线程获得尚未完成初始化的实例，导致程序运行出错。

那么，如何实现线程安全的单例呢？

---
# 多线程中的单例模式
提到多线程，一定离不开锁机制。提到 Java 中的多线程，一定离不开 `synchronized`、`volatile` 等关键字。

## **synchronized** 保平安
一般来说，使用 `synchronized` 关键字修饰方法是最容易想到的实现方式：
```java
public class SyncHelper {
    private static SyncHelper INSTANCE = null;
    private SyncHelper() {}
    public synchronized static SyncHelper getInstance() {
        if (INSTANCE == null) {
            INSTANCE = new SyncHelper();
        }
        return INSTANCE;
    }
}
```

然而，这样实现导致每次请求实例都要执行加锁、释放锁的操作，这会对性能造成不小的影响。而事实上，只有最初请求实例的几个线程可能会遇到竞争的情况，当唯一的实例创建完成后，后续请求实例的线程本可以直接获得实例，这时还要先申请锁实在是没必要。

---
## 双重校验锁升性能
为了**减少不必要的加锁操作带来的性能开销**，可以采用 **双重校验锁 (Double-checked Locking)** 机制。该机制按照如下逻辑进行同步：

1. 检查实例是否已创建，如果是，则直接返回已有实例
2. 否则，请求锁
3. 再次检查实例是否已创建，如果之前拿到锁的线程已经创建好实例，那么直接返回已有实例
4. 否则，创建并返回实例

代码如下：
```java
public class DoubleCheckedHelper {
    private static DoubleCheckedHelper INSTANCE = null;
    private DoubleCheckedHelper() {}
    public static DoubleCheckedHelper getInstance() {
        if (INSTANCE == null) {
            synchronized (DoubleCheckedHelper.class) {
                if (INSTANCE == null) {
                    INSTANCE = new DoubleCheckedHelper();
                }
            }
        }
        return INSTANCE;
    }
}
```

上面这段代码真的没有问题吗？我们要意识到**实例的初始化并不是一瞬间完成的事情**。考虑如下情况：
当线程 A 正在初始化实例但尚未完成，这时 `INSTANCE` 已经指向了一块分配好的内存。此时线程 B 调用 `getInstance` 方法，`INSTANCE == null` 这句判断的值为 false，这个尚未完成初始化的“半成品”实例引用会被返回给线程 B，导致线程 B 运行出错。

因此，上面这个版本的双重校验锁模式是有问题的。如何修正呢？很简单，引入 `volatile` 关键字。这里用到了 `volatile` 提供的 **happens-before** 保证。（不熟悉 `volatile`？，请移步各大搜索引擎～）
修正后的版本如下：
```java
public class DoubleCheckedHelper {
    private volatile static DoubleCheckedHelper INSTANCE = null;
    private DoubleCheckedHelper() {}
    public static DoubleCheckedHelper getInstance() {
        DoubleCheckedHelper localRef = INSTANCE;
        if (localRef == null) {
            synchronized (DoubleCheckedHelper.class) {
                localRef = INSTANCE;
                if (localRef == null) {
                    localRef = INSTANCE = new DoubleCheckedHelper();
                }
            }
        }
        return localRef;
    }
}
```
在上面这段代码中，我们除了引入了 `volatile` 修饰实例变量之外，还在 `getInstance` 方法中引入了一个局部变量，这么做是为了降低访问 `volatile` 变量带来的性能开销。

---
## 巧用内部类优化代码
修正过的双重校验锁机制虽然是线程安全的，但这段代码看起来实在是不怎么优美。那么，有没有更简洁优美的方式呢？
追求尽善尽美的程序员们开发出了 **Initialization-on-demand holder** 模式，利用私有内部类对代码进行了优化。
实现如下：
```java
public class InitOnDemandHelper {
    private InitOnDemandHelper() {}
    private static class LazyHolder {
        static final InitOnDemandHelper INSTANCE = new InitOnDemandHelper();
    }
    public static InitOnDemandHelper getInstance() {
        return LazyHolder.INSTANCE;
    }
}
```
**Initialization-on-demand holder** 模式利用了 JVM 的类初始化机制：
* 当类加载器加载 `InitOnDemandHelper` 类时，由于它没有定义任何静态类变量，`InitOnDemandHelper` 类的初始化基本上啥事儿也不用干，**节省了类初始化的开销**；
* 只有当外部代码调用 `InitOnDemandHelper.getInstance()` 方法时，`LazyHolder` 才会被初始化，这时才会创建实例，**实现了 Lazy Initialization**；
* JVM 会保证类的初始化在多线程环境中被正确地加锁、同步，如果有多个线程同时初始化一个类，那么只会有一个线程执行类的初始化方法，其他线程将被阻塞直到类初始化完成。这一 JVM 的硬性要求保证了这一实现是**线程安全**的。


---
## Enum 大法好！
实不相瞒，本小白也是今天才学到如此奇技淫巧，着实没想到枚举类型还能这样用，真是妙啊～

### 来自反射机制的反击
上文提到的所有对**单一**实例的限制都依赖于**私有类构造器**筑起的高墙。然而，Java 的**反射**大法可以轻易打破这道壁垒。利用反射机制，我们可以拿到一个类声明的所有构造器，还可以修改构造器的可见性。

举个例子：
```java
public class Helper {
    private static final Helper INSTANCE = new Helper();
    private Helper() {}
    public static Helper getInstance() {
        return INSTANCE;
    }

    public static void main(String[] args) {
        Helper instance1 = Helper.getInstance();

        Helper instance2 = null;
        try {
            Constructor[] constructors = Helper.class.getDeclaredConstructors();
            for (Constructor constructor : constructors) {
                constructor.setAccessible(true);
                instance2 = (Helper) constructor.newInstance();
                break;
            }
        } catch (Exception e) {
            e.printStackTrace();
        }

        System.out.println(instance1.equals(instance2));
        System.out.println("Instance 1: " + instance1.hashCode());
        System.out.println("Instance 2: " + instance2.hashCode()));
    }
}
```
运行这段代码的控制台输出如下：
```
false
Instance 1: 1639705018
Instance 2: 1627674070
```
事实证明，我们成功地利用反射机制策反了 `Helper` 类，更改了它的构造器的可见性，创建出了另一个实例。

---
### 序列化也是个问题
除了会被反射机制轻易的瓦解，上述单例模式的实现在遇到序列化与反序列化时也一样不堪一击。
当我们从某个文件中反序列化一个实例时，Java 中的序列化机制并不受类构造器可见性的限制，也就是说，即使是类构造器是私有的，仍然可以反序化出一个新的实例。
举个例子：
```java
public class Helper implements Serializable {

    private static Helper helper = new Helper();

    private Helper() {}

    public static Helper getInstance() {
        return helper;
    }

    public static void main(String[] args) {
        try {
            ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("sample.dat"));
            Helper h1 = Helper.getInstance();
            out.writeObject(h1);
            out.close();

            ObjectInputStream in = new ObjectInputStream(new FileInputStream("sample.dat"));
            Helper h2 = (Helper) in.readObject();
            in.close();

            System.out.println(h1.equals(h2));
            System.out.println("instance 1: " + h1.hashCode());
            System.out.println("instance 2: " + h2.hashCode());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
控制台输出如下：
```
false
instance 1: 491044090
instance 2: 189568618
```
在上面这段代码中，`Helper` 类实现了 `Serialzable` 接口。在 `main` 方法中，我们先将实例序列化存到了一个文件中，然后将其反序列化得到了一个新的实例。通过比较两个实例的哈希值也能证明它们确实是两个不同的实例。这明显违反了单例模式唯一实例的要求。

要解决这个问题，我们可以为上面的 `Helper` 类实现一个特殊的方法 `readResolve`。这个方法返回一个 Object 类型，其目的就是用来替换通过反序列化得到的实例。
更改后的 `Helper` 类如下：
```java
public class Helper implements Serializable {

    private static Helper helper = new Helper();

    private Helper() {}

    public static Helper getInstance() {
        return helper;
    }

    // 新增 readResolve 方法直接返回现有的 helper 实例
    protected Object readResolve()
    {
        return helper;
    }
}
```
再运行上面的 `main` 方法，控制台输出如下，实现了单一实例：
```
true
instance 1: 491044090
instance 2: 491044090
```

---

既然上文提到的诸多实现都面临着这些潜在的威胁，就没有更好的实现单例模式的方法了吗？
请回顾本小节的标题：**Enum 大法好！**

在 Java 中，声明一个枚举类型实际上定义了一个枚举类，这个类和普通的类一样，可以拥有属性和方法。除此之外，枚举类还有一些特别的属性，其中就包括如下几点：
* **枚举类的构造器是私有的，且不允许通过反射创建枚举类型实例** —— 完美阻碍通过反射搞破坏这条路
* **JVM 会保证枚举类的序列化和反序列化的正确执行** —— 省去了自行处理序列化/反序列化的麻烦
* **JVM 还负责保证枚举类实例是线程安全的** —— 适用于多线程

通过枚举类实现单例模式的代码相当精简，定义一个只拥有唯一值的枚举类就搞定了：
```java
public enum EnumHelper {
    INSTANCE;
    // other methods
}
```

若我们将上面的 `Helper` 类声明为 `enum`，再运行上面的 `main` 方法，控制台会报如下错误：
```java
java.lang.IllegalArgumentException: Cannot reflectively create enum objects
	at java.lang.reflect.Constructor.newInstance(Constructor.java:417)
```

鉴于枚举类型在实现单例模式中的优秀表现，Joshua Bloch 在 *Effective Java* 一书中也提出了在 Java 中：**"A single-element enum type is often the best way to implement a singleton."** 的观点。

---

那么，关于 Java 中单例模式实现的讨论到这里就结束啦～

**参考资料**
* [Double-checked Locking - wiki](https://en.wikipedia.org/wiki/Double-checked_locking#Usage_in_Java)
* [Why is volatile used in double checked locking](https://stackoverflow.com/questions/7855700/why-is-volatile-used-in-double-checked-locking)
* [Initialization-on-demand Holder - wiki](https://en.wikipedia.org/wiki/Initialization-on-demand_holder_idiom)
* [Java Singletons Using Enum](https://dzone.com/articles/java-singletons-using-enum)
* Effective Java, 3rd edition