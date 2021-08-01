---
title: Maven学习（四）- 坐标和仓库
date: 2018-05-14 19:59:26
categories: 工具
tags: Maven
---

## 精准定位之Maven坐标
POM作为Maven中最最重要的部分，无论是在项目的基本信息，项目构建过程中插件的管理，还是项目依赖管理，所有的这些信息都使用POM来描述的。在POM中我们使用一组定位符来唯一标识一个项目，一个插件或者是一个依赖，这组定位符就被是Maven**坐标（Coordinates）**。
<!--more-->

Maven坐标包括如下组成部分：

* **groupId** - 创建项目的组织，通常是这个组织域名的逆序表示。比如Apache Software Foundation下的项目都隶属于*org.apach*这个groupId。
* **artifactId** - 属于某个groupId下的单个项目的唯一标识符。
* **version** - 项目当前所处的版本。
* **packaging** - 项目的打包方式。
* **classifier** - 用于辅助定义构建输出的一些附属构件。比如主构件是hello-world-1.0-SNAPSHOT.jar，那么对应这个项目可能还有Java文档，那么它的附属构件就包括hello-world-1.0-SNAPSHOT-javadoc.jar。这里的*javadoc*就是这个附属构件的classifier。

上面五个元素，packaging是可选的，classifier不能直接定义，另外三个是必须有的~

回顾一下我们的helloworld项目的pom.xml中的项目基本信息
```xml
<groupId>com.example</groupId>
<artifactId>hello-world</artifactId>
<version>1.0-SNAPSHOT</version>
```
packaging省略了，默认就是jar包。那么当我们运行mvn install把helloworld打包成hello-world-1.0-SNAPSHOT.jar并安装到本地仓库之后呢，在其他项目中就可以通过groupId, artifactId, version和packaging定位并使用它了。

比如在之前创建的hello-world-web项目中添加如下依赖：
```xml
<dependency>
  <groupId>com.example</groupId>
  <artifactId>hello-world</artifactId>
  <version>1.0-SNAPSHOT</version>
</dependency>

```
再运行一下mvn dependency:resolve，可以看到hello-world-1.0-SNAPSHOT.jar包在依赖列表下。运行截图如下：
![dependency-hello-world][1]

上图中显示的`groupId:artifactId:packaging:version`是Maven中坐标的常见格式。

有了Maven坐标，我们就能在偌大的Maven世界中精准定位每一个artifact了。那Maven中有这么多的artifacts，它们都是如何存放的呢？

---
## Maven仓库
Maven把所有Maven项目共享的构件（artifact，可以是插件，依赖或任何项目构建的输出）存放在一个统一的位置，这个位置就是Maven仓库。不做任何配置的情况下，Maven在运行时会从默认仓库即中央仓库下载所需构建。
这个中央仓库的地址可以在Maven的安装文件中找到，使用解压工具打开`$M2_HOME/lib/maven-model-builder-3.5.3.jar`包，找到`org/apache/maven/model/pom-4.0.0.xml`，可以看到如下配置：
```xml
<repositories>
  <repository>
    <id>central</id>
    <name>Central Repository</name>
    <url>https://repo.maven.apache.org/maven2</url>
    <layout>default</layout>
    <snapshots>
      <enabled>false</enabled>
    </snapshots>
  </repository>
</repositories>
```
配置中的`<url>`元素指定的值https://repo.maven.apache.org/maven2 就是3.5.3版本的Maven的默认中央仓库了。所有Maven的核心插件都放在这个仓库中。
在安装maven那一篇笔记中曾提到，第一次运行mvn命令时，会看到控制台里打印出了很多`Download from central...`的日志，这是因为Maven的安装文件非常精简，只包含了安装必须的部分，很多插件只有在首次使用时才会从远程仓库下载到本地仓库（即前文中提到的`~/.m2/repository`），之后再使用就是直接调用本地的了。

---
### 仓库的布局
仓库的布局跟Maven的坐标十分接近，Maven仓库是基于简单文件系统存储的。将仓库的布局与坐标结合可以得到一个通用的构件相对路径（相对仓库根目录）：`/<groupId>/<artifactId>/<version>/<artifactId>-<version>.<packaging>`

---
### 仓库的分类
![maven-repository-type][2]

#### 中央仓库
前文已经提到了，中央仓库是Maven默认的远程仓库。它包含了世界上绝大多数流行的开源Java构件及其源码和其他信息。

#### 远程仓库
除了中央仓库外，还有很多其他的公开仓库，比如JBoss仓库（http://repository.jboss.org/maven2/）。
要在项目中使用出中央仓库外的其他远程仓库，可以在POM配置该仓。举个栗子，下面这段配置添加了JBoss远程仓库：
```xml
<project>
  ...
  <repositories>
    <repository>
      <id>jboss</id>
      <name>JBoss Repo</name>
      <url>http://repository.jboss.org/maven2/</url>
      <releases>
        <enabled>true</enabled>
      </releases>
      <snapshots>
        <enabled>false</enabled>
      </snapshots>
      <layout>default</layout>
    </repository>
  </repositories>
</project>
```
#### 私服
私服是一类特殊的远程仓库，一般为了节省带宽和时间，会在局域网内搭建一个私有的仓库服务器，用于代理所有外部的远程仓库。
Maven下载构件时，会先请求私服，如果私服不存在，则从外部的远程仓库下载并缓存在私服。
内部的项目也可以部署到私服上供其他项目使用。一般在公司内部都会架设私服便于内部开发。
私服带来的便利：节省外网带宽，加速Maven构建，部署第三方构件，提高稳定性，降低中央仓库负荷。
*感兴趣的朋友可以尝试一下使用[**Nexus OSS**][3]搭建私服~*

#### 本地仓库
之前在介绍`~/.m2`时已经提到，Maven默认会在用户目录下创建一个`.m2/repository/`目录作为本地仓库。在Maven项目中没有如`lib/`这样用于存放依赖文件的地方，当Maven执行编译或测试时，如果需要依赖文件，它会基于坐标先到本地仓库中查询。如果没有找到，则从远程仓库下载到本地。<br>当我们在本地的某个项目中运行`mvn install`时，也会把当前构建的项目输出到本地仓库，这样本地的其他依赖于它的项目就可以添加依赖并使用它了。<br>如果想要修改本地仓库的目录，可以在`~/.m2/settings.xml`中添加配置:
```xml
<localRepository>wherever you want</localRepository>
```

---
### 仓库的镜像
如果仓库A可以提供仓库B中的所有内容，就可以认为A是B的一个镜像。通过配置镜像可以优化Maven构建过程，比如为中央仓库配置一个国内的镜像，就能提高访问速度。
通常，可以将镜像与私服结合使用。因为私服可以代理任意外部的公共仓库（包括中央仓库），对于内部用户而言，使用私服地址就相当于使用所有的外部仓库。通过将配置集中到私服，可以简化Maven本身的配置。
举个栗子，在settings.xml中添加如下代码，就可以将私服作为镜像啦：
```xml
<settings>
  <mirrors>
    <id>nexus</id>
    <name>Nexus</name>
    <url>http://localhost:8081/nexus/content/groups/public</url>
    <mirrorOf>*</mirrorOf>
  </mirrors>
</settings>
```
更多关于镜像配置请戳 [**Using Mirrors for Repositories**][4]

未完待续~

---
## 参考资料
* *Maven: The Definitive Guide*
* 《Maven实战》
* 官方文档 [*Introduction to Repositories*][5]


  [1]: http://static.zybuluo.com/JaneL/7w6k0h3jgmhqup9uo4x5m6uc/image.png
  [2]: http://static.zybuluo.com/JaneL/mvo9pnwmmb0pwib0xqj4cav9/maven_repostiory.png
  [3]: https://www.sonatype.com/nexus-repository-oss
  [4]: http://maven.apache.org/guides/mini/guide-mirror-settings.html
  [5]: http://maven.apache.org/guides/introduction/introduction-to-repositories.html