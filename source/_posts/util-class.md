---
title: 实现Util类的最佳实践
date: 2019-09-18 08:22:12
categories: 技术笔记
---

在日常开发中，我们难免会要实现自己一些 Util 类。Util 类通常是一组静态方法的集合，它本身是无状态的 (stateless)，所有方法执行依赖的参数都由显式参数给出。基于这样的特性，大家总结出了一些针对 Util 类的设计和实现的最佳实践，其中最主要就是如下两点：

* 将类声明为`final`
* 构造函数私有化

其实很简单，不过自己写代码的时候总是容易忘记，所以特地记下来～
<!--more-->

---
# 使用 final 关键字修饰 Util 类
作为一组通用方法的集合，很显然我们不需要也不希望这个类能被继承和修改。为了从根本上杜绝这一点，我们可以使用 `final` 关键字来修饰 Util 类。

一个 `final` 类中的所有方法也都是 `final` 方法，整个类都不能继承了，自然也就不存在方法被 override 的情况。需要注意的一点是，一个 `final` 类中定义的字段并不是 `final` 的，如果需要定义 `final` 字段，需要显式声明。

---
# 私有构造函数
Util 类既然是一组静态方法的集合，自然也不应该也不需要被实例化。在 Java 中，如果不显式自定义构造函数，那么编译器会隐式为所有类创建一个默认的 `public` 的空构造函数，为了避免 Util 类被实例化，我们最好给 Util 类添加一个 `private` 构造函数。

---

最后，我们来一个简单例子吧：
```java
public final class StringUtils {

    // final 字段需要显式声明
    public static final String EMPTY = "";

    // 私有构造函数
    private StringUtils() {}
    
    // 静态方法
    public static boolean isEmpty(Object str) {
        return (str == null || "".equals(str));
    }

}
```

像 `java.lang.System`， `java.lang.Math` 这些类都遵循以上两点哦～

---
# 使用 Lombok @UtilityClass 注解
除了自己实现 Util 类时注意加上 final 关键字和私有构造函数，第三方库 [Lombok][1] 中也提供了一个便捷的注解 [**@UtilityClass**][2]。
这个注解帮我们做了以上两件事，除此之外它还会将所有类中的方法加上 `static` 修饰符。

使用 `@UtilityClass`，上面的例子可以改写为如下这样：
```java
import lombok.experimental.UtilityClass;

@UtilityClass
public class StringUtils { // 类声明中的 final 修饰符可以省去，也不需要再自己写一个 private 构造函数

    public static final String EMPTY = "";

    // 方法声明中可以省去 static 关键字
    public boolean isEmpty(Object str) {
        return (str == null || "".equals(str));
    }

}
```

---

**参考资料**

* [What is the best way to write utility classes in Java?][3]
* [SonarSource Rules - Utility classes should not have public constructors][4]

  [1]: https://projectlombok.org/
  [2]: https://projectlombok.org/features/experimental/UtilityClass
  [3]: https://www.quora.com/What-is-the-best-way-to-write-utility-classes-in-Java
  [4]: https://rules.sonarsource.com/java/tag/design/RSPEC-1118