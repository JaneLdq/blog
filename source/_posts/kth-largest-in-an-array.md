---
title: Kth Largest Element in an Array
date: 2018-08-18 17:22:04
categories: 算法
tags:
- 数组查询
---

**问题**：给定一个数组，找出这个数组中第k大的元素。

这是一道非常经典的算法题，总结一下几种典型解法。

# 将数组排序，取第k个
最容易想到的解法，关键在排序算法的选择。
在Java中可以偷懒直接调Arrays.sort方法，时间复杂度是`O(NlgN)`，空间复杂度是`O(1)`
```java
public int findKthLargest(int[] nums, int k) {
    int n = nums.length;
    Arrays.sort(nums);
    return nums[n-k];
}
```

<!--more-->

---
# 使用优先队列
使用一个优先队列保存前k大的值，时间复杂度`O(NlgK)`，空间复杂度`O(K)`。
```java
public int findKthLargest(int[] nums, int k) {
    // Java中的优先队列是最小堆
    PriorityQueue<Integer> pq = new PriorityQueue<>();
    for (int val: nums) {
        pq.add(val);
        // 只保存前k个元素
        if (pq.size() > k) {
            pq.poll();
        }
    }
    return pq.peek();
}
```

---
# 使用基于分治的选择算法
这个算法在最优情况下时间复杂度为`O(N)`，最坏情况为`O(N^2)`，空间复杂度为`O(1)`
```java
public int findKthLargest(int[] nums, int k) {
    // 从小到大排序，取整数第n - k个
    int p = nums.length - k;
    int lo = 0, hi = nums.length - 1;
    while(lo < hi) {
        int m = partition(nums, lo, hi);
        if (m < p) {
            lo = m + 1;  // 在后半部分查
        } else if (m > p) {
            hi = p - 1;  // 在前半部分查
        } else {
            break;
        }
    }
    return nums[p];
}

private int partition(int[] nums, int lo, int hi) {
    int i = lo, j = hi + 1;
    // 这里使用nums[lo]作为pivot
    int pivot = nums[lo];
    while(true) {
        while(i < hi && nums[++i] < pivot);
        while(j > lo && nums[--j] > pivot);
        if (i >= j) {
            break;
        }
        // nums[j] <= nums[lo] 总是成立
        swap(nums, i, j);
    }
    // 交换pivot到它应该在的位置
    swap(nums, lo, j);
    return j;
}

private void swap(int[] nums, int i, int j) {
    int temp = nums[i];
    nums[i] = nums[j];
    nums[j] = temp;
}
```

基于上面这个算法，其实还可以进一步优化避免出现O(N^2)最坏情况：在排序之前对打乱数组就好啦。
在以上代码中引入如下“洗牌”算法，并在最开始调用它：
```java
private void shuffle(int[] nums) {
    int len = nums.length;
    for (int i = 0; i < nums.length; i++) {
        int pos = (int)(Math.random() * (len - i));
        swap(nums, pos, len - i - 1);
    }
}
```

---

这几个算法主要涵盖了`快速排序`、`快速选择`和`堆排序`相关的知识点，正好放在一起，可以比较一下~
