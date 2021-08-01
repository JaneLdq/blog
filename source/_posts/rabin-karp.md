---
title: 字符串搜索玩耍指南（三）- Rabin-Karp算法
date: 2018-10-12 20:13:08
categories: 算法
tags: 
- Rabin-Karp
- 字符串搜索
---

前面介绍的KMP和Boyer-Moore算法都是基于最基本的字符比较来搜索字符串，这里要介绍的第三种经典算法，其思路与前者完全不同，它就是——Rabin-Karp算法，一个**基于Hashing进行字符串搜索的算法**。

*Rabin-Karp算法主要由两位大师在1987年提出，一位是以色列的[Michael O. Rabin][1]，一位是美国的[Richard M. Karp][2]。因为其内部哈希函数的实现多采用Rabin提出的[Rabin fingerprint][3]算法。Rabin-Karp算法又被称为"the Fingerprint Method"。*

**基于Hashing搜索字符串**，这么简单的一句话，就涵盖了Rabin-Karp算法的主要思想。

给定长度为M的目标字符串pattern和长度为`N`的文本`text`，采用**某种Hash函数**计算出`pattern`的`Hash`值`P`，然后遍历`text`，计算每段长度为`M`的子串的Hash值`Q`，如果`P等于Q`，那么就匹配成功。

这里面最重要的问题就是如何选择Hash函数：

1. 如果Hash函数本身的计算成本太高，也会降低效率；
2. 如果Hash函数的冲突率太高，那么对于两个Hash值相同的字符串我们并不敢相信它们是相同的，最纵横还是会需要挨个进行字符比较，Hash值也就失去其意义所在，最差的情况下时间复杂度会达到O(MN)，甚至比暴力解决更慢（因为计算Hash值也要时间）。

<!--more-->

---
## Hashing效率是关键
我们可以将长度为`M`的字符串`pattern`映射为一个`M`位的以`R`为底(M-digit base-R number)的数值`X`，那么一种最基本的取其Hash值的方式就是选取一个素数`Q`，然后取`X mod Q`为`pattern`的Hash值。

代码如下：
```java
public long hash(String key, int M) {
    long h = 0;
    for (int i = 0; i < M; i++) {
        // M-digit base-R number的值，可能非常大，所以这里应用了一个小技巧
        // 借用霍纳规则和余数的特性
        h = (R * h + key.charAt(i)) % Q;
    }
    return h;
}
```

显然，上面这个Hash函数函数是关键
一般想到求字符串的Hash值，大多要用到字符串的每个字符，对于长度为M的字符串而言，这就需要O(M)的时间开销。在最坏的情况下，字符串搜索会消耗O(MN)的时间。

这显然不是我们想要的。

---

Rabin-Karp算法之所以被称之为Rabin-karp，正式因为Rabin和Karp找到了一种只需常数时间开销的Hash函数，使得搜索能在线性时间内完成。

这个算法的思路看懂之后也是真的很精简，就两个公式。

我们假设 **t<sub>i</sub>** 表示`text.charAt(i)`，那么从`text`的位置`i`开始长度为`M`的字符串对应的以`R`为底的数值可以用如下公式表示：

> ***x*<sub>i</sub> = *t*<sub>i</sub>*R*<sup>M-1</sup> + *t*<sub>i+1</sub>*R*<sup>M-2</sup> + ... + *t*<sub>i+M-1</sub>*R*<sup>0</sup>**

那将`i`向前挪一步，可得：
> ***x*<sub>i+1</sub> = (*x*<sub>i</sub> - *t*<sub>i</sub>*R*<sup>M-1</sup>)*R* + *t*<sub>i+M</sub>*R*<sup>0</sup>**

定义**x<sub>i</sub>**的Hash值：
> **h(*x*<sub>i</sub>) = *x*<sub>i</sub> mod *Q***

那么，利用取余的几个等价关系，我们并不需要保留这些值(x<sub>i</sub> for i = 0, 1, ...)，只保留h(x<sub>i</sub>)就可以了。（记不清余数定理可以看[Wiki - Modulo Operation Equivalencies][4]回顾一下。）

---
## 代码实现
如果觉得还差了点直观感受，可以看看下面的实现代码，加深一下理解。
```java
public class RabinKarp {

    private static final int R = 256;

    private final String pattern;

    private final long patternHash;

    private final int M;

    private final long Q;

    private long RM; // R^(M-1) % Q

    public RabinKarp(String pattern) {
        this.pattern = pattern;
        M = pattern.length();
        Q = longRandomPrime();
        RM = 1;
        for (int i = 1; i <= M - 1; i++) {
            RM = (R * RM) % Q;
        }
        patternHash = hash(pattern, M);
    }

    public int search(String txt) {
        int n = txt.length();
        long txtHash = hash(txt, M);
        if (patternHash == txtHash) return 0;
        for (int i = M; i < n; i++) {
            // 向前移动一位，先移除前一位字符
            txtHash = (txtHash + Q - RM * txt.charAt(i - M) % Q) % Q;
            // 加上最后新增的的一位字符
            txtHash = (txtHash * R + txt.charAt(i)) % Q;
            // check for match
            if (patternHash == txtHash) {
                if (check(txt, i - M + 1)) return i - M + 1;
            }
        }
        return n;
    }
    
    /**
     * Monte Carlo version - 允许一定的误差，不做进一步的字符串比较
     * Las Vegas version - 在Hash值相同的情况下，进行挨个字符的比较
     */
    public boolean check(String txt, int i) {
        // Las Vegas
        // for (int j = 0; j < M; j++) {
        //   if (pattern.charAt(j) != txt.charAt(i + j)) return false;
        // }
        return true;  // Monte Carlo Version
    }

    private long hash(String key, int M) {
        long h = 0;
        for (int j = 0; j < M; j++) {
            h = (R * h + key.charAt(j)) % Q;
        }
        return h;
    }

    private static long longRandomPrime() {
        BigInteger prime = BigInteger.probablePrime(31, new Random());
        return prime.longValue();
    }

}
```

### Monte Carlo v.s. Las Vegas
认真开代码的朋友会发现，代码中提供了一个`check`方法，注释里提到了两个版本**Monte Carlo**和**Las Vegas**。这两个赌城的名字在这里分别代表了两种随机算法：

* [Monte Carlo Alogrithms][5]：尽量找到好的，但不保证是最好的
* [Las Vegas Algorithms][6]：尽量找到最好的，但不保证能找到

对于上面的实现来说，如果我们选择的Q的值非常大，比如10<sup>20</sup>，这就表示在Hash值相等时出现冲突的概率为10<sup>-20</sup>，如果你能接受这个误差，那么就可以选择Monte Carlo版本，以保证线性时间复杂度。如果这是应用在一个要求非常严谨的程序中，那么你就可以选择Las Vegas版本，牺牲时间来确保正确。

PS：对这两个名字感到好奇，上网查了一下，挺有意思：
> *“这两个名字来源于两个著名的赌城，因为不完全采样的随机算法，就是在赌自己能在有限的采样次数内命中最优解或者足够接近最优解的解。”*

---

关于字符串搜索的系列到这里就暂时告一段落啦，太多细枝末节的东西也没有深究，还是加上“入门”两个字比较合适～

---
参考资料：

* [Wikipedia - Rabin-Karp Alogrithm][7]
* [蒙特卡罗算法是什么？][8]


  [1]: https://en.wikipedia.org/wiki/Michael_O._Rabin
  [2]: https://en.wikipedia.org/wiki/Richard_M._Karp
  [3]: https://en.wikipedia.org/wiki/Rabin_fingerprint
  [4]: https://en.wikipedia.org/wiki/Modulo_operation#Equivalencies
  [5]: https://en.wikipedia.org/wiki/Monte_Carlo_algorithm
  [6]: https://en.wikipedia.org/wiki/Las_Vegas_algorithm
  [7]: https://en.wikipedia.org/wiki/Rabin%E2%80%93Karp_algorithm
  [8]: https://www.zhihu.com/question/20254139