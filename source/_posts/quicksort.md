---
title: 快速排序避坑指南
date: 2018-08-21 10:21:21
categories: 算法
tags: 
- 排序
---

上一篇笔记里提到了快速排序，顺便翻了翻书复习了一遍，关于算法的实现和优化其实有不少需要留意的细节，写篇笔记巩固一下。

快速排序这个名字也是非常直白了，它的平均运行时间为**O(NlogN)**，当然了，最坏的情况也可能达到**O(N^2)**，不过使用一些小技巧就可以极大避免这种极端情况的出现，比如上一篇笔记里提到的“洗牌”。

快速排序本质上属于一种分治的递归算法，将数组`S`排序的基本算法由以下几步组成：
1. 如果`S`中元素个数是0或1，返回；
2. 取`S`中任一元素`v`作为`pivot`；
3. 将`S`中除`v`以外的元素划分为两个不相交集合：大于等于v(`S1`)和小于等于v的(`S2`)；
4. 分别`S1`和`S2`进行快速排序得到`S1'`，`S2'`，返回`S1'`接上`v`接上`S2'`。

看起来很简单，实现也不难，但是很有可能一不小心就导致算法出错或者性能很差哦。下面我们就来看看有哪些需要注意的地方吧～

<!--more-->
---
# Pivot怎么选？
虽然上面的说法是取任一元素作为`pivot`，这样做虽然能最终达到排序的目标，但实际上`pivot`的选择会对算法的性能产生不小的影响，也是一项技术活哦。
常见的选择可能有如下几种：

* **将第一个元素作为pivot**
  如果输入数组是随机的，那么总是选第一个是可以接受的。然而如果数组是预排序的或是反序的，那么这样选择`pivot`就会产生一个非常差的分割，而且这糟糕的情况会发生在所有递归中。所以，**一定不要这样做！**

* **随机选取pivot**
  随机选取pivot是一种非常安全的策略，不过由于生成随机数的开销也比较大，也减少不了算法其余部分的平均运行时间。

* **三数中值分割法** ★
  pivot最好的选择是数组的中值，但是要找到中值本身就会减慢排序的速度，可以通过随机选取三个元素取它们的中值作为数组中值的估计值。考虑到随性并没有太大的帮助，一般的做法是**选择数组最左端，最右端和中间位置上的三个元素作为`pivot`**。

```java
private static <T extends Comparable<? super T>> T findPivot(T[] arr, int lo, int hi) {
    int mid = (lo + hi) / 2;
    if (arr[mid].compareTo(arr[lo]) < 0)
        Util.swap(arr, lo, mid);
    if (arr[hi].compareTo(arr[lo]) < 0)
        Util.swap(arr, lo, hi);
    if (arr[hi].compareTo(arr[mid]) < 0)
        Util.swap(arr, mid, hi);
    // 将倒数第二个元素与pivot替换，因为经过以上比较，arr[hi]一定大于枢纽元，arr[lo]一定小于pivot
    Util.swap(arr, mid, hi-1);
    return arr[hi-1];
}
```
注意上面这段代码，其实还做了一小步优化。三个元素经过比较和交换后，`arr[lo]`已经是小于`pivot`的元素，`arr[hi]`已经是大于`pivot`的元素，对于这一轮快速排序，它们已经在正确的分割里了，而且`pivot`也被交换到了`arr[hi-1]`，后续只需要对`[lo+1, hi-2]`这个区间里的元素进行分割。

---
# 分割策略如何定？
分割是一种很容易出错或导致低效的操作，其实主要就是边界情况的考虑。在快速排序中一种需要谨慎考虑的情况就是**如何处理与pivot相等的元素**。（这个我在自己实现的时候深有体会啊，一不留神就掉坑里了...）

首先，无论我们如何处理相等的元素，一般第一步都是**把pivot与数组最后的元素进行交换，把pivot与要进行分割的数据段隔离开**。

我们先讨论数组中不包含相等元素的情况：
用两个指针`i`, `j`分别从第0个元素和第n-2个元素向中间移动，当`i < j`时，将`i`右移，移过那些小于`pivot`的元素，将`j`左移，移过大于`pivot`的元素。当`i`和`j`停止时，如果`i < j`则将这两个元素互换。如此直到`i`和`j`交错。

最后一步将`pivot`与`i`所指的元素交换。（这里稍微分析以下就可以确定最终i停下的位置一定是大于`pivot`的）

下面考虑如何处理相等的情况，问题关键在于遇到与`pivot`相等的元素时，`i`和`j`是否停止？
有如下三种可能：

1. **`i`和`j`都停下**★ ：如果数组中元素全部相同，那么这会导致相等的元素要进行很多次交换。但是它的好处在于i和j将在中间交叉，这种分割将建立两个几乎相等的数组。
2. **`i`和`j`只有一个停下**：这会导致分割非常不均衡，因为相等的元素都会被划分到一边。
3. **`i`和`j`都不停下**：如果都不停下，就要考虑`i`和`j`越界的情况，而且实现的时候要精确定位i最终到过的位置与`pivot`进行交换。这个判断由于要考虑越界的情况，会相对复杂。而且产生的分割不平衡。在所有元素都相同的情况下，时间复杂度为O(N^2)。

对比以上三种情况，相比于得到两个不平衡子数组，我们宁愿多进行几次交换获得相对平衡的分割。所以我们选择**`i`和`j`都停下**这种策略。

---
# 小数组怎么处理？
对于小数组，快速排序的效果不如插入排序。而且快速排序作为递归算法，对小数组进行排序是无可避免的。
通常的解决办法是对于小数组不使用递归的快速排序，而选择比如插入排序这样对小数组有效的排序算法。
那么如何界定小数组呢？**一般选择5-20的范围，10就不错**。
通过对小数组单独处理，还能避免在进行三数中值比较时出现只有一到两个元素的情况。（亲测这也是个坑啊...如果不处理小数组，那么取出来的`pivot`可能位于第0或1的位置，一不小心就越界了...）

---
# 快速排序的实现
综上所述，最终实现的快速排序代码如下：

```java
public class QuickSort {

    private static int N = 10;
    
    public static <T extends Comparable<? super T>> void sort(T[] arr) {
        quicksort(arr, 0, arr.length-1);
    }

    private static <T extends Comparable<? super T>> void quicksort(T[] arr, int lo, int hi) {
        // 对大数组使用快速排序
        if (hi - lo >= N) {
            if (lo < hi) {
                T pivot = findPivot(arr, lo, hi);
                // 这里i和j分别初始化为lo和hi-1，如前所述lo和hi-1的位置上已经是正确划分的值
                // 下面循环里的[++i]和[++j]在取值之前，先自加自减，相当于从lo+1和hi-2的位置开始比较
                // 这里还有一点需要注意的，如果不区别对待小数组，得到的j的值有可能就是0，这时[--j]就会报数组越界啦
                int i = lo, j = hi-1;
                while(true) {
                    // 
                    while (arr[++i].compareTo(pivot) < 0) {}
                    while (arr[--j].compareTo(pivot) > 0) {}
                    if (i > j)
                        break;
                    Util.swap(arr, i, j);
                }
                Util.swap(arr, i, hi-1);
                quicksort(arr, lo, i-1);
                quicksort(arr, i+1, hi);
            }
        } else {
            // 对小数组使用插入排序
            InsertionSort.sort(arr);
        }
    }
    
    private static <T extends Comparable<? super T>> T findPivot(T[] arr, int lo, int hi) {
        int mid = (lo + hi) / 2;
        if (arr[mid].compareTo(arr[lo]) < 0)
            Util.swap(arr, lo, mid);
        if (arr[hi].compareTo(arr[lo]) < 0)
            Util.swap(arr, lo, hi);
        if (arr[hi].compareTo(arr[mid]) < 0)
            Util.swap(arr, mid, hi);
        // 将倒数第二个元素与枢纽元替换，因为经过以上比较，arr[hi]一定大于枢纽元，arr[lo]一定小于枢纽元
        Util.swap(arr, mid, hi-1);
        return arr[hi-1];
    }

}
```

思考一下：如果把上面的16-24行换成下面这几行会发生什么？

```java
int i = lo + 1, j = hi;
while(true) {
    while(arr[i].compareTo(pivot) < 0) i++;
    while(arr[j].compareTo(pivot) > 0) j--;
    if (i < j) 
        Util.swap(arr, i, j);
    else 
        break;
}
```
如果这样写，当`arr[i]=arr[j]=pivot`的时候，就会死循环啦，一定要小心哦～（别问我为什么知道的，凭实力翻车〒▽〒

事实证明前辈们总结出来的经验和小技巧小细节都是非常容易出错的地方，我自己实现的时候差不多把这些坑都踩了个遍，好了，我去面壁思过了。

