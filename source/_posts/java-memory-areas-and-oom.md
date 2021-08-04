---
title: Java内存管理之运行时数据区
date: 2019-07-13 22:24:13
categories: 技术笔记
tags: Java
---

“你是什么垃圾？” ¯\\\_(ツ)\_/¯

最近垃圾分类的话题那是相当火热，作为紧跟时代的程序员，我们不仅要开创格子衫的天下，还要走在时尚的最前沿～于是，我决定趁此机会，理一理Java中的内存管理与垃圾回收机制～

「辣鸡系列」正式开坑～(⊙v⊙)

---

在日常的 Java 开发中，有 JVM 的自动内存管理机制，我们一般也不怎么关注 Java 的内存分配和垃圾回收，因为不太容易出现内存泄漏和溢出的问题。不太容易并不代表不会，如果对 JVM 的内存管理不了解的话，一旦出现问题，debug 就不那么美妙了。

作为本系列的开篇，自然要从垃圾的来源 —— Java 的运行时数据区 (Run-Time Data Area) 讲起。

<!--more-->
---
## 运行时数据区
JVM 在执行 Java 程序时会把它管理的内存区域划分为不同的数据区域，每个区域的用途、创建和销毁的时间都不同。
根据最新的 JVM 规范 *[The Java® Virtual Machine Specification (Java SE 12 Edition)][2]*，JVM管理的内存包括如下几个区域。
（虽然说是最新的，不过就规范而言，这一部分的内容倒是基本没怎么变动）。

### 程序计数器
JVM 对多线程的支持是通过线程轮流切换并分配处理器执行时间的方式实现的，因此每个线程都有一个独立的程序计数器 (Program Counter Register)，用于保存当前线程所执行到的位置，这样在线程切换后就能恢复到正确的位置。
* 如果当前执行的是一个 Java 方法，那么 *pc* 的值为正在执行的虚拟机字节码指令的地址；
* 如果当前执行的是一个 Native 方法，那么 *pc* 的值为 **Undefined**

**「异常情况」**
*程序计数器是 JVM 规范中唯一一个没有规定任何 OutOfMemoryError 情况的区域。*

---
### JVM 栈
每一个 JVM 线程都有一个 JVM 栈 (Java Virtual Machine Stack)， 它的生命周期与线程相同。JVM 栈既可以是固定大小的，也可以是可扩展的。
* JVM 栈描述的是 Java 方法执行的内存模型，每个方法在执行时都会创建一个栈帧 （Frame），用于存储局部变量表、操作数栈、动态链接、方法出口等信息
* 每个方法从调用到执行完成，就对应一个栈帧在 JVM 栈中从入栈到出栈


**「异常情况」**
1. *如果线程请求的栈深度大于虚拟机所允许的深度，将抛出 *StackOverflowError*。*
2. *如果 JVM 栈可以动态扩展（当前大部分虚拟机都可以动态扩展），如果扩展是无法申请到足够的内存，将抛出 *OutOfMemoryError*。*

---
### Native 方法栈
Native 方法栈 (Native Method Stack) 与 JVM 栈的作用非常相似，唯一的区别在于 Native 方法栈为 Native 方法服务，而 JVM 栈是为 Java 方法服务。(Native方法指用非Java语言写的方法)。
Native 方法栈与 JVM 栈一样，即可以是固定大小的，也可以扩展。

**「异常情况」**
*也会抛出 StackOverflowError 和 OutOfMemoryError，具体情况参见 JVM 栈「异常情况」。*

---
### 堆
Java 堆 (Heap) 是被**所有线程共享**的一块内存区域，在虚拟机启动时创建，用于存放对象实例。
JVM 规范的描述如下：
> The heap is the run-time data area from which memory for all class instances and arrays is allocated.
> (所有的对象实例以及数组都要在堆上分配。)

不过随着 JIT(Just In Time) 编译器和逃逸分析技术的发展，栈上分配、标量替换优化技术等使得这里的“所有”没有那么绝对了。

根据 JVM 规范的规定，**Java 堆可以处于物理上不连续的内存空间，只要逻辑上是连续的即可**。
与栈相似，堆可以是固定大小的，也可以是可扩展的。（目前的主流虚拟机都按照可扩展实现的）。

**堆是垃圾收集管理的主要区域**，因此也被称作“GC堆”。（划重点，这里就回答了我们本篇笔记标题的提问：**垃圾从何而来？从堆而来～**）

**「异常情况」**
*如果堆中没有足够的内存完成实例分配，并且堆也无法再扩展时，会抛出 OutOfMemoryError。*

举个例子：
```java
/**
 * VM Args: -Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
 */
public class HeapOOM {
    static class OOMObject {}

    public static void main(String[] args) {
        List<OOMObject> list = new ArrayList<>();
        while(true) {
            list.add(new OOMObject());
        }
    }
}
```
运行结果：
```java
java.lang.OutOfMemoryError: Java heap space
Dumping heap to java_pid82766.hprof ...
Heap dump file created [27851304 bytes in 0.120 secs]
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
    ...
    at oom.HeapOOM.main(HeapOOM.java:17)
```

---
### 方法区
方法区 (Method Area) 跟 Java 堆一样，也是**所有线程共享**的内存区域，它在虚拟机启动时创建，用于存储已被虚拟机加载的类信息、常量、静态看量、即时编译器编译后的代码等数据
JVM 规范对方法区的限制很宽松，它和堆一样不需要连续的物理内存，可以是固定大小或可扩展的，**还可以选择不实现垃圾回收**。
在方法区中，主要的回收目标是**对常量池的回收**和**对类的卸载**，一般来说，这两类可回收的空间不大，但是请记住，**不回收并不是个好选择**。

在早期版本(6及之前)的 JDK 中，HotSpot 虚拟机把GC分代收集扩展到了方法区，因此很多人又称方法区为“永久代”。需要注意的是，二者并不等价，方法区是概念，而永久代只是方法区的一种实现方式。对于其他虚拟机是不存在永久代这一概念的。事实上，在 JDK8 中，HotSpot 也不再以永久代实现方法区。

**「异常情况」**
*当方法区无法满足内存分配需求时，会抛出 OutOfMemoryError。*

---
### 运行时常量池
运行时常量池 (Run-Time Constant Pool) 是方法区的一部分。
Class 文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池 (Constant Pool Table)，用于存放编译期生成的各种字面量和符号引用，这部分内容在类加载后存放于运行时常量池。也就是说每一个被加载的类或接口都对应有一个运行时常量池。

相比于 Class 文件中的常量池，运行时常量池**具备动态性**。在运行期间，也可能将新的常量放入池中，这种特性利用的较多的是 `String` 类的 `intern()` 方法。`String.intern()` 是一个 Native 方法，它的作用是：如果字符串常量池中已经包含一个等于此 `String`对象的字符串，则返回代表池中这个字符串的 `String`对象，否则，将此 `String` 对象包含的字符串添加到常量池中，并且返回此 `String` 对象的引用。

**「异常情况」**
*运行时常量池位于方法区，自然也收到方法区内存的限制，当常量池无法申请到内存时将会抛出 OutOfMemeoryError*。

---
关于方法区和常量池 OutOfMemoryError 的例子，根据 JDK 版本的不同，抛异常时的提示信息会不同。比如下面这段代码：
```java
/**
 * VM Args: 
 * JDK6 and before: -XX:PerSize=10M -XX:MaxPersize=10M
 */
public class RuntimeConstantPoolOOM {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();
        int i = 0;
        while (true) {
            list.add(String.valueOf(i++).intern());
        }
    }
}
```
在 JDK6 及之前的版本，方法区是被分配到永久代中的，可以通过 JVM 参数 `-XX:PermSize` 和 `-XX:MaxPermSize` 来限制方法区的大小，从而间接限制常量池的容量。据说运行以上代码会抛出如下错误（古早的JDK版本已经Oracle不给下载了，本来还想跑一下看看的）：
```java
Exception in thread "main" java.lang.OutOfMemoryError: PermGen space
```
**"PermGen space"** 说明运行时常量池属于方法区（JDK6 JVM中的永久代）。
但是在 JDK7 及之后，HotSpot JVM 开始逐步取缔“永久代”，到 JDK8 就被彻底移除了，由新引入的“元空间” (Metaspace)接替它的角色。

我试了一下在 JDK8 上运行以上代码，由于“永久代”已经是过去式，上述两个参数 `Permsize` 和 `MaxPersize` 自然被移除了。在不显式限制内存大小的情况下运行上面的代码，可以一直运行下去而不会抛 OutOfMemoryError。
如果设置 `-Xmx10m` 来限制运行时内存的大小，运行后的结果如下所示：
```java
Exception in thread "main" java.lang.OutOfMemoryError: GC overhead limit exceeded
    ...
    at oom.RuntimeConstantPoolOOM.main(RuntimeConstantPoolOOM.java:15)
```
提示信息为 **"GC overhead limit exceeded"**。它表示程序运行时花在垃圾收集上的时间太多，效果却太差。
> By default the JVM is configured to throw this error if it spends more than 98% of the total time doing GC and when after the GC only less than 2% of the heap is recovered.

如果再调小一点，比如设为 `-Xmx5m`，则运行结果如下所示，这次变为了 **"Java heap space"**：
```java
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
    ...
    at oom.RuntimeConstantPoolOOM.main(RuntimeConstantPoolOOM.java:15)
```
关于 OutOfMemoryError 的各种情况，有官方的文档介绍，详情请戳[Understand the OutOfMemoryError Exception][3]，

---

总结一下，JVM 的运行时数据区可分为线程共享和线程私有两大类，一共包含六个区域，如下图表所示。

|线程共享|线程私有|
|---|---|
|堆<br>方法区<br>运行时常量池（包含在方法区内）|程序计数器<br>JVM 栈<br>Native 方法栈|

![Boyermoore Example][1]

---
## 直接内存
直接内存 (Direct Memory) 并不是 JVM 运行时数据区的一部分，也不是 JVM 规范中定义的内存区域，不过由于这部分内存也被频繁使用，既然都是内存，就稍微提一下。
直接内存由 NIO 类引入，它可以使用 Native 函数库**直接分配堆外内存**，然后通过存储在 Java 堆中的 `DirectByteBuffer` 对象引用这块内存进行操作。
直接内存的分配不受 Java 堆大小的限制，但是会**受本机总内存大小和处理器寻址空间的限制**。
在配置虚拟机参数时，如果忽略了直接内存，容易使得各个内存区域总和大于物理内存限制，导致动态扩展时出现 OutOfMemoryError。

### 为什么要引入直接内存呢？

`java.nio` 包中的 `ByteBuffer` 类有三个子类: `HeapByteBuffer`、`DirectByteBuffer` 和 `MappedByteBuffer`。顾名思义，`HeapByteBuffer`采用的从堆中分配内存的方式，其本质就是封装了一个 `byte[]`。那么为什么还要提供另外两个采用分配直接内存的实现类呢？
要回答这个问题，要从操作系统的I/O操作说起。操作系统的读写操作都是基于一块连续的空间 (contiguous sequence of bytes)，但是一个 `byte[]` 在堆上的会是一块连续的空间吗？并不一定，按照 JVM 规范中的描述，堆的物理实现都可以是不连续的，那这就无法保证堆上的字节数组一定会在一个连续空间里了。虽然 JVM 把一个一维字节数组分开存储的可能性不大，但是Java堆仍是不能被原生的I/O直接操作的，得先把堆上的数据拷贝到本机内存 (native memeory) 上，才能执行操作系统级的I/O操作。这样效率就低了很多，**直接内存**的出现就是为了提高这类操作的效率。

---

在本篇笔记中，我们搞清楚了 JVM 中到底有哪些运行时数据区，它们的创建时间、用途和可能造成的异常情况是什么。
那么，还记得标题问题的答案吗？六个数据区，谁是垃圾回收的重点关注对象呀？

**是它，是它，就是它，我们的小垃圾，Java 堆！**

不过别忘了，方法区的回收虽然不在 JVM 规范的强制要求内，也是不可忽视的哦～

---

**参考资料**
* *深入理解 Java 虚拟机（第二版）*
* [The Java® Virtual Machine Specification (Java SE 12 Edition)][2]
* [Understand the OutOfMemoryError Exception][3]
* [Understanding Java Buffer Pool Memory Space][4]

  [1]: /blog/uploads/images/jvm-runtime-data-area.svg
  [2]: https://docs.oracle.com/javase/specs/jvms/se12/html/index.html
  [3]: https://docs.oracle.com/en/java/javase/12/troubleshoot/troubleshoot-memory-leaks.html
  [4]: https://www.fusion-reactor.com/evangelism/understanding-java-buffer-pool-memory-space/
