---
title: Floyd's Cycle-Finding Algorithm
date: 2019-01-13 15:33:29
categories: 算法
---

9102年来啦，怎么还是随手一戳就是盲点呢_(:з」∠)_

---

**问题**：给定一个链表(Linked List)，请判断这个链表中是不是有环(Cycle)。

在第一次遇到这个问题时，最容易想到的无脑解法就是挨个遍历节点，并将访问过的节点放入一个集合中，如果下一个访问的节点已经在这个集合中存在了，那么就说明环存在，绕回来了。
```java
/**
 * 链表节点的定义
 * class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) {
 *         val = x;
 *         next = null;
 *     }
 * }
 */
public ListNode detectCycle(ListNode head) {
    ListNode cursor = head;
    Set<Integer> nodes = new HashSet<>();
    while(cursor != null) {
        if (nodes.contains(cursor)) return true;
        nodes.add(cursor);
        cursor = cursor.next;
    }
    return false;
}
```
然而这种既耗空间又耗时间的解法怎么可能满足我们的高要求呢？能不能用O(1)空间解决这个问题呢？
<!--more-->
在学到如下这个算法之前我是一脸蒙圈的，看懂之后的我：为什么天才都如此机智......QAQ

---
# Floyd cycle-finding algorithm
关于这个算法的由来，在wiki上找到了如下一段介绍：
> [Robert W. Floyd][1] describes algorithms for listing all simple cycles in a directed graph in a 1967 paper, but this paper does not describe the cycle-finding problem in functional graphs that is the subject of this article. In fact, Knuth's statement (in 1969), attributing it to Floyd, without citation, is the first known appearance in print, and it thus may be a folk theorem, not attributable to a single individual.

这个算法又被称为**the tortoise and the hare algorithm**，听到这个名字，会不会一下子有了思路呢？
正如龟兔赛跑，算法的主要思路就是**使用两个前进速度不同的指针来找出是否存在环，以及环的起始位置**。

## 是否存在环？
一个像乌龟爬的慢，每次只走一步，一个像兔子跑得快，每次走两步。两个指针同时从起点出发，如果链表存在环，那么总有一个时刻他俩会相遇，如果链表不存在环，他们就不可能会相遇啦。**
真是，大道至简啊。
代码实现如下：
```java
public boolean hasCycle(ListNode head) {
    if (head == null || head.next == null) return false;
    ListNode tortoise = head;
    ListNode hare = head;
    while(hare != null && hare.next != null) {
        tortoise = tortoise.next;
        hare = hare.next.next;
        if (tortoise == hare) return true;
    }
    return false;
}
```

---
## 如何定位环？
如果在原问题的基础上再追加一问：**如果链表中存在环，请找到进入环的起始位置**。这就涉及到龟兔赛跑算法的另一大关键点。

我们先做如下设定：
* 乌龟`T`和兔子`H`同时从起点`A`出发，相遇的点在`B`（B是环上某一点），环开始的位置（从链表头部算起进入环的位置）为`C`（也就是我们要找到的位置）
* 相遇时乌龟走了`k`步；兔子走了`2k`步
* 设|AC| = a, |CB| = b, |BC| = c，我们取最简单的情况，也就是乌龟第一次进入环，兔子完整地跑完了一次环

```
A                C           B
|------- a ------|---- b ----|
                 |           |
                  ---- c ----
```

那么我们可以有如下公式：
> a + b = k
> a + b + b + c = 2k 
> => a = c

所以要定位`C`就变得很简单了，首先找到首次相遇的点`B`，然后分别`A`和`B`同步前进，遇到的第一个相同的点就是环开始的地方啦~

代码如下所示：
```java
public ListNode detectCycle(ListNode head) {
    if (head == null || head.next == null) return null;
    ListNode tortoise = head;
    ListNode hare = head;
    boolean hasCycle = false;
    while (hare != null && hare.next != null) {
        tortoise = tortoise.next;
        hare = hare.next.next;
        if (tortoise == hare) {
            hasCycle = true;
            break;
        }
    }
    if (!hasCycle) return null;
    tortoise = head;
    while (tortoise != hare) {
        tortoise = tortoise.next;
        hare = hare.next;
    }
    return tortoise;
}
```

留个小思考：上面的推理过程是简化过的，情况可能会更复杂，如果乌龟与兔子相遇时各绕环走了r1和r2圈，那结果会怎么样呢？


  [1]: https://en.wikipedia.org/wiki/Robert_W._Floyd