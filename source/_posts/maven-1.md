---
title: Maven学习（一）- 重识Maven
date: 2018-05-07 20:04:11
categories: 工具
tags: Maven
---

要说初次接触到Maven，应该是大一的事了，在好几个项目里都有用到，但多数时间都是其他人搭好了架子，自己就迷迷糊糊用着，一直对Maven的概念和使用不甚清晰。正好现在手头的项目也是用Maven作为项目管理工具，借此机会好好了解一番。
<!--more-->
---

## Maven是什么
Maven是一个主要服务于基于Java平台的项目的构建(build)、依赖管理(dependency)和项目信息管理工具。

### 构建工具
要把源代码变成最终可以运行的应用程序，要完成很多事情：编码、编译、运行单元测试、生成文档、打包、部署，所以这些流程的组合就是构建。

而作为程序员，你会发现每天做的很多工作都会一遍遍地经历这个流程，尤其是在修bug的时候。如果每个步骤都要人工操作，就会显得很浪费资源，那Maven就是一个能帮我们自动化构建的工具，通过配置，只需运行简单的命令就可以让Maven帮我们搞定这些繁琐的任务。

### 依赖管理工具
大多数Java项目或多或少都会使用一些第三方的开源类库，这些类库可以通过依赖的方式引用进来，而随着依赖的增多，依赖的版本冲突、重复等问题都是需要解决的。
Maven通过一套坐标系统实现了清晰地定位和管理每一个Java类库，为依赖管理提供了一个很好地解决方案。关于这个坐标系统的详细介绍请见[TODO]()

### 项目信息管理工具
在Maven的项目配置文件可以说提供了一个统一的地方来管理项目的各类信息，如项目描述、开发者列表、版本控制地址、许可证等等。

当然了，除了Maven之外，还有其他的构建解决方案，如Make，Ant等，不过这些我在项目中基本没有用过，了解很少。

---
## 使用Maven的第一步 - 安装
话不多说，正好重装系统后机子上还没有装Maven，一边写笔记，一边重温一下安装操作。

1. 安装JDK
Maven是基于Java的工具，也正因如此Maven具有跨平台的特性。
可以通过`java -version`命令来判断一下JDK有没有正确安装，如下图所示：
![java-version][1]

2. 下载Maven
到`http://maven.apache.org/download.cgi`下载Maven文件，想自己构建的可以下*-src玩耍，偷懒如我就直接选择bin啦。
将下载好的文件解压到某个地方（你想放哪儿随意），比如我放在了`D:\maven\`下。

3. 配置环境变量
* 在系统变量中添加`M2_HOME`并将其设为解压后的maven文件夹所在路径，如下图所示：
![system-variable][2]
* 在Path变量中加上`%M2_HOME%\bin`，如下图所示：
![path-setting][3]

4. 检查一下
在命令行敲入`mvn -v`，看到如下图所示就是安装好啦
![mvn-v][4]

---
### M2_HOME目录
在正式使用Maven之前，简单了解一下这个文件夹里的内容吧。目录结构如下图所示：
![m2-home][5]

* **bin** - 包含了mvn运行的脚本，主要用于配置Java命令，准备classpath和相关的Java系统属性，并执行Java命令。在命令行输入`mvn`命令时，就是在调用这些脚本。
* **boot** - 这个目录里只有一个文件，plexus-classworlds是一个类加载器框架，相比Java默认类加载器，它的语法更丰富，更方便配置，因此Maven使用这个框架加载自己的类库。一般用户不必关心。
* **conf** - 这个目录中的**settings.xml**，直接修改该文件用于全局定制Maven的行为，而对于一般用户而言，推荐大家将该文件复制至各自用户目录（一般在~/.m2/下）进行修改，这样只会在用户范围产生影响。
* **lib** - 包含了Maven运行时需要的Java类库

---
### ~/.m2
`~/.m2`文件夹在安装好Maven后并不存在，需要初始化。运行`mvn help:system`命令使Maven真正执行一个任务，这样它就会使用到Java类库，从而会初始化一个本地仓库用于存放依赖。
在运行命令时可以看到控制台log输出了很多download信息，这个时候再进到用户目录下就可以看到`.m2`文件夹了。
`~/.m2`文件夹下放置了Maven本地仓库`.m2/repository`，所有的Maven构建都被存储到此仓库中以便重用。
另外，如上所述，一般Maven用户还会将`$M2_HOME/conf/settings.xml`复制一份到`~/.m2/settings.xml`。通过使用用户范围的settings.xml，可以避免对系统中其他用户的影响。

`~/.m2`一般结构如下所示：
![m2-structure][6]

---
安装好了Maven之后，就可以愉快地使用Maven进行项目管理啦~

未完待续~

---
## 参考资料
* 《Maven实战》

  [1]: http://static.zybuluo.com/JaneL/akusy0uqhrtyg8k6gqkny7sk/image.png
  [2]: http://static.zybuluo.com/JaneL/ge0s30tqx07cv4s77deby7sw/image.png
  [3]: http://static.zybuluo.com/JaneL/6ox5s3zdgcz5hyt5cvfhss00/image.png
  [4]: http://static.zybuluo.com/JaneL/fhfgf3jr8z1thnd96qw0k168/image.png
  [5]: http://static.zybuluo.com/JaneL/7l4t2iplrg9nsvzb7svbxh28/image.png
  [6]: http://static.zybuluo.com/JaneL/565mufg5hzdbow9pxre6q6ev/image.png
