---
title: GC算法笔记（四）标记-整理算法
date: 2019-08-06 19:32:20
categories: 技术笔记
tags: GC
---

前面我们提到 GC 算法的基本类型无非**标记-清除**、**引用计数**和**复制**算法三类，那么这篇笔记中介绍的**标记-整理 (Mark-Compact)** 算法就开始将 **标记-清除** 算法与**复制**算法结合的产物。不同于复制算法是从一个空间往另一个空间复制，标记-整理算法是在整个堆上**原地整理**，因此也提高了堆的利用率。

标记-整理算法的思路也很简单，分为两个阶段：
* **标记阶段** - 与标记-清除算法中的标记阶段一样，遍历堆上的对象，标记出活动对象。
* **整理阶段** - 移动或复制活动对象，将所有存活的对象都移动到堆的一端（仿佛被压缩），然后直接清理掉端边界以外的内容。

值得注意的一点是，整理的顺序会影响到程序的局部性。重排堆中对象时所遵循的顺序包括如下三种：
<!--more-->
* **任意顺序** - 对象的移动方式与它们的原始排列顺序和引用关系无关。*（实现简单且执行速度快，但只能处理单一大小的对象或对不同大小的对象分别整理；可能会把原本相邻的独享分散到不同的高速缓存行或虚拟内存页中，降低空间局部性）*。
* **线性顺序** - 将具有关联关系的对象排列在一起，如引用关系或同意数据结构中的相邻对象。
* **滑动顺序** - 将对象滑动到堆的一端，从而保持对象在堆中原有的分配顺序。*（不改变原有对象的相对排列顺序，因此不会改变局部性）*。

大多整理式回收算法都遵循任意顺序或滑动顺序。

接下来，我们一起来看看几种不同的标记-整理算法吧。

---
# Lisp2 算法
Lisp2 算法是由 Donald E. Knuth提出来的。Lisp2 算法属于滑动顺序整理算法。
该算法有个前提，即要在对象头里为 forwarding 指针专门留出空间。
Lisp2 算法的整理阶段由三次遍历完成：
1. 第一次遍历，**设定 forwarding 指针**
2. 第二次遍历，**更新指针**
3. 第三次遍历，**移动对象**

下面来看一下代码：
```c++
lisp2_compact() {
    set_forwarding_ptr()
    adjust_ptr()
    move_obj()
}

set_forwarding_ptr() {
    // scan用来遍历堆中的对象，new_address指向对象移动到的新地址
    scan = new_address = $heap_start
    while (scan < $head_end) {
        if (scan.mark == TRUE) {
            scan.forwarding = new_address
            new_address += scan.size
        }
        scan += scan.size
    }
}

adjust_ptr() {
    // 重写roots的指针
    for (r: $roots) {
        *r = (*r).forwarding
    }

    scan = $heap_start
    while (scan < $heap_end) {
        // 第二次遍历堆，重写所有活动对象的指针
        if (scan.mark == TRUE) {
            for (child: children(scan)) {
                *child = (*child).forwarding
            }
        }
        scan += scan.size
    }
}

move_obj() {
    scan = $free = $heap_start
    while (scan < $heap_end) {
        // 第三次遍历堆，找到活动对象并将其移动到forwarding指针指定的地方
        if (scan.mark == TRUE) {
            new_address = scan.forwarding
            copy_data(new_address, scan, scan.size)
            new_address.forwarding = NULL
            new_address.mark = FALSE
            $free += new_address.size
            scan += scan.size
        }
    }
}
```

值得一提的是，在 GC 复制算法中把对象从 From 空间复制到 To 空间后，可以复用 From 空间中原对象的域来保存 forwarding 指针，但是在 Lisp2 算法中，是在同一个空间中移动对象，因此要事先（即**在移动对象前**）将各个对象的指针全部更新为预计要移动到的地址，如果在移动后将forwarding指针存在原副本的域中，在后续移动时可能出现将其覆盖的情况。

---
## 优点
* 可有效利用堆。

---
## 缺点
* 整理阶段要进行三次遍历，计算成本大。
* 每个对象需要额外的空间来记录 forwarding 指针。

---
# Two-Finger 算法
Two-Finger 算法属于任意顺序整理算法，且最佳适用场景为固定大小对象的堆。该算法使用两个指针，指针 $free 从堆首向后移动，指针 live 从堆尾向前移动，就像两只手指逐渐靠拢，故得名“Two-Finger”。

Two-Finger 算法的整理阶段由两次遍历完成：
1. 第一次遍历，**移动对象**
2. 第二次遍历，**更新指针**

我们来看一下代码，这里假设堆上对象大小固定为 `OBJ_SIZE`：
```c++
two_finger_compact() {
    move_obj()
    adjust_ptr()
}

move_obj() {
    // 使用 $free 和 live 从两端向中间遍历，$free 用于查找空闲空间，live 用于查找活动对象
    $free = $heap_start
    live = $heap_end - OBJ_SIZE
    while (TRUE) {
        // $free 指针跳过活动对象
        while ($free.mark == TRUE) {
            $free += OBJ_SIZE
        }
        // live 指针跳过非活动对象
        while (live.mark == FALSE) {
            live -= OBJ_SIZE
        }
        // 此时 $free 指向空闲区域，live 指向活动对象，如果此时两个指针尚未交错，则将活动对象移到前面的空闲区域
        if ($free < scan) {
            copy_data($free, live, OBJ_SIZE)
            // 将活动对象的forwarding指针指向移动后的对象
            live.forwarding = $free
            live.mark = FALSE
        } else {
            break
        }
    }
}

adjust_ptr() {
    for (r: $roots) {
        // 当移动结束时，$free 指针指向空闲分块的开头
        // 这时 $free 指针右边包含两类对象：非活动对象和移动前的对象
        // 因此，指向 $free 指针右边地址的指针引用的是移动前的对象，所以要更新原来的引用
        if (*r >= $free) {
            *r = (*r).forwarding
        }
    }
    // 移动后，活动对象已经都被压缩到 $heap_start 到 $free 之间了
    scan = $heap_start
    while (scan < $free) {
        scan.mark = FALSE
        for (child: children(scan)) {
            if ((*child) >= $free) {
                *child = (*child).forwarding
            }
        }
        scan += OBJ_SIZE
    }
}
```

Two-Finger 算法完成一次 GC 后，$free 指针指向空闲分块的开头，因此之后再为新对象分配空间时可以直接进行分配操作。

---
## 优点
* 简单快速，整理阶段遍历过程只有两次，相对较少。
* 无需额外空间来记录 forwarding 指针，因为 forwarding指针是在移动完对象后放在原对象域中的，不需要单独占用空间，也不会导致信息丢失。

---
## 缺点
* 采用任意顺序，不考虑对象间的引用关系，破坏了访问局部性。
* 受制于“所有对象大小最好一致”的条件，如果对象大小不一致，在移动过程中还是难以避免产生碎片。

---

# 其他算法
除了上面提到的 Lisp2 算法和 Two-Finger 算法外，还有很多其他的标记-整理算法，比如：
* 表格算法
* Immix GC 算法
* 引线算法
* 单次遍历算法

这些算法相对而言更复杂一点，这里就不详细展开了，我自己也只是大概了解了一下。感兴趣的话可以看看《垃圾回收的算法和实现》和《垃圾回收算法手册：自动内存管理的艺术》这两本书。

---
# 标记-整理算法的局限性

摘取一段《垃圾回收算法手册》的笔记：
> Jones [1996] 的第 5 章对许多较老的整理策略进行了较为详尽的描述，这些算法大多都存在一些不良的或者不可接受的缺陷。需要考虑的问题包括转发指针要付出多大的空间开销。某些整理算法对赋值器具有特定限制：双指针（注：Two-Finger）算法等简单整理算法只能处理固定大小的对象，将对象依照大小进行分级当然是可以的，但既已如此又何须进行整理；引线整理要求能够将指针和暂存在指针域中的非指针临时值进行区分，由于引线算法会（临时性地）破坏针域信息，因而不适用于并发回收器；Morris [1978, 1979, 1982] 的引线整理算法对引用可能的指向方向存在限制；最后，除了双指针算法，大多数整理算法都不支持内部指针。

由此可见，关于算法的研究真是永无止境呀~

---

关于垃圾回收算法的几个基本类型和典型算法的梳理大概就到这里，之后打算整理个汇总比较。自从发现了《垃圾回收算法手册》这本书宛如打开了新世界的大门，不愧为**手册**，大而全。最新版是 2016 年的，不过中文版好像还是翻译的 2012 年的原版。不过算法嘛，时效性不是那么重要啦，毕竟经典的算法都是好几十年前的啦。

日常吐槽：就是最近一个月都在学习和整理垃圾回收的内容，感觉有点疲劳了 \_(:з」∠)\_

---

**参考资料**
* *垃圾回收的算法与实现*
* *垃圾回收算法手册：自动内存管理的艺术*

