---
title: Maven学习（五）- 依赖管理
date: 2018-05-21 23:59:04
categories: 工具
tags: Maven
---
前面几篇笔记陆续介绍了Maven中的POM、生命周期、插件、坐标和仓库等概念，在核心概念里还有一个非常重要的部分——依赖管理，也就是这篇文章的主要内容啦。

一个相对复杂的项目通常会包含对第三方类库的依赖，甚至内部各模块之间也会有依赖。Maven的依赖管理就是用来协助开发者进行这部分工作的。

<!--more-->
正如插件管理，依赖的管理也基于Maven的坐标系统。
在我们的hello-world项目的POM中，已经包含了一个很简单的依赖，代码如下：

```xml
<dependencies>
  <dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.11</version>
    <scope>test</scope>
  </dependency>
</dependencies>
```
在上面这段配置中，我们又见到了熟悉的groupId, artifactId, version，这里不再赘述。另外多了一个`scope`元素，它就是专属于`<dependency>`的一个子元素。下面列出了`<dependency>`下可以包含的所有元素，下文将分别详细介绍：

* groupId, artifactId, version
* type - 依赖的类型，对应于packaging。一般不用指明，默认为jar
* scope - 依赖的范围
* optional - 标记依赖是否可选
* exclusions - 排除传递性依赖

---
## 依赖范围（Dependency Scope）
首先我们要知道Maven在编译和编译+执行测试时使用的不同的classpath，而Maven项目在实际运行时又是另一套classpath。
scope就是用来控制依赖和classpath的关系的，比如依赖应该处于哪些classpath中，哪些依赖需要被包括在最终打包的应用中等等。

scope一共有以下6种：

* **compile** - compile是默认的scope，所有使用此范围的依赖会被加到所有classpath中，并被打包到输出中。
* **provided** - 使用provided范围的依赖对于compile/test classpath有效，但不在runtime classpath中。provided的意思就是在运行时，这个依赖已经由运行时环境（JDK或者其他container）提供了。举个栗子，Servlet API就不需要被打包进war中，因为Servlet API Jar应该由容器（server container）提供。
* **runtime** - 执行时需要，但编译时不需要的依赖。使用此范围的依赖会被加到runtime/test classpath，但不在compile classpath。举个栗子，JDBC Driver的具体实现只在运行时需要，而编译时只需要JDBC接口就可以了。
* **test** - 使用此依赖表示依赖只在测试时需要（比如JUnit），因此只会加到test classpath。
* **system** - 与classpath的关系跟provided相同，只是使用system范围的依赖要显式指出依赖所处的目录。这类依赖通常不是通过Maven仓库解析而与本机系统相关，会影响项目的可移植性。使用需谨慎哦。举个栗子：

```xml
<dependency>
  <groupId>javax.sql</groupId>
  <artifactId>jdbc-stdext</artifactId>
  <version>2.0</version>
  <scope>system</scope>
  <systemPath>${java.home}/lib/rt.jar</systemPath>
</dependency>
```
* **import** - 与`dependencyManagement`的使用相关。

---

下表总结了依赖范围与三个classpath之间的关系：

scope|compile classpath|test classpath|runtime classpath| example
---|---|---|---|---
compile | √ | √ | √ | spring-core
provided | √ | √ | N/A | servlet api
runtime| N/A | √ | √ | JDBC Driver Implementation
test | N/A | √ | N/A | JUnit
system | √ | √ | N/A | 本地类库（不在Maven仓库内的）

另外，考虑到依赖的传递性，这些scope在组合之后还会有不同的效果。想了解更多相关知识请戳 [**Dependency Scope**][1]

---
## 可选依赖
假设现在我们在做一个支持多种数据库的工具包，项目本身在构建时会依赖多种数据库类库，比如MySQL，MongoDB等。而这个工具包在真正使用时只会依赖其中的一种数据库。我们希望在使用这个工具包时，可以避免加载不必要的传递性依赖。这时就可以使用可选依赖。
举个栗子：

```xml
<project>
  <groupId>com.example</groupId>
  <artifactId>db-tools</artifactId>
  <version>1.0.0</version>
  <dependencies>
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>8.0.11</version>
      <optional>true</optional>
    </dependency>
    <dependency>
      <groupId>org.mongodb</groupId>
      <artifactId>mongo-java-driver</artifactId>
      <version>3.7.0</version>
      <optional>true</optional>
    </dependency>
  </dependencies>
  ...
</project>
```
上面这两个依赖配置都添加了一条`<optional>true</optional>`的元素，声明这两个是可选依赖。
对于这些被声明为可选的依赖，当添加对`db-tools`这个构件的依赖时，必须显式声明同时需要使用的可选依赖。比如要写一个基于MySQL使用db-tools的demo项目，你需要在demo项目中如下声明：

```xml
<project>
  <groupId>com.example</groupId>
  <artifactId>db-tools-mysql-demo</artifactId>
  <version>1.0.0</version>
  <dependencies>
    <dependency>
      <groupId>com.example</groupId>
      <artifactId>db-tools-project</artifactId>
      <version>1.0.0</version>
    </dependency>
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>8.0.11</version>
    </dependency>
  </dependencies>
  ...
</project>
```

在实际应用中，比起在给一个项目添加很多可选以来，不如将项目划分为多个子模块，每个模块引用各自需要的依赖。项目结构反而清晰很多，别人引用起来也更方便。比如上面的例子就可以划分成`db-tools-mysql`和`db-tools-mongo`两个子模块。

---
## 依赖的传递性
在讲可选依赖时提到了一个词——传递性，也就说项目A依赖于项目B，而项目B又依赖于项目C。那么项目C对于项目A而言就是传递性依赖。如果项目B还有两个可选依赖D和E，那么D和E对于项目A就不存在传递性，但A的运行又依赖于D或E，那么A就需要在自己的pom中显式添加对D或E的依赖。

Maven会解析项目各个直接依赖的POM，然后将那些必要的间接依赖以传递性依赖的形式引入到当前项目中。不过这其中也有可能有Maven搞不定的情况，比如依赖冲突，或者你想用直接依赖替换掉某个传递性依赖，这时就需要人工介入了。可以使用`exclusion`移除某个依赖。

---

### 排除依赖
以下是一些可能会需要排除传递性依赖的情形：

* 某个依赖的groupId或artifactId改了，而当前项目使用新的名字引入了一个版本（比如某个传递性依赖定义为了snapshot版本，而在当前项目中想使用发布版本）。通常情况下Maven会自动解决同一依赖的版本冲突，但是由于groupId或artifactId的不同，Maven会认为是两个不同的依赖而没有处理。
    * 印象很深，以前写项目有遇到过抛找不到类的异常的情况，后来debug很久发现是某个jar包在两个不同的依赖中都有引用，结果就冲突了，JVM蠢蠢地不知道用谁就只好报错了。
* 当前项目并不会用到某个依赖，但是这个依赖又没有被标记为可选依赖。
* 某个依赖可由运行时容器提供。


---

### 依赖的传递性与依赖范围
依赖的传递性也会对依赖范围产生一定影响。以A依赖于B，B依赖于C为例，看下表：

first/second | compile | test | provided | runtime
---|---|---|---|---
compile | compile | - | - | runtime
test | test | - | - | test
provided | provided | - | provided | provided
runtime | runtime | - | - | runtime

上表第一列表示第一直接依赖范围（即A对B的依赖范围），第一行表示第二直接依赖（B对C的依赖范围），交叉部分表示传递性依赖范围（A对C的依赖范围） 

---
## 依赖管理
终于聊到依赖管理了。我们之前用来举例的项目都是hello world这种超级简单的，而实际应用中，项目复杂度通常都很高，一个项目里引用几十上百个依赖是很常见的。而且同一项目的不同模块很可能会重复引用同一个依赖。如果所有依赖都像我们之前看到的写法来声明，像version的值就有可能重复出现在多个地方。如果某天某个依赖的版本要升级，那你就得把所有引用的地方都改一遍，这是我们在写代码的时候都会极力避免的情况。

有没有一个地方可以统一管理这些版本信息呢？当然有啦！`dependencyManagement`元素就是帮你解决这个问题的~
通常`dependencyManagement`元素都会放在项目最顶层的父POM中，在`dependencyManagement`元素下的依赖声明并不会引入实际的依赖，但它能约束`dependencies`元素下的依赖使用（`dependencies`通常放在各子模块的POM中，用于声明并引用该模块需要的依赖），通过在`dependencyManagement`中统一配置version, scope等信息，可以有效消除重复，方便管理。

举个栗子。

* parent project pom.xml

```xml
<project>
  <groupId>com.example</groupId>
  <artifactId>hello-world</artifactId>
  <version>1.0-SNAPSHOT</version>
  ...
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>8.0.11</version>
      </dependency>
    </dependencies>
  </dependencyManagement>
</project>
```

* child project pom.xml

```xml
<project>
  <parent>
    <groupId>com.example</groupId>
    <artifactId>hello-world</artifactId>
    <version>1.0-SNAPSHOT</version>
  </parent>
  <artifactId>hello-jane</artifactId>
  ...
  <dependencies>
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
    </dependency>
  </dependencies>
</project>
```
`dependencyManagement`定义在父POM中，管理整个项目中会用到的依赖，而在子POM中通过`dependencies`声明子模块用到的依赖。在上面两段POM配置，hello-jane项目里的mysql依赖不用再声明version，只需配置groupId和artifactId就能从父POM中获取对应的依赖信息。

---
### import 与 dependencyManagement
上文没有详细介绍的依赖范围import，其用法与dependencyManagement相关。使用import范围的依赖通常指向一个POM，作用是将目标POM的dependencyManagement配置导入并合并到当前POM的dependencyManagement元素中。

举个栗子。

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>com.example</groupId>
      <artifactId>hello-world-parent</artifactId>
      <version>1.0-SNAPSHOT</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

---

写到这里，Maven中最最常见的几个概念都有提到了，再重复一次：生命周期、插件、坐标、仓库、依赖。然而到目前为止都是用比较零散的代码片段举例，其中也涉猎到了一些还没有提到的但非常有用的概念，比如POM的继承啦，聚合啦，项目模块的划分啦等等。

概念性的东西都挺简单的，本来Maven也是工具，最重要的还是应用。接下来我想还是用一个相对完整的项目实践一下~

未完待续~

---

## 参考资料
* *Maven: The Definitive Guide*
* 《Maven实战》






  [1]: https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html#Dependency_Scope