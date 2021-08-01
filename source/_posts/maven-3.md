---
title: Maven学习（三） - 生命周期与插件
date: 2018-05-12 21:07:43
categories: 工具
tags: Maven
---
当我们运行`mvn clean install`命令时，这里的`clean`和`install`分别表示不同**生命周期(**lifecycle)中的不同**阶段**(phase)。

## 什么是生命周期 （Lifecycle）？
生命周期(lifecycle)是Maven的核心概念之一，它好比一个抽象统一的模板，里面包括了项目的清理、初始化、编译、调试、打包、集成测试、验证、部署和站点生成等几乎所有项目构建步骤。
<!--more-->
Maven一共有三套内置生命周期(built-in build lifecycles)，分别为：
* clean - 负责项目清理
* default - 负责项目的构建和部署
* site - 负责项目站点文档的创建

生命周期本身是一个抽象概念，在应用到具体项目时，通过POM文件的配置，我们可以指定在生命周期的每个阶段应该使用什么插件。当运行项目构建命令时，Maven就会根据配置执行具体的插件目标。

---
### 生命周期阶段 （Phase）
每个生命周期都由一些不同的阶段(phase)组成，这些阶段是有一定顺序的，后面的阶段依赖于之前的阶段。

**Note** *这里不要弄混淆的是，三个生命周期是相互独立的，它们之间不存在依赖关系。*

我们在使用Maven时最直接的方式就是用命令调用这些生命周期阶段。

---
### clean 生命周期
clean生命周期的目的是清理项目，它比较简单，只包含三个阶段：
* pre-clean 执行清理前需要完成的工作
* clean 清理上一次构建生成的文件
* post-clean 执行清理后需要完成的工作

---
### default 生命周期
default生命周期定义了项目构建时所需要的所有步骤，是所有生命周期中最重要的一部分。下面列出了其中最为重要的几个阶段（不是全部）：
* validate - 验证项目的正确性和必要信息
* compile - 编译项目源代码
* test - 使用单元测试框架运行测试，测试代码不会被打包或部署
* package - 将编译好的代码打包成可发布的格式，比如JAR
* verify - 运行集成测试，保证质量标准都已达标
* install - 将包安装到本地仓库，供本地其他Maven项目使用
* deploy - 将最终的包复制到远程仓库，供其他开发人员和Maven项目使用

想要查看完整的生命周期及其各阶段请戳官网 [**Lifecycle Reference**][1]

---
### 再看`mvn clean install`
在了解了生命周期和生命周期阶段之后，我们再来看这条命令，实际上它就是告诉Maven分别调用：
* clean生命周期的clean阶段
* default生命周期的install阶段

按照生命周期阶段之间的前后依赖关系，实际上执行的是：
* clean生命周期的pre-clean和clean阶段
* default生命周期的从validate一直到install的所有阶段

这条命令可以说是在日常开发中最常用的了，敲黑板，**在项目构建之前先清理是一个很好的实践**喔。

---

## 插件绑定
虽然我们说每个阶段代表了一个生命周期中的某一步骤，但是在这一步骤中实际执行的行为可能是多样的。

就好比把做一道菜的流程看做一个生命周期，我们把它简单分割成备菜，烹饪，装盘三个阶段。看起来还是比较标准正常的一个过程吧，而且各个阶段也是有前后依赖关系。当你真正在准备一道菜时，对应每个阶段要做的事情会根据这道菜的需求有对应的具体的任务。比如说炒菜和蒸菜，对烹饪阶段的需求就完全不同了。

回到项目构建上来，比如在package阶段，打jar包和打war包要执行的操作也不同。

那么如何为一个阶段指定行为呢？就是通过**为某个阶段绑定特定的插件目标(plugin goal)**来实现的。
先来张图直观感受一下（图引自*Maven: The Definitive Guide*）
![phase-goal-binding][2]
*注：上图的Goals中如`compiler:compile`, compiler代表插件，compile代表插件目标。*

---
### 插件目标（Plugin  Goal）
对于Maven插件来说，一个插件通常能完成多个功能，每个功能都可以看做是一个插件目标。
像这样（图引自*Maven: The Definitive Guide*）：
![plugin-goals][3]

一个插件目标可以被绑定到零个或多个生命周期阶段上：
* 如果一个插件目标被绑定到多个阶段，每个阶段它都会被执行
* 没被绑定给任何阶段的目标可以在生命周期之外直接调用<br>举个栗子，`mvn dependency:tree` 这里就是直接从命令行调用maven-dependency-plugin插件的tree目标，它的功能就是分析项目依赖树，运行截图如下，
    ![dependency:tree][4]

一个生命周期阶段也可以被绑定零个或多个插件目标：

* 如果某个阶段没有绑定插件目标，那么它就不会被执行
* 如果某个阶段被绑定了多个插件目标，那么所有目标都会按照声明的顺序依次执行

---
### 内置绑定
实际上，我们在创建hello-world的项目时，它的pom文件里并没有做任何插件绑定，但当你运行`mvn clean install`命令时，会发现很多生命周期阶段都已经有绑定好的插件目标了，运行截图如下，

![mvn-clean-install][5]

从上图看出，有黄色字符大致格式是`plugin:goal`，表示在执行某个插件的某个目标。这些都是Maven为了让用户几乎不用做任何配置就能构建Maven项目，而内置好的绑定。
此处还可以验证上图中没有出现clean生命周期的pre-clean阶段，因为Maven内置并没有给pre-clean阶段绑定任何插件目标，所以这个阶段没有被执行。

**Note** *内置绑定与项目的打包类型相关。比如package为jar的项目默认绑定的插件目标与package为war的项目默认绑定目标不同。很好理解，因为要打成不同的包，output不同，在每个阶段要做的事情自然也不同。*

想查看更多内置绑定相关内容请戳 [**Built-in lifecycle bindings**][6]

---
### 自定义绑定
除了内置绑定外，Maven也允许用户自定义将某个插件目标绑定到某个生命周期阶段上。
举个栗子，为了创建项目的源码jar包，因为内置绑定中没有这一任务，这就需要用户自行配置。代码如下：

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-source-plugin</artifactId>
      <version>3.0.1</version>
      <executions>
        <execution>
          <id>attach-sources</id>
          <phase>verify</phase>
          <goals>
            <goal>jar-no-fork</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```
上面这段配置就将maven-source-plugin插件的jar-no-fork目标绑定在了verify这个生命周期阶段上。运行截图如下：
![mvn-selfdefined-pluginbinding][7]
从上图可以看到，在package和install之间多执行了自定义的插件绑定任务jar-no-fork，将源码打成hello-world-1.0-SNAPSHOT-sources.jar包并在install阶段将其安装到了本地仓库。

**Note** *很多插件目标在编写时就一定设置了默认绑定的生命周期阶段，例如上例中的jar-no-fork目标，它的默认绑定阶段时package。因此即使在plugin配置中不指明phase为verify，调用install也是没问题的。*

**Tips** *调用`mvn help:describe -Dplugin=org.apache.maven.plugins:maven-source-plugin:3.0.1 -Ddetail`可以查看这个plugin的详细信息哦~*

---
## 插件配置
在将插件目标绑定到某个生命周期阶段时，我们还可以通过`<configuration>`元素配置插件的参数。

### 插件全局配置 
在声明插件的时候对插件进行全局配置，这样所有绑定了这个插件的某个目标的任务都会使用这些配置。
举个栗子，

```xml
<properties>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  <maven.compiler.source>1.7</maven.compiler.source>
  <maven.compiler.target>1.7</maven.compiler.target>
</properties>
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>3.1</version>
      <configuration>
        <source>${maven.compiler.source}</source>
        <target>${maven.compiler.target}</target>
        <encoding>${project.build.sourceEncoding}</encoding>
        <compilerArguments>
          <extdirs>libs</extdirs>
        </compilerArguments>
      </configuration>
    </plugin>
  </plugins>
</build>
```
上面这段配置给maven-compiler-plugin指定了编译的Java版本。这样不管是执行compiler:compile还是compiler:testCompile，都会基于Java 1.7版本编译。

---
### 插件目标配置
除了在全局级别配置插件参数外，还可以在`<execution>`元素下为每个goal配置各自的参数。
举个栗子, 以下配置给pre-clean阶段绑定了maven-antrun-plugin的目标run，并且在给其配置了一个任务:输出一条log。

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-antrun-plugin</artifactId>
  <executions>
    <execution>
      <id>file-exists</id>
      <phase>pre-clean</phase>
      <goals>
        <goal>run</goal>
      </goals>
      <configuration>
        <tasks>
          <echo>Deleting ${project.build.finalName}.${project.packaging}</echo>
        </tasks>
      </configuration>
    </execution>
  </executions>
</plugin>
```
运行截图如下：
![pre-clean-echo][8]

更多插件配置相关信息请戳 [**Guide to Configuring Plug-ins**][9]

未完待续~

---
## 参考资料
* 官方入门[Maven Getting Started Guide][10]
* 官方文档[Introduction to the Lifecycle][11]
* 官方文档[Guide to Configuring Plug-ins][12]
* *Maven: The Definitive Guide*, Chapter 10
* 《Maven实战》第7章


  [1]: http://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html#Lifecycle_Reference
  [2]: http://static.zybuluo.com/JaneL/qojpcb473pu9gmy9yvyfmbu5/image.png
  [3]: http://static.zybuluo.com/JaneL/n2j3lajn2h7zh7i15vazqcvs/image.png
  [4]: http://static.zybuluo.com/JaneL/7f84cbwvuf9wn19wt3arhr0f/image.png
  [5]: http://static.zybuluo.com/JaneL/jpm66bqyiuqmbz9fd4ny9v4r/image.png
  [6]: http://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html#Built-in_Lifecycle_Bindings
  [7]: http://static.zybuluo.com/JaneL/eliqspubou45j3sbku2f93tz/image.png
  [8]: http://static.zybuluo.com/JaneL/q5sgtw6z4mfgclyfy0tiymyw/image.png
  [9]: https://maven.apache.org/guides/mini/guide-configuring-plugins.html
  [10]: http://maven.apache.org/guides/getting-started/index.html
  [11]: http://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html#
  [12]: https://maven.apache.org/guides/mini/guide-configuring-plugins.html