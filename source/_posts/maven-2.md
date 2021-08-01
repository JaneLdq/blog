---
title: Maven学习（二）- POM之初体验
date: 2018-05-08 19:52:51
categories: 工具
tags: Maven
---
安装好Maven就可以开始使用Maven构建并管理项目了。

## 使用Archetype搭建框架
第一步，自然是先搭建项目框架。运行如下命令：
```
mvn archetype:generate
```
在控制台会有一些交互，要求你输入一些值，如
<!--more-->

* **groupId** - 定义项目的groupId
* **artifactId** - 定义项目的artifactId
* **version** - 项目的版本
* **package** - 项目的默认包名

按照惯例，我们就从HelloWorld开始吧。截图如下所示：
![mvn-archetype-generate][1]

version和package还好理解，那么groupId和artifactId表示什么呢？我们稍后再详细介绍，先来看看这个命令到底做了什么。

当命令成功执行之后，会看到当前路径下新建了一个名为`hello-world`的文件夹，这个名字默认与创建时输入的**artifactId**一致，但之后你也可以随意修改文件夹名称。
整个项目的目录结构如下图所示：
![hello-world][2]

一条命令就搞定了项目的框架，这都归功于Maven的**Archetype**插件。
在经历过长期的实践之后，根据不同的应用场景会演化出各自相对固定的项目结构（它们也是某个领域的Best Practice的体现）。在开发同一类型的项目时，大家可能会要重复组建相同的结构，为了避免重复劳动，Maven提供了Archetype帮助我们快速搭建项目框架。

**一个Archetype可以看做一个项目模板，我们在使用Archetype时只需要提供最基本的元素（比如groupId, artifactId, version等），它就能生成项目的基本结构和POM文件。**

---

在我们运行`mvn archetype:generate`这条命令时，其实是采用了默认的**maven-archetype-quickstart**， 它定义的项目结构十分简单，基本内容如下：

* **src/main/java** - 放置项目主代码，里面的package的层级由创建时输入的package决定。例如输入com.example就会对应com/example的package层级。
* **src/test/java** - 放置项目测试代码。
* **pom.xml** - 包含JUnit依赖声明的POM文件。

除此之外，Maven还提供了很多Archetype，比如maven-archetype-webapp，maven-archetype-j2ee-simple等等，甚至如果你平时做的项目都遵循一定的结构和配置，你也可以自定义archetype。这里关于Archetype不做详细介绍，感兴趣的话请戳官网[**Introduction to Archetypes**][3]

举个栗子， 用maven-archetype-webapp创建项目：
![archetype-webapp][4]

生成的目录结构跟quickstart就不一样啦，如下图所示：
![hello-world-webapp][5]

当然，如果你很享受手工搭框架的过程，不用archetype命令，而选择自己手工敲pom.xml，手动建项目结构也是完全没问题哒，你开心就好。XD

Note: Maven自身有一套约定好的[Standard Directory Layout][6]，就是它默认某个路径存放某类内容。这样做的好处是使得所有的Maven项目画风统一，使用户即使接触全新的Maven项目时也不会觉得陌生。当然了，如果你想自定义目录结构，也是可以的，只需要在POM中配置一下就可以了。但强烈不推荐，个性化什么的在这里并不是那么合适噢~

---
## 初识POM
POM可以说是Maven最重要的一部分了，跟我一起念：POM - Project Object Model，翻译成中文就是**项目对象模型**啦。

顾名思义，在POM中定义了项目的基本信息，用于描述项目如何构建，声明项目依赖等等。这些信息都被放在一个名为pom.xml的文件中。

POM的存在使得项目对象模型与实际代码解耦，避免了Java代码与POM代码之间的影响。比如项目升级版本时不会影响Java代码，而POM更新之后，日常的Java开发也几乎不涉及POM的更改。

我们就以hello-world项目中的pom.xml为例感受一下。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  
  <!-- Part 1: Project basic information -->
  <groupId>com.example</groupId>
  <artifactId>hello-world</artifactId>
  <version>1.0-SNAPSHOT</version>
  <name>Hello World</name>
  <url>http://www.example.com</url>
  <packaging>jar</packaging>
  
  <!-- Part 2: Self-defined properties -->
  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <maven.compiler.source>1.7</maven.compiler.source>
    <maven.compiler.target>1.7</maven.compiler.target>
    <junit.version>4.11</junit.version>
  </properties>
  
  <!-- Part3: Dependencies -->
  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>${junit.version}</version>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <!-- Part 4: Build -->
  <build>
    <pluginManagement>
      <plugins>
        <plugin>
          <artifactId>maven-clean-plugin</artifactId>
          <version>3.0.0</version>
        </plugin>
        ...
      </plugins>
    </pluginManagement>
  </build>
</project>
```

**modelVersion** - 指定当前POM模型的版本，对于Maven3而言，只能是4.0.0。
### Part 1 - 项目基本信息
这里最重要的部分是groupId, artifactId和version这三个元素，它们一起定义了这个项目的基本坐标。在Maven中，任何一个构件（jar, war或者pom）都是基于这三个值进行区分的。
* **groupId** - 定义了项目所属的组，这个组通常与项目所在组织或者公司关联。比如
* **artifactId** - 定义当前项目在组中的唯一ID
* **version** - 定义项目当前的版本。SNAPSHOT是一个特殊的“快照”版本，说明项目还处于开发中，不是稳定版本。
  *另外三个值，name指定一个对用户更友好的项目名称，url指定项目站点，它们都不是必须的；packaging指定项目打包方式，如果不显示设定，默认值就是jar，另外还有两种常用打包方式：pom用于继承和聚合，war用于web应用。*

### Part 2 - properties
`<properties>`元素下可以自定义属性，就有点类似Java类中的成员属性，在pom文件的其他地方可以使用`${property.name}`来引用，比如上例中我自定义了一个property`<junit.version>4.11</junit.version>`，然后在依赖管理中配置添加junit依赖的版本时就可以使用`${junit.version}`来代替4.11这个值。<br>*这样做的一个好处是可以统一管理重复利用的值，比如当你依赖同一个groupId下的多个artifact时，可以在properties里添加一个group级别的版本号，然后在添加依赖时都引用同一个property，这样当需要修改版本时就不需要逐个更改artifact的版本，而只要在properties里把group版本改一下就好了。*

### Part 3 - dependencies
`<dependencies>`元素下可以包含多个dependency元素，用以声明项目的依赖。通过在dependency元素中指明groupId, artifactId, version就能找到对应的类库，可以说这三个值在Maven这个坐标系中精准定位了一个坐标。

### Part 4 - build
build元素包含了整个构建过程的配置。

---

上面这个pom.xml文件的结构只是一个非常简单的例子，实际上POM文件中的元素有很多组合方式，包括POM本身还有如继承、聚合等关系。 但是我想在继续深入了解POM之前，不如先玩一玩Maven命令，直观地感受一下Maven是如何进行项目构建的。

在项目根目录执行命令`mvn clean install`，你会看到很多诸如Downlord from central: http.....的输出，以及如下图所示看起来像是区分不同阶段的日志信息：

![mvn-clean-install][7]

那么这行命令又是干什么的呢？这些log是什么意思呢？

未完待续~

---
## 参考资料
* 官方入门指南[Maven Getting Started Guide][8]
* 《Maven实战》


  [1]: http://static.zybuluo.com/JaneL/qu9spmlwxct76vshhwsyyjp2/image.png
  [2]: http://static.zybuluo.com/JaneL/d7a17qq6brrjnk1vafozfkc6/image.png
  [3]: http://maven.apache.org/guides/introduction/introduction-to-archetypes.html
  [4]: http://static.zybuluo.com/JaneL/f4k7jixegd9o2clk44g0qby5/image.png
  [5]: http://static.zybuluo.com/JaneL/t32umtdxz1pvou7gcauvcaat/image.png
  [6]: http://maven.apache.org/guides/introduction/introduction-to-the-standard-directory-layout.html
  [7]: http://static.zybuluo.com/JaneL/vzn959yejnbew820tmb5a3q8/image.png
  [8]: http://maven.apache.org/guides/getting-started/index.html