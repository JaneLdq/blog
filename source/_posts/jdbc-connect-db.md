---
title: JDBC二三事之连接数据库
date: 2018-12-07 00:03:07
categories: 技术笔记
tags: 
- Java
- JDBC
---

一个复古的主题，复习一波Java中数据库的基本操作。现在框架用起来这么方便，好多基础的概念都模糊了。

在这个话题下，首先要提到的一个词就是**JDBC**了，作为一个看到缩写就头大的人，我们就从它开始吧。

# JDBC是什么？
数据库的种类这么多，每种数据库都有其特定的协议。如果在开发或者维护Java程序过程中使用的数据库发生了变化，就意味着程序代码要根据特定的数据库进行调整。这个维护成本是非常高的。

而JDBC(Java Database Connectivity)的出现就是为了让开发者能用统一通用的纯Java语言与多种不同的数据库打交道。简单说来，**JDBC就是一组用于SQL数据库访问的Java API，采用driver manager来管理和动态绑定driver的机制来连接特定的数据库。**

<!--more-->
这样做带来了两大好处：
* 对于程序员而言，在开发Java程序时只需与JDBC API打交道，可以使用标准的SQL语句访问任意数据库。
* 对于数据库和数据库工具提供商，由他们来提供更底层一些的driver，这也使得他们有机会根据各自的协议对driver进行更好的优化。


## 4种JDBC Driver
在JDBC中定义了四类driver类型，命名简单粗暴，分别是：
* **type 1** - JDBC/ODBC bridges, 将JDBC转为ODBC，然后依赖ODBC driver连接数据库。在Java 8之后不再支持这种driver。
* **type 2** - 使用Java和原始代码混写的driver，采用数据库的client API连接数据库。如果使用这种类型的driver，就必须同时安装相关的Java包和一些平台相关的代码。
* **type 3** - 纯Java包，使用独立于数据库的协议传输数据库请求给一个服务端组件，由这个组件把请求转为特定的数据库协议。这种类型简化了客户端的部署，因为平台相关的代码都放在服务端了。
* **type 4** - 纯Java包，直接把JDBC请求转化为特定的数据库协议。

---
# 使用JDBC连接数据库
要对数据进行操作，第一件要做的事情就是先连接上数据库了。JDBC应用主要使用两个类（分别代表两种方式）连接数据库：
* **DriverManager**
* **DataSource**

下面我们将分别介绍这两种方式。

---
## 使用DriverManager连接数据库
DriverManager通过**数据库URL**建立连接，那么，数据库URL又是什么呢？

### JDBC数据库URL
JDBC使用一组类似URL的语法来描述数据源，通常包括数据库类型（vendor），以及相关的属性，比如主机名、端口号、数据库名字、用户名、密码等。不同的数据库提供商其URL的组成大体相同，但或多或少会有些小差别。这就好比每个地区的电话号码组成结构都不太一样。

通用格式如下：
> **jdbc:subprotocol:other**

上面的`subprotocol`用于指定特定的driver，`other`里面的内容与选择的`subprotocol`相关。

举两个例子，比如连接MySQL数据库或者PostgreSQL数据库：
> **jdbc:mysql://localhost/test?user=jane$password=secret**
> **jdbc:postgresql://localhost/test?user=jane&password=secret&ssl=true**

---
### 准备Driver
别急，在建立连接之前还有一件事情要做——别忘记**把要用的Driver的 JAR包放到运行时的classpath中**。

其次，别忘记**注册Driver**，不过现在大多数Driver都能自动完成注册了：很多JDBC Driver JAR包都可以自动完成driver的注册——在这类JAR包中包含一个`META-INF/services/java.sql.Driver`文件，里面包含这个driver的名字。

如果driver不能自动注册(JDBC4.0以前的driver)，则可以通过如下两种方式手动注册：
1. 静态注册，在Java程序中加载driver类：
```java
Class.forName("com.mysql.cj.jdbc.Driver").newInstance();
```
2. 动态注册，在命令行参数中指定`jdbc.drivers`属性：
```java
java -Djdbc.drivers=com.mysql.cj.jdbc.Driver demoApp
```
或者在程序中指定系统属性，还可以同时指定多个driver，采用分号分隔，如下所示：
```java
System.setProperty("jdbc.drivers", "com.mysql.cj.jdbc.Driver:org.postgresql.Driver");
```

---
### DriverManager创建连接
把准备工作做好后，建立数据库的连接就是几行代码的事了。
在本例中，我将数据库配置放在了`/resources/database.properties`中，内容如下：

> jdbc.drivers=com.mysql.cj.jdbc.Driver
> jdbc.url=jdbc:mysql://localhost/dbdemo
> jdbc.username=root
> jdbc.password=12345678

下面这段代码从database.properties中读取数据库配置，调用DriverManager获取连接：
```java
public static Connection getDBConnection() throws SQLException, IOException {
    // read database config from 'resources/database.properties' file
    Properties props = new Properties();
    try(InputStream in = DBConnector.class.getResourceAsStream("database.properties")) {
        props.load(in);
    }
    String drivers = props.getProperty("jdbc.drivers");
    // set jdbc drivers
    if (drivers != null) System.setProperty("jdbc.drivers", drivers);
    // get database config
    String url = props.getProperty("jdbc.url");
    String username = props.getProperty("jdbc.username");
    String password = props.getProperty("jdbc.password");

    // use DriverManager to get database connection
    return DriverManager.getConnection(url, username, password);
}
```

---
## 使用DataSource连接数据库
现在可以把上文介绍的DriverManager的方式忘掉了，DataSource才是真正的宠儿～它也是更为推荐的方式，其优势有如下几点：

1. 使底层的细节完全对应用的开发透明
2. 支持连接池和分布式事务（这对于企业级应用开发是非常重要的～）


### DataSource的三种实现方式
一个DataSource实例就代表了一个数据源，它可以是一个DBMS，也可以只是一个文件。DataSource自身只是定义了一组接口，具体的实现由提供商完成。

DataSource的实现可以按照其支持的功能划分为三种：
* **Basic DataSource**：可以建立标准的Connection实例，不过既不支持连接池也不支持分布式事务
* **支持连接池的DataSource**：使用连接池管理连接，使得连接可以重复使用
* **支持分布式事务的DataSource**：能提供用于分布式事务处理的连接（一个事务可通过访问多个数据源完成）。

一个JDBC Driver里至少要包含最基本的DataSource实现。举个例子，在MySQL的Driver里可以找到`MysqlDataSource`和`MysqlConnectionPoolDataSource`。而且一般支持分布式事务的DataSource也会支持连接池哦。

了解了DataSource的三种实现，那么到底怎样使用DataSource建立连接呢？

单独使用DataSource的方式跟DriverManager没有什么不同，就是创建DataSource实例，配置数据库属性，然后调用实例获取连接就可以了。示例代码如下：

```java
public class MyDataSourceFactory {

    public static DataSource getMySQLDataSource() throws IOException {
        Properties props = new Properties();
        try(InputStream in = ClassLoader.getSystemResourceAsStream("database.properties")) {
            props.load(in);
        }
        // 创建DataSource示例并配置数据库属性
        MysqlDataSource ds = new MysqlDataSource();
        ds.setURL(props.getProperty("jdbc.url"));
        ds.setUser(props.getProperty("jdbc.username"));
        ds.setPassword(props.getProperty("jdbc.password"));
        return ds;
    }
    public static void main(String[] args) throws SQLException, IOException {
        DataSource ds = MyDataSourceFactory.getMySQLDataSource();
        // 拿到DataSource示例创建连接
        Connection conn = ds.getConnection();
        try (Statement stat = conn.createStatement()) {
            // do whatever you want
        }
    }
}

```

不过像这样的代码应该只会出现在学生作业或者练习里了，在相对正式的应用开发中，一般会**把DataSource与JNDI结合使用**，这样才能更好的发挥出DataSource的几大优势。

在这类场景中，建立连接通常分为两部分：
1. 首先，由管理员之类的角色负责部署DataSource，注册到JNDI Context，通常只要一次操作就OK。
2. 然后，开发人员就可以愉快地通过JNDI的lookup获取DataSource。

（写着写着脑子冒出了饲养员儿给一群🐒投食的场景...

我们依旧按步骤来看。

#### 注册DataSource
只接触过Web应用的场景，所以这里就以Tomcat为例介绍。像Tomcat这样的Servlet Container一般都内置了对JNDI Context和资源管理的支持，作为使用者，我们只需要提供一些配置信息，像创建DataSource示例和注册JNDI这些操作都由Container接管了。

```xml
<Resource name="jdbc/demodb" 
      global="jdbc/demodb" 
      auth="Container" 
      type="javax.sql.DataSource" 
      driverClassName="com.mysql.cj.jdbc.Driver" 
      url="jdbc:mysql://localhost/demodb" 
      username="root" 
      password="secret" 
      
      maxActive="100" 
      maxIdle="20" 
      minIdle="5" 
      maxWait="10000"/>
```
将上面这段代码加到context.xml中就代表注册了一个名为`jdbc/demodb`，类型为DataSource的资源到JNDI Context中了。可以看到，除了配置基本的数据源信息之外，我们还可以指定连接池相关的属性。

Note：在Tomcat中配置DataSource也有好几种方式，不同的地方在于你把DataSource配在应用层级还是服务器层级，各有利弊：
* 配置在Application的`META-INF/context.xml`中：DataSource只能被当前应用使用，不能与其他应用共享；并且由于这个context.xml文件会被打包在war中部署到服务器，每次更新DataSource的配置，都要重新打包部署。
* 配置在Tomcat`apache-tomcat/conf/context.xml`中：会给部署在当前Container中的
每个应用创建一个DataSource。如果配置的连接池最大连接数比较多，而且部署的应用也比较多的化，会非常消耗资源。比如`maxActive=100`，然后部署了30个应用，那么就会导致一共有3000个连接。
* **Tomcat的server.xml和context.xml组合使用**：将DataSource定义在在server.xml的`GlobalNamingResources`中，然后可以选择在server的context.xml或Application的context.xml中定义`ResourceLink`来使用。

组合的方式是比较灵活且推荐的。（关于Tomcat配置DataSource的详细例子可戳参考资料第二条～

#### 在应用中通过JNDI获取DataSource
这个操作就非常简单了，两行代码搞定：
```java
Context ctx = new InitialContext();
DataSource ds = (DataSource)ctx.lookup("java:/comp/env/jdbc/demodb");
```

通过JNDI的方式，即使我们去修改配置文件中的DataSource的信息，比如换了一个数据库，只要不更改JNDI名字，对于应用开发者而言，这些变更就是完全透明的。

---

花了好几个晚上整理，常常看到一个名词，只有模糊印象，然后去查资料先了解，N多次冒出怀疑自己的理解能力不足的念头。最终这篇笔记里提到的内容多还是浅尝辄止，仅仅能作为引子，让这个模糊的印象稍微清晰了一点。

---

* **参考资料**
* [Java Tutorial - JDBC Basics][1]
* [Tomcat DataSource JNDI Example in Java][2]
* *Core Java 2, Volume 2: Advanced Features*


  [1]: https://docs.oracle.com/javase/tutorial/jdbc/basics/index.html
  [2]: https://www.journaldev.com/2513/tomcat-datasource-jndi-example-java#tomcat-datasource-jndi-configuration-example-8211-serverxml