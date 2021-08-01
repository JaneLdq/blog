---
title: 0-1 Knapsack Problem
date: 2019-02-28 23:56:08
categories: 算法
---

赶在正月的尾巴上更一篇<del>强行</del>新年主题的笔记好了～

---

问题：新年到啦，又到了收获红包的节日，可是小猪猪的口袋有限，最多只能装下总重量不超过$W$的红包们。假设今年七大姑八大姨们一共准备了$n$个红包，每个红包的重量为$w_{i}$，价值为$v_{i}$。小猪猪应该挑哪些红包才能最合理地利用口袋的容量，获得价值最高的红包总额呢？

如果用数学公式来描述这个问题，可以将其转换为如下定义：

给定两个长度为$n$的正整数数组，分别为$\left[w_1, w_2, ..., w_n \right]$和$\left[v_1, v_2, ..., v_n \right]$, 以及一个最大容积$W\gt0$，我们想
$$最大化 \sum_{i=0}^n v_{i}x_{i}, 且有 \sum_{i=0}^n w_{i}x_{i} \leq W, x_{i} \in \lbrace0, 1 \rbrace$$

对于这$n$个物品，每个都有两个状态，放进背包或者被舍弃，$x_{i}=0$表示舍弃，$x_{i}=1$则表示获取。0-1 Knapsack中的0-1也是由此而来啦～

那么，要如何求解呢？
<!--more-->

---
## Brute-force
暴力解法就是从1到n依次尝试0和1两种状态：

```java
public static int bruteForceKnapsack(int[] weights, int[] values, int W, int n) {
    if (n == 0 || W == 0) return 0;
    // 如果超出总重量，则直接跳过不予考虑
    if (weights[n-1] > W)
        return bruteForceKnapsack(weights, values, W, n-1);
    // 不放进背包
    int zero = bruteForceKnapsack(weights, values, W, n-1);
    // 放进背包
    int one = values[n-1] + bruteForceKnapsack(weights, values, W-weights[n-1], n-1);
    return Math.max(zero, one);
}
```
Brute-force解法的时间复杂度为$O(2^n)$，呈指数增长。

举个例子，$n=3, weights=\left[1,2,1\right], values=\left[10, 15, 25\right], W=3$：
```
 // 用K(W, n)简略表示bruteForceKnapsack(weights, values, W, n)
 
                            K(3, 3)
                          /         \
            K(3, 2)                           K(2, 2)
            /     \                            /     \
      K(3, 1)      K(1, 1)               K(2, 1)     K(0, 1)
      /     \      /     \               /     \      
K(3, 0)  K(2, 0)  K(1, 0) K(0, 0)    K(2, 0)   K(1, 0)

// 这个递归中，K(1, 0)和K(2, 0)都计算了两次
```

不难看出，上述递归算法将问题划分为子问题的过程中，出现了重复的子问题，从而导致重复计算，降低了效率。

---

针对子问题重复的情况，我们可以使用动态规划进行优化。

## 使用动态规划求解
动态规划与分治法类似，都是把大问题拆分成小问题，通过寻找大问题与小问题的递推关系，一个个解决小问题，最终达到解决原问题的效果。不同的是，动态规划**使用表将已经解决的子问题记下来，在遇到重复子问题时直接提取**，避免了重复计算，从而节约了时间（实际上也是在用空间换时间了）。

### 动态规划求解思路
1. **Structure** 找到最优解之间的关系：将原始问题分解为子问题，找到原始问题的最优解与子问题的最优解之间的关系
2. **Principle of Optimality** 递推地定义最优解的值：用子问题的最优解来表示原始问题的最优解
3. **Bottom-up computation**：借助表（存储子问题最优解）自底向上计算最优解

---

现在我们就按这个步骤来设计0-1 Knapsack问题的动态规划解法：

### Step 1 - 将原始问题划分为小一点的子问题
我们可以构建一个二维数组$V[n+1, W+1]$，对于$V[i][w], 1 <= i <= n, 0 <= w <= W $保存的是$n$个红包的子集$\lbrace 1, 2, ... i \rbrace$在最大容积为$w$的约束下能得到的最大值。那么$V[n][W]$就是原始问题的最优解了。

### Step 2 - 递归求解

* **初始条件**：
$$V[0][w] = 0 \space for \space 0 \lt w \leq W$$ $$V[i][0] = 0 \space for \space i \leq 0 \leq n, w = 0$$

表的初始状态：

|-|0|1|2|...|W|-|
|-|-|-|-|-|-|-|
|0|0|0|0|0|0|bottom|
|1|0| | | | |↓|
|...|0| | | | |↓|
|n|0| | | | |up|


* **递归计算**：
$$V[i][w] = max(V[i-1][w], v_{i} + V[i-1][w-w_{i}])$$$$for \space 1 \leq i \leq n, 1 \leq w \leq W $$
也就是两种情况：1）不拿第i个红包，则能拿到的最大值为在$\lbrace1, 2, ..., i-1\rbrace$取最大容积为$w$的解；2）拿第i个红包，则能拿到的最大值为$v_{i}$加上在$\lbrace1, 2, ..., i-1\rbrace$取最大容积为$w-w_{i}$的解。两种情况中的较大值为当前子问题的最优解。

### Step 3 - 自底而上计算$V[i, w]$
也就是从$i=1,...,n$开始依次计算每行的值。

代码如下：
```java
public static int dpKnapsack(int[] weights, int[] values, int W, int n) {
    if (n == 0 || W == 0) return 0;
    int[][] dp = new int[n+1][W+1];
    for (int i = 1; i <= n; i++) {
        for (int w = 0; w <= W; w++) {
            if (weights[i-1] <= w) {
                dp[i][w] = Math.max(dp[i-1][w], values[i-1] + dp[i-1][w - weights[i-1]]);
            } else {
                dp[i][w] = dp[i-1][w];
            }
        }
    }
    return dp[n][W];
}
```
显然，动态规划算法的时间复杂度为$O(nW)$，比$O(2^n)$快多了。

---

在以上算法的基础上，还可以进一步优化空间复杂度。不难发现，$dp[i][w]$的更新只跟$dp[i-1][w]$，因此，我们只需使用一个一维数组保存遍表中上一行的结果就够了。

优化后的代码如下所示：
```java
public static int optimizedDpKnapsack(int[] weights, int[] values, int W, int n) {
    if (n == 0 || W == 0) return 0;
    int[] dp = new int[W+1];
    for (int i = 1; i < n; i++) {
        // 需要注意的是，内循环中w的遍历是从大到小进行的
        // 否则前一轮循环得到的值就被覆盖了
        for (int w = W; w >= 0; w--) {
            if (w >= weights[i]) {
                dp[w] = Math.max(dp[w-weights[i]] + values[i], dp[w]);
            }
        }
    }
    return dp[W];
}
```

---

本期日常：刚刚看到一篇介绍传奇校友的推文，今夜的我是一颗枯萎的🍋了。

---

* **参考资料**
[The Knapsack Problem - PDF][1]
[GeeksforGeeks: 0-1 Knapsack Problem | DP-10][2]
[动态规划解决01背包问题][3]


  [1]: http://www.es.ele.tue.nl/education/5MC10/Solutions/knapsack.pdf
  [2]: https://www.geeksforgeeks.org/0-1-knapsack-problem-dp-10/
  [3]: https://www.cnblogs.com/Christal-R/p/Dynamic_programming.html