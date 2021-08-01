---
title: Maven学习（七）- Profile和资源过滤
date: 2018-05-28 09:34:03
categories: 工具
tags: Maven
---

不比课程作业的随意性，正式项目的开发一般要经过开发-测试-部署等流程，相应的也会有多个环境，比如开发环境，测试环境，生产环境等等。相应的项目部署在不同的环境，就可能会要做一些针对环境的配置。举个栗子，数据库的配置，测试环境显然不能跟生产环境用同一个数据库。

Maven提供了Profile机制来协助项目构建在不同环境下的移植。
<!--more-->

还是以甜品店项目作为示例，假设现在我想区分dev和test用的数据库，如何利用Profile来做到呢？

## Profile一览

在父POM中（也可以在tian-service中）添加如下配置:
```xml
<profiles>
    <profile>
        <!--#1: id -->
        <id>dev</id>
        <!-- #2: activation condition -->
        <activation>
            <property>
                <name>environment.type</name>
                <value>dev</value>
            </property>
        </activation>
        <!-- #3: detail -->
        <properties>
            <db.driver>com.mysql.cj.jdbc.Driver</db.driver>
            <db.url>jdbc:mysql://localhost:3306/dessert?useUnicode=true&amp;characterEncoding=UTF-8</db.url>
            <db.user>root</db.user>
            <db.password>123456</db.password>
        </properties>
    </profile>
    <profile>
        <id>test</id>
        <activation>
            <property>
                <name>environment.type</name>
                <value>test</value>
            </property>
        </activation>
        <properties>
            <db.driver>com.mysql.cj.jdbc.Driver</db.driver>
            <db.url>jdbc:mysql://localhost:3306/dessertTest?useUnicode=true&amp;characterEncoding=UTF-8</db.url>
            <db.user>test</db.user>
            <db.password>123456</db.password>
        </properties>
    </profile>
</profiles>
```

`profiles`元素下可以定义一个或多个子元素`profile`，上例中定义了两个profile，一个是dev，一个是test，分别对应开发环境和测试环境。

1. `id` - 在命令行构建时，使用参数`-P`加上某个profile的ID就可以激活指定profile。比如`mvn -Pdev install`就会使用dev这个profile。

2. `activation`，除了使用命令行激活profile外，我们也可以在`activation`元素下预先定义某个profile被激活的条件。比如如果构建时Maven解析出`environment.type`的值为`test`，那么上例中id为test的profile就会被激活。
3. `properties` -除了propeties元素之外，根据profile作用范围的不同，可以在profile中声明的元素种类也不同，详见下文。

### Profile的激活
Profile的activation可以包含如下内容：

* `activeByDefault` - 是否默认激活，true or false。如下所示表示当前这个profile默认就是激活状态。
    ```xml
    <activation>
        <activeByDefault>true</activeByDefault>
    </activation>
    ```
* `property` - 系统属性，比如上例使用`mvn install -Denvironment.type=test`就会激活test profile
* `os` - 比如说Windows和Linux系统可以分别激活不同的profile
    ```xml
    <activation>
        <os> <!--兼容古老的XP-->
            <name>Windows XP</name>
            <family>Windows</family>
            <arch>x86</arch>
        </os>
    </activation>
    ```

* `file` - 文件存在与否激活
    ```xml
    <activation>
        <file>
            <missing>a.properties</missing>
            <exists>b.properties</exists>
        </file>
    </activation>
    ```


---

## Profile的范围
放在不同地方的profile，其作用范围也不同，而作用范围又决定了它所能影响的元素类型。

按照profile定义的位置可将其分为如下四种：
* **Project Profiles** - 在**pom.xml**文件中定义的profiles，只对当前项目有效。
* **User Settings Profiles** - 定义在**~/.m2/settings.xml**中的profiles，对本机上该用户所有的Maven项目有效。
* **Global Settings Profiles** - 定义在Maven安装目录下的conf/settings.xml中的全局profiles，对本机上所有Maven项目有效。
* **External Profiles** - 在项目根目录下新建一个单独的**profiles.xml**。Maven3已经不支持该特性。

---
## Profile中可使用的元素
Project Profile和其他三种外部Profile可使用的元素类型是不同的：

Project profile | Others
---|---
repositories, <br>pluginRepositories, <br>distributionManagement,<br> dependencies,<br> dependencyManagement, <br>modules, <br>properties, <br>reporting,<br> build| repositories, <br>pluginRepositories,<br> properties

*提问：为什么要做这样的限制呢？*

---
在了解了几种profile的作用范围后，回到日常开发中，我们可以在`~/.m2/settings.xml`中配置一个默认激活的profile，并且在其中声明一个属性`environment.type`，将其赋值为dev。代码如下：

```xml
<!--~/.m2/settings.xml-->
<profiles>
  <profile>
      <id>env-dev</id>
      <activation>
        <activeByDefault>true</activeByDefault>
      </activation>
      <properties>
        <environment.type>dev</environment.type>
      </properties>
    </profile>
</profiles>
```

这样在构建项目时，settings.xml中的`env-dev`这个profile默认被激活，它设定了environment.type的值为dev，然后在项目pom.xml中定义的`dev` profile的激活条件也被满足了，这个profile也被激活，因此构建时使用的就是`dev` profile中定义的数据库相关属性了。

我们可以运行`mvn help:active-profiles`来查看当前项目下激活的profiles。示例结果如下：
![active profiles][1]

---
## 资源过滤
前面我们谈到可以把数据库的相关配置放在Maven的property中，在POM中，我们可以使用`${propery.name}`的形式来引用属性的值，那么对于项目中非Maven的文件如何获取这些值呢？
比如在Spring框架中，DataSource的配置一般都放在contextConfiguration.xml中，而这些文件都被视为resource放在`src/main/resources`路径下。即使我们在Spring的配置文件中写成如下的形式：

```xml
<bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource" destroy-method="close">
    <property name="driverClass" value="${db.driver}"/>
    <property name="jdbcUrl" value="${db.url}" />
    <property name="user" value="${db.user}" />
    <property name="password" value="${db.password}" />
</bean>
```

在构建的时候，诸如`${db.driver}`就还是一个普通字符串"${db.driver}"，默认情况下并不会被Maven解析。

不过！Maven中负责资源处理的插件maven-resources-plugin有一个神奇的特性，叫**资源过滤(Resouce Filtering)**。激活这个特性，就能解析资源文件中的Maven属性啦。

配置很简单，为资源文件指定目录，并设置`filtering`的值为true就好啦~

```xml
<build>
    <resources>
        <resource>
            <directory>${project.basedir}/src/main/resources</directory>
            <filtering>true</filtering>
        </resource>
    </resources>
</build>
```
---

## 后记
这个系列，到此为止就先告一段落啦。

断断续续这几篇笔记也整理了三个星期。明明大部分都是很简单基础的概念，但是想自己再组织一遍语言，从头到尾捋一遍的过程还挺痛苦的。就我个人而言，对Maven这个工具的使用又加深了很多了解，这也正是记笔记的初衷，我觉得效果还是可以的~

就内容而言，笔记的粒度是偏粗的。我是工具类的东西，了解它整体的思想或重点概念比较有意思，细节的地方在使用的时候查文档或翻书都可以啦~

<完>

---
## 参考资料

* *Maven: The Definitive Guide*
    *比Maven的官方网站组织得好很多，而且其中有一部分专门用了五个示例项目介绍Maven的使用，非常详细且实用。*
* 《Maven实战》
    *感觉这本书很大程度上参考了**Maven: The Definitive Guide**，因为是中文书，想要快速了解Maven的前提下可以一读（对于我还有一层原因，为了记笔记组织语言作参考）。但有些概念感觉还是看英文更容易理解_(:з」∠)_*
* [**Maven官网**][2]
    *看书难免有滞后，新特性什么的还是要直接戳官网比较靠谱哦~*

  [1]: http://static.zybuluo.com/JaneL/3n3zamoz9h372ds7vakxm9j9/image.png
  [2]: http://maven.apache.org/