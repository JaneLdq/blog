---
title: GC算法笔记（三）复制算法
date: 2019-07-31 19:50:57
categories: 技术笔记
tags: GC
---

复制 (Copying GC) 算法最早由 Marvin Lee Minsky 于1963年在 *A LISP Garbage Collector Algorithm Using Serial Secondary Storage* 中提出。

复制算法的基本思想很简单，就是把整个堆空间一分为二，每次只使用一半的空间用于分配：
* 正在使用的空间叫做 **From** 空间，空置的另一半称为 **To** 空间；
* 执行 GC 时，将 From 空间的**活动对象**都复制到 To 空间，然后对整个 From 空间进行回收；
* 复制完成后，将 From 空间和 To 空间交换，即原来的 From 空间变成了 To 空间，原来的 To 空间变成了 From 空间，直到下一次 GC，两个空间再次交换。

复制算法的一大特点就是它关注的是**活动对象**，而不像之前提到的**标记-清除算法**及**引用计数法**，都是遍历查找非活动对象。

<!--more-->
复制算法概要如下图所示：
![copying-gc][1]

接下来，我们将主要介绍两种复制算法的实现方式：
* **递归实现** - 由 Robert R. Fenichel 和 Jerome C. Yochelson 于 1969 年在 *A LISP Garbage-Collector for Virtual-Memory Computer Systems* 中提出。
* **迭代实现** - 由 C.J. Cheney 于 1970 年在 *A Nonrecursive List Compacting Algorithm* 中提出。

这两篇文章都不长，可以找来读一读～

---
# 递归复制算法
Robert R. Fenichel 和 Jerome C. Yochelson 给出的GC复制算法的实现**采用递归的形式，遍历每个 GC Root，并递归复制其子对象**。

下面是算法伪码：
```c++
copying() {
    $free = $to_start
    // 遍历每个GC Root
    for (r: $roots) {
        // 将root指向GC后的新地址
        *r = copy(*r)
    }
    // 交换From和To空间
    swap($from_start, $to_start)
}

copy(obj) {
    if (obj.tag != COPIED) {
        // 这里虽然将obj拷贝到了To空间，但是obj中包含的指针还指向From空间
        copy_data($free, obj, obj.size)
        obj.tag = COPIED
        // forwarding指针指向该对象在To空间的新地址，之后遇到指向From空间的该对象时要将指针更新为forwarding指针
        obj.forwarding = $free
        $free += obj.size

        // 递归复制子对象
        for (child: children(obj.forwarding)) {
            *child = copy(*child)
        }
    }
    return obj.forwarding
}
```

很认真地画了个图，希望能带大家更直观地感受一下整个过程：

![recursive-copying-gc][2]

上图中一共有四个对象存在于 From 空间，其中从 Root 开始遍历一共能找到 3 个活动对象 `A`、`B` 和 `D`，通过递归复制，将这个三个对象依次复制到 To 空间得到 `A'`、`B'` 和 `D'`。可以看到复制到 To 空间后，对象内部包含的指针还是指向 From 空间的，直到递归返回时，用复制子对象得到的 forwarding 覆盖原子对象指针才完成指针的更新。最后，回收 From 空间所有对象并完成空间交换。

---
## 优点
* **优秀的吞吐量** - GC 复制算法只搜索并复制活动对象，因此能在较短时间内完成 GC，吞吐量优秀。它消耗的时间与活动对象的数量成正比，而不受堆大小的影响。
* **可实现高速分配** - GC 复制算法不使用空闲链表，分块在一个连续的内存空间，因此在进行分配时只需比较整个空闲分块的大小是否足够就可以了，而不需要链表遍历操作。
* **不会发生碎片化** - 每次进行复制时都会将活动对象重新集中（压缩），避免了碎片化。
* **与缓存兼容** - 在上述递归实现的复制算法中，有引用关系的对象会被安排在堆中离彼此较近的位置，有利于提高高速缓存命中率。

---
## 缺点
* **堆利用率低** - GC 复制算法将堆空间一分为二，每次只能利用一般用于分配。这一缺点可以通过搭配使用复制算法和标记-清除算法得到改善。
* **递归调用函数** - 每次递归调用都会消耗栈空间，由此带来了不少额外负担，而且还有栈溢出的可能。


我们接下来要介绍的 Cheney 复制算法就采用迭代的方式避免了递归调用的问题。

---
# Cheney 复制算法
Cheney提出的复制算法采用迭代的方式进行复制，具体思路如下所示：
```C++
copying() {
    // scan是用于搜索复制完成的对象的指针，$free是指向To空间分块开头的指针
    scan = $free = $to_start
    // 首先复制所有的roots
    for (r: $root) {
        *r = copy(*r)
    }

    // 然后从已经复制到To空间的对象开始复制它们引用的子对象
    while (scan != $free) {
        for (child: children(scan)) {
            *child = copy(*child)
        }
        // 向前移动scan指针，指向下一个已经复制到To空间的对象
        scan += scan.size
    }

    // 交换From和To空间
    swap($from_start, $to_start)
}

copy(obj) {
    // 检查obj.forwarding指针是否指向To空间，如果是，说明该对象已经被复制过，直接返回其forwarding指针
    // 否则进行复制
    if (is_not_pointer_to_heap(obj.forwarding, $to_start)) {
        copy_data($free, obj, obj.size)
        obj.forwarding = $free
        $free += obj.size
    }
    return obj.forwarding
}
```

Cheney算法其实质其实就是**广度优先遍历**：
* 新引入的 **scan** 指针和 **$free** 指针组成了一个 **FIFO队列**（**scan** 指向队首，**$free** 指向队尾）；
* 首先将所有 GC Roots 依次复制并加入队列（**scan**保持不动，**$free**移到下一个空闲分块）；
* GC Root复制完成后，移动 **scan** 指针从队列中依次读取被复制的对象，遍历其引用的子对象，完成复制并加入队列（整个过程中 **scan** 每前进一次， **$free** 可能前进零次或 N 次，取决于是否有引用子对象）；
* 直到 **scan** 和 **$free** 相遇，说明队列为空，复制结束。

看个例子加深一下理解吧～
![cheney copying gc][3]

---
## 优点
* **迭代算法**，消除了递归调用函数的额外负担和栈的消耗。
* **直接将堆空间用作队列**，省去了额外内存空间开销。

---
## 缺点
* 由于广度优先遍历的特点，Cheney复制算法中无法将有引用关系的对象复制到相邻的位置，降低了高速缓存的命中率（还记得“访问的局部性”吗）。

---

最后。总结一下本文的要点：
* GC 复制算法最主要的优势在于它消除了碎片化，连续空间有利于高速分配，而代价是牺牲了一半的堆空间，降低了堆空间的利用率。
* GC 复制算法的具体实现有递归和迭代两种方式，各有利弊。

除此之外，还有诸如近似深度优先搜索算法、多空间复制算法等改进算法，不过这里就不详细介绍了，感兴趣的话可以自己找资料研究～

---

8/5 21:49
为走心配图的自己点个赞～画图虽然有点费时间，但我觉得对于理解过程还是很有帮助哒！

---

**参考资料**
* *垃圾回收的算法与实现*
* Robert R. Fenichel, Jerome C. Yochelson, *A LISP Garbage-Collector for Virtual-Memory Computer Systems*, 1969
* C. J. Cheney, *A Nonrecursive List Compacting Algorithm*, 1970

  [1]:/uploads/images/copying-gc.svg
  [2]:/uploads/images/recursive-copying-gc.png
  [3]:/uploads/images/cheney-copying-gc.png