---
title: 【译记】关于equals那些事
date: 2018-08-13 14:20:43
categories: 技术笔记
tags: Java
---
在跑Code Scan的时候报了一个关于equals()的问题：

> 父类重写了equals，子类继承了父类并添加了新的属性，却没有重写equals。

印象中equals的重写是个比较难处理好的问题。首先我们先回顾一下Java中默认的Object类的默认equals实现。

---
# Object.equals
Java中最顶层的Object类中默认的equals实现很简单，就是比较两个对象的引用，如果指向同一个地址则相同，反之则不同。
```java
public class Object {
    public boolean equals(Object obj) {
        return (this == obj);
    }
}
```
这种实现适用于绝大部分情况。但是如果我们想通过对象内部的某些属性（或者说是对象的状态）来判断两个对象是否相同，就会需要重写equals。
<!--more-->

---
# 那么如何在Java中如何重写equals呢？
恰好找到了一篇文章[**How to Write an Equality Method in Java**][1]，虽然是2009年的文章，但是并不影响它写得很好，下文是主要部分的翻译笔记。

这篇文章里主要介绍了4个在Java中重写equals时容易踩的坑，我们一个个来看。

## Pitfall #1: Defining equals with wrong signature.
考虑给如下的Point类增加一个equals：
```java
public class Point {
    private final int x;
    private final int y;
    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
    public int getX() {
        return x;
    }
    public int getY() {
        return y;
    }
}
```
一种显而易见错误定义方式如下：
```java
// 完全错误！
public boolean equals(Point other) {
        return (this.getX() == other.getX() && this.getY() == other.getY());
    }
```

这种定义错在哪里呢？第一眼看过去好像没什么问题诶：
```java
Point p1 = new Point(1, 2);
Point p2 = new Point(1, 2);
Point q = new Point(2, 3);
System.out.println(p1.equals(p2)); // true
System.out.println(p1.equals(q)); // false
```
然而，一旦你把points放到一个集合中，问题就出现了：
```java
HashSet<Point> coll = new HashSet<>();
coll.add(p1);
System.out.println(coll.contains(p2)); // false
```
为什么coll中不包含p2呢？虽然coll中添加的是p1，但是p1和p2应该是等同的呀。原因就出在这里：在进行比较的两个point中有一个point的精确类型被遮盖了的。
HashSet的contains方法是这样的：
```java
public boolean contains(Object o) {
    return map.containsKey(o);
}
```
所以上面那段比较等同于如下代码：
```java
Object p2a = p2;
System.out.println(p1.equals(p2a)); // false
```
到底是哪里错了呢？实际上，之前定义的`equals(Point p)`并没有重写标准的`equals(Object other)`，因为参数类型不同，这是重载。重载在Java中是由参数的静态类型决定的，而不是运行时类型。所以，只要静态类型是Point，那么Point类中的equals才会被调用，如果静态类型是Object，那么就会调用Object类中的equals。而在上例中，`equals(Object other)`并没有被重写，它还是按照对象的引用来进行比较的。所以`p1.equals(p2)`返回的是`false`，即使p1和p2a都有相同的x和y。这也是为什么HashSet的contains会返回`false`。

一种稍微好一点的实现如下：
```java
@Override
public boolean equals(Object other) {
    boolean result = false;
    if (other instanceof Point) {
        Point that = (Point) other;
        result = (this.getX() == that.getX() && this.getY() == that.getY());
    }
    return result;
}
```
这样equals以Object类型作为参数，以boolean作为返回值，这样就是正确的重写方式。在方法内部，先使用`instanceof`判断传入的参数other是不是Point类型，如果是，则进一步比较坐标，否则返回false。

---
## Pitfall #2：Changing equals without also changing hashCode

如果你重新使用上面的定义比较p1和p2a，你会得到true。然而，如果你重新使用HashSet.contains进行测试，那么你有可能得到false。
为什么会这样呢？就是因为**在Point中重写了equals却没有重写hashCode**。

在上例中用到的集合是HashSet，也就意味着每个元素都被放到一个由它们的hash值决定的篮子里。`contains`方法首先会计算出需要查找的这个对象的hash值，然后到对应的篮子里逐一比较。而我们上面定义的Point类，重写了equals，却没有同时重写hashCode，那么它的hashCode还是调用的在Object类中定义的那个版本，也就是使用对象分配的地址进行转换得到的某个值。所以p1和p2即使有相同的坐标也极大概率不会被分到同一个篮子里了（当然这个几率还是有的，只是非常小而已）。

所以呢，问题就出在上个版本的Point的实现违反了hashCode的定义：
> If two objects are equal according to the equals(Object) method, then calling the hashCode method on each of the two objects must produce the same integer result.
 
实际上，在Java中hashCode和equals总是如影随形的已经是众所周知的事情啦。进一步说，hashCode中只应该依赖于equals中依赖的那些属性。以Point类为例，如下是一个合适的hashCode的定义：
```java
public class Point {
    private final int x;
    private final int y;
    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
    public int getX() {
        return x;
    }
    public int getY() {
        return y;
    }
    @Override
    public boolean equals(Object other) {
        boolean result = false;
        if (other instanceof Point) {
            Point that = (Point) other;
            result = (this.getX() == that.getX() && this.getY() == that.getY());
        }
        return result;
    }
    @Override public int hashCode() {
        return (41 * (41 + getX()) + getY());
    }
}
```
上面这个hashCode的实现只是诸多实现方式中的一种。
通过同时重写hashCode能解决像如上的Point类的equals问题，但是，仍然还有其他需要小心的陷阱哦~

---
## Pitfall #3: Defining equals in terms of mutable fields.
考虑对Point类做如下改动：
```java
public class Point {
    private int x;
    private int y;
    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
    public void setX(int x) {
        this.x = x;
    }
    public void setY(int y) {
        this.y = y;
    }
}
```
这里唯一的变动就是x和y不再是final的，并且新增了两个set方法。而这里的equals和hashCode都依赖于这些可变的属性，当属性的值发生变化时它们运行的结果也会发生变化。这时若把points放到集合中，会出现奇怪的结果：
```java
Point p = new Point(1, 2);
HashSet<Point> coll = new HashSet<>();
coll.add(p);
System.out.println(coll.contains(p)); // true
```
此时，如果你改变p的某个值，这个point还在集合中吗？
```java
p.setX(p.getX() + 1);
System.out.println(coll.contains(p)); // false(probably)
```
好奇怪，p去哪里了？如果你使用迭代器遍历来检查coll中是否包含p还会遇到更奇怪的事情哦：
```java
Iterator<Point> itr = coll.iterator();
boolean containedP = false;
while(itr.hasNext()) {
    Point nextP = itr.next();
    if (nextP.equals(p)) {
        containedP = true;
        break;
    }
}
System.out.println(containedP); // true
```
集合说它不包含p，但是p又明明出现在了它的元素中。怎么回事？其实很简单啦，在p的x值发生更改之后，再计算得到的hash值会指向一个错误的篮子，这个篮子不再是原来放的那个篮子，自然就找不到了。换句话说，就是p还在coll中，但是我们已经看不到它了(dropped out of sight)。

这个例子想说明的就是当equals和hashCode依赖于可变的属性时，可能会对使用者造成影响。如果使用时把对象放到集合中，就要非常小心不要去改变这些方法依赖的状态，这可不是什么好事。如果你在进行比较时需要考虑到一个对象当前的状态，最好是单独定义一个方法，而不要使用equals。
以上面这个版本的Point类为例，一种更好的方式是不要重写equals和hashCode，而是新建一个方法比如叫equalContents来实现坐标的比较。这样Point类就会继承默认的Object的equals和hashCode方法。这样即使修改了p的内容也可以在集合中找到p了。

---
## Pitfall #4: Failing to define equals as an equivalence relation
Object中的equals方法必须在非空对象间为等价关系：

* 自反性：for any non-null value x，x.equals(x) returns true
* 对称性：for any non-null values x and y, x.equals(y) returns true iff y.equals(x) returns true.
* 传递性：for any non-null values x, y and z, if x.equals(y) returns true and y.equals(z) returns true, then x.equals(z) returns true.
* 一致性：for any non-null values x and y, multiple invocations of x.equals(y) should consistently return true or consistently return false, provided no information used in equals comparisons on the objects is modified.
* For any non-null value x, x.equals(null) returns false.

上面的Point类的例子中的equals实现都是符合如上规则的，然而，一旦我们引入了继承关系，情况就变得复杂了起来。（这正是这篇笔记最最开始抛出问题的地方。）

现在引入一个新的Point的子类ColoredPoint，定义如下：
```java
public enum Color {
    RED, ORANGE, YELLOW, GREEN, BLUE, INDIGO, VIOLET;
}

public class ColoredPoint extends Point {

    private Color color;

    public ColoredPoint(int x, int y, Color color) {
        super(x, y);
        this.color = color;
    }

    @Override
    public boolean equals(Object other) {
        boolean result = false;
        if (other instanceof ColoredPoint) {
            ColoredPoint that = (ColoredPoint) other;
            result = this.color.equals(that.color) && super.equals(that);
        }
        return result;
    }
}
```
上面是很多程序员可能写出来的代码。注意，在这里ColoredPoint类不需要重写hashCode，因为ColoredPoint类中的equals比Point中定义的更严格，因此hashCode仍然是有效的。如果两个彩色的point相同，它们一定有相同的坐标，那么它们的hashCode也一定是相同的。

这时单独比较ColoredPoint，它的equals是没问题的。但是如果把Point和ColoredPoint混合起来，equals的等价的约束就被打破了。考虑如下代码：
```java
Point p = new Point(1, 2);
ColoredPoint cp = new ColoredPoint(1, 2, Color.RED);
System.out.println(p.equals(cp)); // true
System.out.println(cp.equals(p)); // false
```
`p.equals(cp)`调用的是Point的equals，它只考虑两个point的坐标，因此返回true。而`cp.equals(p)`调用的是ColoredPoint的equals，它会返回false，因为p不是ColoredPoint类的实例。那么，这个关系就不满足对称性了。

要怎么做才能使得equals满足对称性呢？有两种方式：

### 把相等关系定义得更宽泛
假设有a和b两个对象，只要a.equals(b)或b.equals(a)中只要有一个返回true，那么就认为a和b是相等的。代码如下：
```java
public class ColoredPoint extends Point {
    private Color color;
    
    public ColoredPoint(int x, int y, Color color) {
        super(x, y);
        this.color = color;
    }
    
    @Override
    public boolean equals(Object other) {
        boolean result = false;
        if (other instanceof ColoredPoint) {
            ColoredPoint that = (ColoredPoint) other;
            result = this.color.equals(that.color) && super.equals(that);
        } else if (other instanceof Point) {
            Point that = (Point) other;
            result = that.equals(this);
        }
        return result;
    }
}
```
这个ColoredPoint中新定义的equals多检测了一种情况：如果other是Point的实例却不是ColoredPoint的实例，那么就用Point的equals方法进行比较。这样，`cp.equals(p)`和`p.equals(cp)`都会返回true。然而，**这样做仍然会破坏equals的约束，新的实现会打破equals的传递性！**看如下代码：
```java
Point p = new Point(1, 2);
ColoredPoint redP = new ColoredPoint(1, 2, Color.RED);
ColoredPoint blueP = new ColoredPoint(1, 2, Color.BLUE);
System.out.println(redP.equals(p)); // true
System.out.println(p.equals(blueP)); // true
System.out.println(redP.equals(blueP)); // false
```

可见，equals的传递性不被满足了。
似乎把equals的关系泛化把我们带到沟里了。那让我们在试试另一条路：

### 把相等关系定义得更严格
使得关系更严格第一种方式是总是把两个不同类的对象看作是不同的，这可以通过修改Point和ColoredPoint类的equals方法实现：在原来的equals方法中加上对运行时类型的比较：
```java
// A technically valid, but unsatisfying, equals method
public class Point {
    private final int x;
    private final int y;
    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
    @Override
    public boolean equals(Object other) {
        boolean result = false;
        if (other instanceof Point) {
            Point that = (Point) other;
            result = (this.getX() == that.getX() && this.getY() == that.getY()
                    && this.getClass().equals(that.getClass()));
        }
        return result;
    }
    @Override
    public int hashCode() {
        return (41 * (41 + getX()) + getY());
    }
}
```
然后把ColoredPoint改回到前一个版本：
```java
public class ColoredPoint extends Point {

    private Color color;

    public ColoredPoint(int x, int y, Color color) {
        super(x, y);
        this.color = color;
    }

    @Override
    public boolean equals(Object other) {
        boolean result = false;
        if (other instanceof ColoredPoint) {
            ColoredPoint that = (ColoredPoint) other;
            result = this.color.equals(that.color) && super.equals(that);
        }
        return result;
    }
}
```
这样，如果一个Point类的实例p等于另一个对象实例other，那么这个对象实例other一定在运行时的类型一定也是Point，并且与p有相同的坐标。这个新的equals方法既满足了对称性又满足了传递性。因为一个ColoredPoint的实例永远不可能等同于一个Point类的实例，因为它们的运行时类型不同。
这个实现在技术上是合法的，但是有人会说了，这个新的equals好像严格过头了。

考虑如下这段代码：
```java
Point pAnon = new Point(1, 1) {
    @Override
    public int getY() {
        return 2;
    }
}
```
这里的pAnon和p是相等的吗？答案是No。因为p和pAnon的java.lang.Class对象是不同的。p的getClass是Point，而pAnon的是一个Point的匿名子类。然而，从内容上来看，pAnon就是另一个坐标也为(1, 2)的点，如果把p和pAnon看成不同的似乎不那么合理呀。

---
### The canEqual Method
那怎么办呢？太宽泛也不行，太严格也不行。到底还能不能愉快地玩耍了呢？解决方法还是有的，这个方法的思想是：一旦一个类重写了它的equals和hashCode方法，它也应该显示地声明，如果这个类的父类实现了一个不同的equals方法，那么任何一个该类的实例都不等于这个父类的某个实例。

> The idea is that as soon as a class redefines equals (and hashCode), it should also explicitly state that objects of this class are never equal to objects of some superclass that implement a different equality method. 

要达到这个目标，需要在每一个重写了equals方法的类中新增一个canEqual方法，canEqual的声明如下：
```java
public boolean canEqual(Object other)
```

如果other是当前定义canEqual类的一个实例则返回true，否则返回false。这个canEqual在equals中被调用，用于确保对象从两个方向进行比较。下面是Point类实现的最新也是最后的版本：
```java
public class Point {
    private final int x;
    private final int y;
    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
    public int getX() {
        return x;
    }
    public int getY() {
        return y;
    }
    @Override
    public boolean equals(Object other) {
        boolean result = false;
        if (other instanceof Point) {
            Point that = (Point) other;
            result = (that.canEqual(this) && this.getX() == that.getX() && this.getY() == that.getY());
        }
        return result;
    }
    @Override
    public int hashCode() {
        return (41 * (41 + getX()) + getY());
    }
    public boolean canEqual(Object other) {
        return other instanceof Point;
    }
}
```
上面这个Point类的equals增加了一个判断条件：other是不是具备跟this进行比较的资格（the other object can equal this one）。在canEqual中进行的判断表示任何一个Point类的实例都是**可以**与this进行比较的。

下面是对应的ColoredPoint的实现：
```java
public class ColoredPoint extends Point {
    private Color color;

    public ColoredPoint(int x, int y, Color color) {
        super(x, y);
        this.color = color;
    }

    @Override
    public boolean equals(Object other) {
        boolean result = false;
        if (other instanceof ColoredPoint) {
            ColoredPoint that = (ColoredPoint)other;
            result = that.canEqual(this) && this.color.equals(that.color) && super.equals(that);
        }
        return result;
    }

    @Override
    public int hashCode() {
        return (41 * super.hashCode() + color.hashCode());
    }

    @Override
    public boolean canEqual(Object other) {
        return other instanceof ColoredPoint;
    }
}
```

上面的Point和ColoredPoint的实现是满足equals的约束的，既满足对称性也满足传递性。把一个Point跟一个ColoredPoint作比较永远都是false。这是显而易见的，`p.equals(cp)`返回false因为`cp.canEqual(p)`会返回false。反过来，`cp.equals(p)`也会返回false，因为p并不是ColoredPoint的实例。

另一方面，如果Point的其他子类没有重写equals方法，那么这些Point子类间的实例可以是相同的。
比如前面提到的p和pAnon的比较结果就是相同的了。举几个例子：
```java
Point p = new Point(1, 2);
ColoredPoint cp = new ColoredPoint(1, 2, Color.RED);
Point pAnon = new Point(1, 1) {
    @Override
    public int getY() {
        return 2;
    }
};
HashSet<Point> coll = new HashSet<>();
coll.add(p);
System.out.println(coll.contains(p)); // true
System.out.println(coll.contains(cp)); // false
System.out.println(coll.contains(pAnon)); // true
```

从上面这些例子可以看到，如果父类重写了equals并定义了canEqual，那么在实现它的子类时，是否允许子类的实例等同于父类的实例是由编写子类的程序员来决定的。比如上例中ColoredPoint子类重写了canEqual，所以一个ColoredPoint的实例就永远不可能等于一个Point的实例，而另外的匿名子类没有重写canEqual，所以它的一个实例可以等于一个Point实例。

canEqual方法有一个潜在的问题，那就是它违反了[**Liskov Substitution Principle**][2](LSP)。举个例子，equals的实现是通过比较运行时的类型，这就导致了无法允许一个子类的实例等于一个父类的实例，这就违反了LSP。因为LSP声明你应当可以使用一个子类实例来替代父类。然而，在我们之前的例子中，`coll.contains(cp)`返回的是false，即使cp的x和y值都与集合中保存的p相同。这看起来是违反了LSP的，因为这里期望的类型是Point，但是却不能使用它的子类ColoredPoint。我们不否认这个行为是不对的，但是呢LSP也没有硬性规定子类和父类的行为必须一模一样，只要在某种程度上满足了父类的约定就行。

其实，在重写的equals方法中使用运行时类型进行比较的主要问题并不是它违反了LSP这一点，而是这样的实现会使得子类的实例永远都不同于父类的实例。

---

最后，我们在回顾一下在重写equals时需要避开的坑：
1. equals的声明错误，把override变成了overload
2. 重写了equals却没有重写hashCode
3. 在equals中用到了mutable的属性，很危险
4. 实现equals时打破了等价关系

一丢丢题外话：之所以会看到这篇文章，是在解决issue时用到了一个Java类库[**Lombok**][3]，它里面有一个
注解[`@EqualsAndHashCode`][4]可以自动生成equals和hashCode方法，在生成的代码里出现了canEqual方法，感觉很神奇，就搜了一下相关内容，里面有link到这篇文章~


---
# Class-level access modifier
除了equals之外，还想提到一个小细节，就是访问修饰符的作用级别。
举个例子，在ColoredPoint的equals方法中可以直接访问`that.color`：
```java
@Override
public boolean equals(Object other) {
    boolean result = false;
    if (other instanceof ColoredPoint) {
        ColoredPoint that = (ColoredPoint)other;
        result = that.canEqual(this) && this.color.equals(that.color) && super.equals(that);
    }
    return result;
}
```
`color`不是ColoredPoint的私有属性吗？为什么that在这里可以直接访问呢？最开始看到这里是有点疑惑的。
原因在于修饰符的作用是class-level而不是object-level的，为什么要封装在类而不是更进一步到对象级别呢？我在Stack Overflow上找到了一段个人认为很有说服力的解释：

> It seems that object level access modifier would enforce the Encapsulation principle even further.
> But actually it's the other way around. Let's take an example. Suppose you want to deep copy an object in a constructor, if you cannot access the private members of that object. Then the only possible way is to add some public accessors to all of the private members. This will make your objects naked to all other parts of the system.
> So encapsulation doesn't mean being closed to all of the rest of the world. It means being selective about whom you want to be open to.

出处：[Access private field of another object in same class][6]


---

# 参考资料
* [How to Write an Equality Method in Java][5]
* Core Java Volume I - Fundamentals


  [1]: https://www.artima.com/lejava/articles/equality.html
  [2]: https://en.wikipedia.org/wiki/Liskov_substitution_principle
  [3]: https://projectlombok.org/
  [4]: https://projectlombok.org/features/EqualsAndHashCode
  [5]: https://www.artima.com/lejava/articles/equality.html
  [6]: https://stackoverflow.com/questions/17027139/access-private-field-of-another-object-in-same-class