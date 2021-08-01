---
title: Spring中的数据访问方式
date: 2018-12-25 23:17:31
categories: 技术笔记
tags: 
- Spring
- Java
---

🎄圣诞快乐～( ´▽｀)

---

在JDBC二三事中我们提到了使用Java提供的JDBC API进行数据操作。不难发现，在使用这种相对底层的接口编码时，会出现很多类似的代码，几乎每次查询操作我们都要执行获取连接、执行查询、释放连接、处理异常、管理事务的提交与回滚一套流程，稍有不慎就会导致数据库连接泄露啊，违反了事务ACID原则啊等等。
因此，在实际的应用开发中，我们一般都会借助框架提供的更可靠更抽象的方式的进行数据访问，让框架帮我们处理这些程式化的操作，解放自己来专注于核心业务逻辑。在这篇笔记里，我们就一起来看看如何使用Spring进行数据访问吧～

Spring整合了多种持久化技术，你可以：

* 使用Spring JDBC简化JDBC API操作；
* 与JPA、Hibernate、MyBatis等ORM(Object-relational Mapping)框架集成；
* 使用Spring提供的对NoSQL数据库的支持访问MongoDB、Redis等非关系型数据库。

<!--more-->
Spring要提供一套通用的数据访问技术，整合各种持久化技术，那么“面向接口编程”自然是要遵守的首要原则。

---
## 面向接口编程
通常我们会将数据访问功能放到一个或多个专注于这项任务的组件中，这类组件一般被成为数据访问对象(Data Access Object, DAO)或Repository。

为了避免应用与特定的数据访问策略耦合在一起，我们应该以接口的方式编写Repository暴露功能，针对不同的持久化技术做具体的实现。如下图所示：
![service-repository][1]

又见熟悉的依赖倒置，这样做的好处显而易见：

* 面向接口编程，屏蔽具体的持久化技术，实现service层与持久化技术的解耦；
* 它使得service对象易于测试。它们不再与特定的数据访问实现绑定在一起，在测试时可以创建mock实现，这样无需连接数据库就可以测试service对象，提高了单元测试的效率，能更好的控制数据的一致性。

为了实现这一目标，将具体的持久化技术隐藏到接口背后，Spring做了这样一件事——**提供了一套统一的异常体系，屏蔽具体持久化技术的异常**，这套异常体系用在了Spring支持的所有持久化方案中。

---
## 统一的异常体系
直接使用JDBC API，几乎所有的访问操作都会抛出SQLException，而SQLExeception又是检查型异常，这就导致代码里无可避免的充斥着try/catch代码块。而这些代码块并没有多大价值，原因如下：

* 大多情况下，try/catch代码块中也就是记个日志，没做多少实质性工作；
* 引发异常的问题多数是不可恢复的，比如数据连接失败、SQL语句语法错误、查询的列或表不存在、更新操作违反了数据库约束等

当然啦，其根本原因是JDBC的异常设计过于简单，它将所有与数据访问相关的异常都塞在了一个通用异常SQLException里。并且，SQLException中包含的具体意思是由具体的持久化技术决定的。

如果使用持久化框架，比如Hibernate。这类框架自身会封装一部分异常，提供相对丰富的异常体系，分别对应 特定的数据访问问题。但这些异常又与持久化框架相关了。

作为更上层的Spring，需要的是具有一定描述性的且与特定持久化框架无关的异常。于是Spring定义一套自己的异常体系。

---
### DataAccessException和它的继承者们
Spring在org.springframework.dao包中提供了一套完备的异常体系，这些异常都集成与DataAccessExecption，DataAccessExecption继承于NestedRuntimeException。这样做一方面保留了原始的异常信息，只要你愿意，就可以通过嵌套的异常找到根本原因；另一方面，将JDBC API抛出的检查型异常转换成了非检查型异常，这样就避免了泛滥成灾的try/catch代码块。当然，你还是可以根据需要捕捉你感兴趣的异常，以进行后续处理。

下面这张图涵盖了DataAccessException和它的儿辈孙辈们～它们各自也还可以衍生出更多的子类～
![DataAccessException and its subclasses][2]
可以看到相比于一个大一统的SQLException，DataAccessException的继承者们的自描述性详细了很多，通过类名就能了解异常所代表的语义。

Spring的这套异常体系具有高度的可扩展性，如果需要引入一种新的持久化技术，只需为其定义对应的自异常就可以了。

---
### 异常转换器
为了将JDBC API的SQLException或者是其他持久化框架特定的异常转换为Spring定义的异常，Spring为JDBC和各个框架分别提供了不同的异常转换器。

🌰「举例时间」
以JDBC为例，在org.springframework.jdbc.support包中定义两个SQLExceptionTranslator接口，它的两个实现类SQLErrorCodeSQLExceptionTranslator和SQLStateSQLExceptionTranslator分别负责处理SQLException中的错误码和SQL状态码。

像JPA的话，在org.springframework.orm.jpa包里定义了一个类EntityManagerFactoryUtils担任了异常转换器的角色。

其他持久化框架这里就不做介绍啦～

---
## 数据访问模版化
在本文开头我们提到数据的访问过程是有一套固定步骤的，无论我们使用什么持久化技术，都需要一些特定的数据访问步骤。比如，我们都需要先获取一个连接并在数据处理完成后释放资源，这些步骤是固定的部分，不过每次数据访问的具体操作不尽相同，这属于数据访问过程中变化的部分。

Spring将数据访问过程中固定的和可变的部分分割开来，将固定的部分放到**模版类(Template)**中，可变的部分通过**回调(Callback)**接口开放出来，用于定义具体数据访问和结果返回的操作。
![template-callback pattern][3]
如上图所示，Spring的模版类负责处理固定部分，包括资源管理
事务管理和异常处理，具体业务相关的数据访问包括查询语句、参数绑定以及结果集处理则由可变的回调实现。
通常资源的管理不是一件容易的事，能交给框架来做简直不能更棒。这样不仅提高了开发效率，同时保证了资源使用的正确性。（妈妈再也不用担心我忘记释放资源从而引起资源泄露了～）

跟异常转换一样，Spring给不同的持久化技术提供了各自的模版类，举个例子，在org.springframework.jdbc.core包中的JdbcTemplate就是为简化JDBC API调用过程提供的模版类啦。

---

本篇笔记我们主要关注于Spring提供的数据访问方式背后的思想，以面向接口编程为出发点提高框架的灵活性和扩展性，以统一异常体系为前提，采用分割模版和回调的方式简化应用开发。个人觉得还是挺有启发性的～

* **参考资料**
* 《Spring实战第四版》
* 《精通Spring 4.x》

  [1]: http://static.zybuluo.com/JaneL/dlzbapvxmhso0tqk0h1mlx16/service-repository%20%281%29.png
  [2]: http://static.zybuluo.com/JaneL/77n96xc9tpnwnv0q0so4ielq/DataAccessException.png
  [3]: http://static.zybuluo.com/JaneL/8qt69fcc3p3nepc480xec8vp/template-callback.png