---
title: JDBC二三事之ResultSet
date: 2018-12-09 16:29:30
categories: 技术笔记
tags: 
- Java
- JDBC
---

<del>本期日常：真是一个高(guan)产(shui)的周末呢~</del>

在前面一篇笔记里，我们提到对于查询拿到的ResultSet也是可以进行增删改查操作的，今天就来聊聊这波又是什么操作~

# ResultSet与增删改查
默认情况下，一个ResultSet实例是既不可以用于更新操作，也不可以前后随意移动访问数据的，也就是说我们拿到一个默认行为的ResultSet实例就只能调用它的next()方法，依次顺序访问每一行数据。
如果我们要做更复杂的操作，需要前后移动cursor的位置，或者要通过ResultSet做更新操作，就需要在创建Statement实例时指定ResultSet的类型，比如这样：

```java
Statement stat = conn.createStatement(ResultSet.TYPE_SCROLL_SENSITIVE, ResultSet.CONCUR_UPDATABLE);
```
<!--more-->

这个createStatement方法接受两个参数，分别是：
* **resultSetType**指定ResutlSet的类型，一共有三个选项，`ResultSet.TYPE_FORWARD_ONLY`、`ResultSet.TYPE_SCROLL_INSENSITIVE`和`ResultSet.TYPE_SCROLL_SENSITIVE`。默认值是`TYPE_FORWARD_ONLY`。
* **resultSetConcurrency**：设定ResultSet是否可用于update操作，有两个选项，`ResultSet.CONCUR_READ_ONLY`和`ResultSet.CONCUR_UPDATABLE`。默认值是`ResultSet.CONCUR_READ_ONLY`。

## ResultSet.updateRow
将ResultSet设置为`CONCUR_UPDATABLE`之后，我们就可以调用ResultSet的`updateXXX`方法来更新数据，然后调用`updateRow()`方法将更改存到数据库。举个例子：

```java
public static void updatePrice(double percentage) throws SQLException, IOException {
    Connection conn = DBConnector.getDBConnection();
    String query = "SELECT * FROM books";
    try (Statement stat = conn.createStatement(ResultSet.TYPE_SCROLL_SENSITIVE, ResultSet.CONCUR_UPDATABLE)) {
        ResultSet rs = stat.executeQuery(query);
        while (rs.next()) {
            double price = rs.getDouble("price");
            rs.updateDouble("price", price * percentage);
            rs.updateRow();
        }
    }
}
```

**NOTE #1**：如果在调用`updateXXX`对当前行修改之后没有调用`updateRow`就把cursor移走了，那之前的修改就丢失了哦。

**NOTE #2**：有的查询可能不会返回可更新的ResultSet，比如一个join了很多表的查询结果。在不确定的时候，可以调用getConcurrency()方法确认一下。（一时没有想到好的例子，仅希望在脑中留下印象，以后说不定就遇到类似情况呢。）

---
## ResultSet.insertRow
除了可以更改数据之外，使用ResultSet.insertRow方法还可以新增数据。
举个例子：
```java
public static void insertBook(String title, int publisherId, double price) throws SQLException{
    Connection conn = DBConnector.getDBConnection();
    String query = "SELECT * FROM books";
    try (Statement stat = conn.createStatement(ResultSet.TYPE_SCROLL_SENSITIVE, ResultSet.CONCUR_UPDATABLE)) {
        ResultSet rs = stat.executeQuery(query);
        rs.moveToInsertRow();
        rs.updateString("title", title);
        rs.updateInt("publisher_id", publisherId);
        rs.updateDouble("price", price);
        rs.insertRow();
        rs.moveToCurrentRow();
    }
}
```

---
## ResultSet.deleteRow
删除操作就很简单啦，直接调用`deleteRow`方法就会删除当前cursor指向的行了。
```java
rs.deleteRow();
```

***Note**：仍然需要强调的是，用ResultSet来做增删改查的效率远不如直接写SQL语句交给数据库去做。ResultSet更适用于交互式场景。*

---

上文我们了解了如何使用ResultSet进行增删改查操作，但是ResultSet有一个最主要的问题：在执行这些操作时，ResultSet实例要一直保持与数据库的连接。如果一个用户在执行操作的过程中花费了很长的时间，这个数据库连接就会一直被占用，非常浪费资源。针对这种场景，我们可以使用RowSet。

# 长江后浪推之RowSet
RowSet继承自ResultSet，比ResultSet更灵活好用。
RowSet又有五个直系子类（接口）继承自RowSet，分别是：
* JdbcRowSet
* CachedRowSet
* JoinRowSet
* FilteredRowSet
* WebRowSet

我们可以将它们分为两大阵营：

* **Connected** RowSet：包含**JdbcRowSet**，它跟ResultSet一样，一直保持与数据库的连接。通常JdbcRowSet会作为ResultSet的Wrapper，以加上scrollable或updatable等属性。
* **Disconnected** RowSet：包含**CachedRowSet**, **JoinRowSet**, **FilteredRowSet和WebRowSet**。它们不需要保持数据库的连接，只在获取和更新时需要连接数据库。

关系图如下所示：
![rowset and its children][1]

关于RowSet在这里就在只简单提一下了，详情请戳[Using RowSet Objects][2]

---

写到这里，今日份的耐心已耗尽，碍于一种承上启下的纠结心理，水了一篇。总结下来，就是了解了一些JDBC ResultSet的内容。对于具体的接口怎么用这类细节，我个人倾向于实际使用是去查官方文档最为靠谱，所以这篇就......不出意料地烂尾了 ˊ_>ˋ

* **参考资料**
* [Java Tutorial - JDBC Basics][1]
* *Core Java 2, Volume 2: Advanced Features*

  [1]: http://static.zybuluo.com/JaneL/ror73c2il7d9sgt16gyqmfc0/jdbc-row-set.png
  [2]: https://docs.oracle.com/javase/tutorial/jdbc/basics/rowset.html