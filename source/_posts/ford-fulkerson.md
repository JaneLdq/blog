---
title: 最大流问题和Ford-Fulkerson方法
date: 2018-09-09 8:12:29
categories: 算法
tags: 
- Ford-Fulkerson
- 最大流问题
---

无论是解决网络流量或是交通流量，最大流问题都是网络流问题中最基本的问题之一。

什么是网络流？很简单：
1. 给定一个有向图G=(V, E)，它的每条边的容量为c(v,w)，对于任意一条边(v, w)最多有c(v, w)个单位的“流”可以通过；
2. 它有两个顶点，一个是s，称为源点(source)，一个是t，称为汇点(sink)，从s发出的流必须等于流入t的流。

最大流问题就是**确定从s到t可以通过的最大流量**。

![flow network][1]
上图是一个简单的网络流图及其最大流的例子，斜杠左边代表经过一条边的流，右边代表边的容量。不难看出，图中最大流为5。

<!--more-->

---

# Ford-Fulkerson方法
在介绍Ford-Fulkerson方法前，先引入几个与网络流相关的概念：

* 流图：从原始的图G构造一个流图Gf，表示当前以达到的流。初始时，所有边的流均为0。
* **残余图(Residual Graph)**：从原始的图G构造一个流图Gr，它表示每条边上还能再添加多少流。边上当前残余的流 = 边容量 - 边上当前通过的流。
* **增广路径(Augmenting Path)**：在图Gr中找到一条从s到t的路径，这就是一条增广路径。一条增广路径上最小的边值（bottleneck）就是可以添加到路径每一条边上的流量。
* **割(Cut)**：定义网络流G=(V, E)的割(S, T)将V划分成S和T=V-S两部分，使得s∈S，t∈T。割的容量为c(S, T)。

Ford-Fulkerson方法是1956年由[L. R. Ford Jr.][2]和
[D. R. Fulkerson][3]提出的一种解决最大流问题的方法。

Ford-Fulkerson的主要思路是这样的：
```java
while (还能找到增广路径) {
    1. 找到一条增广路径
    2. 找到增广路径上最小的流量（bottleneck）
    3. 沿着增广路径增加总流量以及每条路径上通过的流量
}
```

反正我第一次看到这么拗口的概念定义时整个人都是晕的，我们还是举个例子更清晰，下图展示的过程就是基于如上算法确定上图所示最大流的过程：
![maxflow step by step example][4]

看起来很棒棒，然而，假设我们在第一步不是选择的s->a->c->t这条路径，而是选择s->a->d->t会发生什么呢？
![wrong maxflow][5]

这样一来，在添加了3个单位的流后，算法就终止了，并没有找到最优解。
如何解决呢？方法很简单，**对应Gf中的每条边(v, w)，我们都在Gr中添加一条容量为f(v,w)的边(w, v).**相当于，我们通过往相反方向发回一个流来使算法解除它原来的决定。（其实好比人脑中思考的过程，我们发现s->a->d->t不对的时候，会考虑从t->d->a回退，那么反向添加边，就是给算法赋予了在选择下一条增广路径时能够进行回退操作的能力。）
下图为修正后的过程：
![correct maxflow][6]
从上图可以看到，在添加了反向流之后，算法可以在Gr中找到一条包含回退操作的边s->b->d->a->c->t，并注入2个单位的流量，最终得到最大流5.

---
## Edmonds–Karp算法
L. R. Ford Jr.和D. R. Fulkerson只是提出了算法的思想并没有定义如何选取增广路径，由此也诞生了很多基于Ford-Fulkerson的实现。

这里介绍一种由[Jack Edmonds][7]和[Richard Karp][8]在1972提出的实现算法。

Edmonds-Karp算法的基本思想是通过广度优先遍历(BFS)获取增广路径。在BFS的基础上他们又提出了两种选择增广路径的方式：

* Shortest Augmenting Path(SAP)，每次选取的增广路径都是残留图中从s到t的最短路径。（实现代码见下一节）
* Maximum Capacity Augmenting Path， 每次选取的增广路径都是残留图中从s到t容量最大的路径。（选取MCAP的过程参考Dijkstra最短路径算法那，使用优先队列实现）

### SAP算法的实现
下面给出Edmons-Karps算法（采用SAP）的实现：

算法的复杂度是O(VE^2)：

* 基于BFS选取增广路径的复杂度为O(E)；
* Edmonds-Karp对流进行增加的全部次数为O(VE)。证明比较复杂，参见《算法导论》26.2节。


下面是实现代码：
```java
public class FordFulkerson {

    private final int V; // number of vertices

    private boolean[] marked; // marked[v] = true if path s->v in residual network

    private FlowEdge[] edgeTo; // edgeTo[v] = last edge on path s->v

    private double value; // value of max flow

    public FordFulkerson(FlowNetwork G, int s, int t) {
        V = G.V();
        validate(s);
        validate(t);
        value = 0.0;
        while(hasAugmentingPath(G, s, t)) {
            double bottle = Double.POSITIVE_INFINITY;
            // compute bottleneck capacity
            for (int v = t; v != s; v = edgeTo[v].other(v)) {
                bottle = Math.min(bottle, edgeTo[v].residualCapacityTo(v));
            }
            // augment flow
            for (int v = t; v != s; v = edgeTo[v].other(v)) {
                edgeTo[v].addResidualFlowTo(v, bottle);
            }
            value += bottle;
        }
    }

    // This implementation finds a shortest augmenting path (fewest number of edges),
    // which performs well both in theory and in practice
    private boolean hasAugmentingPath(FlowNetwork G, int s, int t) {
        edgeTo = new FlowEdge[G.V()];
        marked = new boolean[G.V()];

        // breadth-first search
        Queue<Integer> q = new Queue<>();
        q.enqueue(s);
        marked[s] = true;
        while(!q.isEmpty()) {
            int v = q.dequeue();
            for (FlowEdge e: G.adj(v)) {
                int w = e.other(v);
                // found path from s to w in the residual network
                // if residual capacity of v -> w > 0
                if (e.residualCapacityTo(w) > 0 && !marked[w]) {
                    edgeTo[w] = e;
                    marked[w] = true;
                    q.enqueue(w);
                }
            }
        }
        // is t reachable from s in residual network
        return marked[t];
    }

    public double value() {
        return value;
    }

    // Reture true if v is on the s side of min-cut
    public boolean inCut(int v) {
        validate(v);
        // If path s->v is in residual network, then v is in min-cut
        return marked[v];
    }

    private void validate(int v) {
        if (v < 0 || v >= V) throw new IllegalArgumentException();
    }
}
```
注：由于主要参考书是Algorithms, 4th Edition, 下面的代码中有引用随书algs4.jar中的类，其实官方也有提供更完善版本的Ford-Fulkerson算法实现[FordFulkerson.java][9]。

另外，上面代码用到了两个辅助类FlowNetWork和FlowEdge定义如下:
```java
public class FlowEdge {
    private final int v, w;
    private final double capacity;
    private double flow;

    public FlowEdge(int v, int w, double capacity) {
        this.v = v;
        this.w = w;
        this.capacity = capacity;
    }

    public int from() {
        return v;
    }

    public int to() {
        return w;
    }

    public int other(int vertex) {
        if (vertex == v) return w;
        else if (vertex == w) return v;
        else throw new RuntimeException("Illegal endpoint");
    }

    public double residualCapacityTo(int vertex) {
        if (vertex == v) return flow;
        else if (vertex == w) return capacity - flow;
        else throw new IllegalArgumentException();
    }

    public void addResidualFlowTo(int vertex, double delta) {
        if (vertex == v) flow -= delta;
        else if (vertex == w) flow += delta;
        else throw new IllegalArgumentException();
    }
}

public class FlowNetwork {
    private final int V;
    private int E;
    private Bag<FlowEdge>[] adj;

    public FlowNetwork(int V) {
        this.V = V;
        this.E = 0;
        adj = (Bag<FlowEdge>[]) new Bag[V];
        for (int v = 0; v < V; v++) {
            adj[v] = new Bag<>();
        }
    }

    public void addEdge(FlowEdge e) {
        int v = e.from();
        int w = e.to();
        adj[v].add(e); // forward edge
        adj[w].add(e); // backward edge
        E++;
    }

    public Iterable<FlowEdge> adj(int v) {
        return adj[v];
    }

    public int V() {
        return V;
    }

    public int E() {
        return E;
    }
}
```

另：推荐一个介绍Ford-Fulkerson方法的视频[Ford-Fulkerson in 5 minutes — Step by step example][10].

---
# 最大流最小割定理
一个流网络的最小割就是网络的所有割中容量最小的割，最大流的值一定不能超过最小割的容量。
实际上，最大流的值就等于某一最小割容量。

最大流最小割定理如下：
> 如果f是具有源点s和汇点t的流网络G=(V, E)中的一个流，则如下条件是等价的：
> 1.f是G的一个最大流
> 2.残留网络Gf不包含增广路径
> 3.对G的某个割(S, T)，有|f|=c(S, T).(|f|仅表示流f的值)

上面的FordFulkerson实现代码中，marked[]中所有值为true的顶点都属于当前找到的最小割中的S子集，因为marked[]中最终包含的是所有能从源点s抵达的却在中途因为没有残余流量而无法抵达汇点t的顶点，中间隔断的所有边都是饱和的，其容量和=最大流的值，因此在找到最大流的同时，我们也找到了一个最小割。

---
# 参考资料
* [Wikipedia - Ford–Fulkerson algorithm][11]
* [Wikipedia - Edmonds–Karp algorithm][12]
* **Data Structures and Algorithms in Java, 3rd Edition**, Book by Mark Allen Weiss
* **Introduction to Algorithms**, Book by Charles E. Leiserson, Clifford Stein, Ronald Rivest, and Thomas H. Cormen


  [1]: http://static.zybuluo.com/JaneL/9z5oz7hfq4lnlqqrsxlkexgy/maxflow.png
  [2]: https://en.wikipedia.org/wiki/L._R._Ford_Jr.
  [3]: https://en.wikipedia.org/wiki/D._R._Fulkerson
  [4]: http://static.zybuluo.com/JaneL/ek3vx5ov4nolkj7g2jszq7t4/maxflow-steps.png
  [5]: http://static.zybuluo.com/JaneL/blzxkwwbnwmxy4zwfnkpdb5j/wrong-maxflow.png
  [6]: http://static.zybuluo.com/JaneL/uw0qiz9it96eo25eupc1g4zq/correct-maxflow.png
  [7]: https://en.wikipedia.org/wiki/Jack_Edmonds
  [8]: https://en.wikipedia.org/wiki/Richard_M._Karp
  [9]: https://algs4.cs.princeton.edu/64maxflow/FordFulkerson.java.html
  [10]: https://www.youtube.com/watch?v=Tl90tNtKvxs&vl=en
  [11]: https://en.wikipedia.org/wiki/Ford%E2%80%93Fulkerson_algorithm
  [12]: https://en.wikipedia.org/wiki/Edmonds%E2%80%93Karp_algorithm