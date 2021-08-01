---
title: Maven学习（六）- 构建多模块项目
date: 2018-05-26 09:58:47
categories: 工具
tags: Maven
---

为了找个相对合适的项目需求，翻出了上大学时的课程大作业，真正的程序猿要敢于直面自己敲出的惨不忍睹的代码 o.o

## 项目需求
先简单叙述一下需求：
整个系统既包括面向客户的线上订购部分，也包括内部员工管理部分。
<!--more-->

* 顾客：浏览甜品；购物车增删改查，个人信息管理（基本信息，积分，消费明细等）
* 员工：店面/甜品/雇员信息的增删改查，库存采购计划管理等
更详细的功能需求请戳项目仓库[**Github-Tian**][1]

---

在分析需求之后，我决定将面向顾客和面向员工做成两个独立的Web模块，共用底层业务逻辑和数据库的增删改查。

在这里，我们就要运用到Maven中POM的**继承**和**聚合**特性啦。在我看来，这跟面向对象里的继承和聚合几乎没有差别~
聚合特性就是指Maven能把项目的各个模块聚合在一起构建，继承特性就是能把各模块相同的依赖和插件等配置抽取出来，放在一个叫做parent-pom的地方实现复用，既能简化POM又便于维护~

下面我们就来看看具体的使用吧。

---

## 建立聚合与继承关系
在项目根目录下创建pom.xml，通常用于继承和聚合会是同一个POM文件。部分内容如下所示
```xml
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>edu.nju</groupId>
    <artifactId>tian-parent</artifactId>
    <version>1.0-SNAPSHOT</version>

    <packaging>pom</packaging>
    <name>Tian - Parent Project</name>

    <modules>
        <module>tian-web</module>
        <module>tian-bms-web</module>
        <module>tian-service</module>
    </modules>
    ...
</project>
```

上面这段配置中，`groupId`, `artifactId`, `version`都跟之前一样，需要注意的是作为父POM和负责聚合的POM的`packaging`打包方式必须为`pom`。
下面新引入了一个元素`modules`来实现模块的聚合，在这里面定义了三个`module` - tian-web, tian-bms-web, tian-service，分别表示指向面向顾客的web项目，面向员工的web项目和数据库service层的所在路径，这个路径是相对当前pom而言的。
一般来说，为了方便，模块所处的目录名通常与其artfactId相同。当然啦，举个栗子，你要是想把tian-web项目放在web-tian/目录下也可以，只要将配置改成`<module>web-tian</module>`就可以了。不过真心不推荐，为什么要给自己增加负担呢~

为了方便用户构建项目，通常将聚合模块放在项目根目录，其他模块则作为子目录存在。

那么按照当前的子模块划分，我们的项目目录长成这样：
![project structure][2]

三个子模块目录分别代表三个子Maven项目，每个子目录下都有各自Maven项目的pom.xml。

---
在tian-parent项目的POM中我们通过`modules`元素实现了模块的聚合关系，那么模块间的继承关系是如何实现的呢？

我们以tian-web项目的POM为例来说明。

```xml
<project>
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>edu.nju</groupId>
        <artifactId>tian-parent</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>

    <artifactId>tian-web</artifactId>
    <packaging>war</packaging>
    <name>Tian - Dessert House</name>
    ...
</project>
```

在子模块的POM中通过引入`parent`元素来建立继承关系，通过`groupId`,`artifactId`,`version`指定父POM的坐标，如果父POM对于当前POM的相对路径不是Maven默认的上一级目录的话，还需要配置`relativePath`元素设置正确的相对路径。

在tian-web的POM中没有为其设定`groupId`和`version`，这两个值都能从父POM中继承得来。如果子模块想单独设定，再在自己的POM中显式声明就可以了。

类似的，在tian-bms-web和tian-service子模块下创建各自pom.xml。

## 依赖管理
上面一小节介绍了如何搭建多模块项目的框架，这一部分就来介绍一下如果通过继承实现依赖和插件的管理。

前一篇笔记在介绍依赖管理时有提到`dependencyManagement`元素：在`dependencyManagement`元素下的依赖声明不会引入实际的依赖，但在其中声明的配置能约束`dependencies`元素下依赖的使用。
利用这一特性，我们可以在父POM的`dependencyManagement`元素下声明所有子模块都要用到的依赖，这样既能让子模块继承父模块的依赖配置，又能保证子模块使用依赖的灵活性。在`dependencyManagement`里声明的内容包括groupId，artifactId, version以及其他配置。对于这些通用信息，就不需要在每个子模块重复配置了。如果某个子模块需要不同的配置，可以通过在子POM中显式声明覆盖父POM的配置。

放到项目中，举个栗子，tian-web和tian-bms-web都是web项目，而且都选择Spring框架来实现，那么这两个模块中都会要声明与Spring框架的依赖。那么为了统一管理Spring依赖的信息，可以在父POM中做如下配置：

```xml
<properties>
    <spring.version>4.3.16.RELEASE</spring.version>
</properties>
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>${project.groupId}</groupId>
            <artifactId>tian-service</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-web</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-webmvc</artifactId>
            <version>${spring.version}</version>
        </dependency>
        ...
    </dependencies>
</dependencyManagement>
```
如上所示，我们在POM中还可以自定义`properties`进一步提高POM的可维护性~

下面是tian-web项目的pom.xml中的部分内容：

```xml
<dependencies>
    <dependency>
        <groupId>${project.groupId}</groupId>
        <artifactId>tian-service</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
    </dependency>
    ...
</dependencies>
```
如上所示，在子模块中，直接在`dependencies`中声明要用的依赖，像version这种信息就可以直接继承父POM的依赖配置了。（另一个比较典型的例子就是Junit依赖中scope的配置。）

看起来使用依赖管理后子POM中的内容并没有减少太多，但是这样做能省去版本声明这一操作能避免多个子模块同一依赖版本不一致的情况，从而降低依赖冲突的发生。

---
## 插件管理
子模块间也可能存在重复配置插件的情况，同理，我们可以把可复用的插件配置放到父POM中统一管理。

最常见的例子就是compiler插件的配置：

```xml
<build>
    <pluginManagement>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.1</version>
                <configuration>
                    <source>1.7</source>
                    <target>1.7</target>
                    <encoding>utf-8</encoding>
                </configuration>
            </plugin>
        </plugins>
    </pluginManagement>
</build>
```
当项目中多个模块有同样的插件配置时，就应该移到父POM的`pluginManagement`元素中。即使各个子模块对同一插件的具体配置不同，也应该在父POM中统一管理插件版本。其他配置可由各子模块自行配置覆盖父模块。

---
## 多模块项目的Reactor
 
 Reactor（反应堆）对于单模块项目就是该模块本身，但对于多模块项目而言，reactor是由所有模块组成的一个构建结构，包含了各模块和他们之间的继承和依赖关系，并能够自动计算出合理的模块构建顺序。

还是来看示例项目，我们在tian-parent的POM中声明了三个子模块：

```xml
<modules>
    <module>tian-web</module>
    <module>tian-bms-web</module>
    <module>tian-service</module>
</modules>
```

项目构建输出如下：
![reactor-order][3]

可以看到Reactor的构建顺序是parent->service->web->web-bms，与声明顺序并不完全一致。这是因为Maven通过分析模块间的依赖关系，得知两个web模块都依赖于service模块，因此Maven会先构建service模块。

反应堆最终可以表示为一个有向无环图。如果各模块间有循环依赖，构建会失败哒~

---

Maven构建多模块项目的基本使用就介绍到这里啦，示例项目Github仓库请戳[**Tian**][4]（PS：为了保留黑历史，master分支还是曾经简单粗暴的架构，更新后的架构放在refactor分支上了~）。

未完待续~

---
## 参考资料

* *Maven: The Definitive Guide*
* 《Maven实战》


  [1]: https://github.com/JaneLdq/Tian-Dessert-House
  [2]: http://static.zybuluo.com/JaneL/krafrwtd8tu0g5uuwo0tc99t/image.png
  [3]: http://static.zybuluo.com/JaneL/vpqah50i79l9l01ic9z0ygh7/image.png
  [4]: https://github.com/JaneLdq/Tian-Dessert-House