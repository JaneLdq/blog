---
title: Maximum Subarray Problem & Kadane's Algorithm
date: 2018-08-08 22:19:16
categories: 算法
tags: 数组查询
---

**问题**：给定一个数组,找到一个连续的子数组,使得这个子数组的和最大。
举个例子，`[1, -3, 2, 1, -1]`的最大子数组是`[2, 1]`，其和为**3**。

# Brute-force Solution
最容易想到的就是暴力解法，遍历所有可能的子数组，时间复杂度为O(n^2)。
<!--more-->

```java
public int maxSubArray(int[] nums) {
    int globalMax = Integer.MIN_VALUE;
    for (int i = 0; i < nums.length; i++) {
        int currentMax = 0;
        for (int j = i; j < nums.length; j++) {
            currentMax += nums[j];
            globalMax = Math.max(globalMax, currentMax);
        }
    }
    return globalMax;
}
```

---

# Kadane's Algorithm
暴力虽然能解决问题，但我们还是要更优雅~
（这道题在leetcode上被标为easy，看题的时候提示有O(n)的解法，结果想了老半天也没解出来，唉o(╥﹏╥)o

Kadane算法可以优化到O(n)，怎么做到的呢？

Kadane算法被归为动态规划，终于有勇气揭开DP神秘的面纱~

> 动态规划(Dynamic programming)是一种通过把原问题分解为相对简单的子问题的方式求解复杂问题的方法。—— 维基百科

两种主要的DP实现方式：
* **Tabulation** - 自底向上(Bottom Up)解决问题
* **Memoization** - 自顶向下(Top Down)解决问题

这部分初次接触，还不太了解，先存几个Link:
* [Tutorial For Dynamic Programming][1]
* [Tabulation vs Memoizatation][2]
* [StackOverflow: Dynamic Programming and Memoization: bottom-up vs top-down approaches][3]

## Kadane算法思路
既然动态规划的中心思想就是分解问题，所以Kadane算法中至关重要的一步就是定义合适的子问题，并找到问题与子问题间的关系。
Kadane算法将子问题定义为：**求以第i个元素结尾的最大子数组**。
什么意思？举个例子：
假设有数组A:`[a, b, c, d, e]`，则：
* 以第0个元素结尾的最大子数组就是它自己：`[a]`
* 以第1个元素结尾的最大子数组可能是这些中的一个：`[a, b]`, `[b]`
* 以第2个元素结尾的最大子数组可能是这些中的一个：`[a, b, c]`, `[b, c]`, `[c]`

考虑**以第`i`个元素结尾**的子数组，一定是**以第`i-1`个元素结尾的子数组**加上**第`i`个元素**组成的，由此可以推导出如下关系：

```java
maxSum[0] = A[0]; // i = 0
maxSum[i] = A[i] + （maxSum[i-1] > 0 ? maxSum[i-1] : 0); // i > 0
```

也就是说，如果以第i-1个元素结尾的最大子数组和大于0，那么以第i个元素结尾的最大子数组就等于它前面的那部分加上它自己，否则就舍弃前面的部分，只剩它自己。

回到问题，maxSum这个数组中的每个元素就代表了以当前位置结尾的最大子数组和，取maxSum中的最大值就是整个数组的最大子数组和啦。

Kadane算法的完整实现如下：

```java
public int maxSubArray(int[] nums) {
    int[] maxSum = new int[nums.length];
    maxSum[0] = nums[0];
    int max = nums[0];
    for (int i = 1; i < nums.length; i++) {
        maxSum[i] = nums[i] + Math.max(0, maxSum[i-1]);
        max = Math.max(maxSum[i], max);
    }
    return max;
}
```

最后安利一个把MSP讲解得非常清晰的视频~[Kadane's Algorithm to Maximum Sum Subarray Problem][4]

---
# 参考资料
* [Maximum Subarray Problem - Wiki][5]
* [Finding the maximum contiguous sum in an array and Kadane’s algorithm][6]


  [1]: https://www.codechef.com/wiki/tutorial-dynamic-programming#Introduction
  [2]: https://www.geeksforgeeks.org/tabulation-vs-memoizatation/
  [3]: https://stackoverflow.com/questions/6164629/dynamic-programming-and-memoization-bottom-up-vs-top-down-approaches
  [4]: https://www.youtube.com/watch?v=86CQq3pKSUw
  [5]: https://en.wikipedia.org/wiki/Maximum_subarray_problem
  [6]: https://medium.com/@marcellamaki/finding-the-maximum-contiguous-sum-in-an-array-and-kadanes-algorithm-e303cd4eb98c