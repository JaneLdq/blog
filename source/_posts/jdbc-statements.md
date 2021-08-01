---
title: JDBC二三事之Statement
date: 2018-12-08 14:03:07
categories: 技术笔记
tags: 
- Java
- JDBC
---

本期日常：下雪啦。如果能窝在温暖的房间里，和家人朋友吃着火锅，聊着天，一起欣赏漫天飞舞的雪花，想想就觉得很美好呢。
然而，此刻的我却只能孤独地在冷飕飕的房间里瑟瑟发抖，敲打着冰冷的键盘，流泪～（好啦，并没有(´･ω･`)

---

上一篇笔记里，我们历经千辛万苦连上了数据库，接下来终于可以愉快地增删改查了～

在JDBC中，有三类SQL语句：
* **Statement**
* **PreparedStatement**
* **CallableStatement**

接下来，我们就来分别看看它们的用途吧~
<!--more-->

# 使用Statement执行简单查询
Statement主要用于实现最最简单的不包含任何输入输出参数的SQL语句。
使用Statement执行简单查询的基本流程如下：
1. 使用Connection创建Statement
2. 定义SQL语句，调用Statement执行SQL语句
3. 处理结果ResultSet

举个例子：
```java
Connection conn = DBConnector.getDBConnection();
try (Statement stat = conn.createStatement()) {
    ResultSet rs = stat.executeQuery("SELECT title, id FROM books");
    while (rs.next()) {
        System.out.println("#" + rs.getInt(2) + " " + rs.getString("title"));
    }
}
```

值得注意的几点：
1. 使用ResultSet获取数据时如果采用index的方式，它是从1开始计数的！（作为一个让早已习惯从零开始的程序员表示这简直...反程序员啊/。\）如上例中，因为SQL语句中是按"title, id"的顺序读取的，所以这里rs.getInt(2)取的是第二列id；除此之外，也可以显式指明列名，如上例中rs.getString("title")的方式。**使用index效率更高，使用列名代码可读性更好。**
2. 每个Statement实例使用之后要**及时关闭释放资源**，上例中使用的`try-with-resources`语句，在try语句块执行结束后会自动释放stat。更保险一点，也可以把Connection放在`try-with-resources`语句里。

---
# 使用PreparedStatement提升效率和安全指数
PreparedStatement继承自Statement，它与Statement的不同之处在于，在创建PreparedStatement实例时就要指定SQL语句，这个语句会被立即传给DBMS编译，这样PreparedStatement实例包含的是一个预编译好的SQL语句，在后续执行时就可以直接运行，以提升查询效率。（而Statement实例是在每次执行时才传入SQL语句，然后每次都要先编译再运行。）

PreparedStatment最常见的使用方式是结合查询参数使用，传入不同参数多次执行同一SQL语句，这在业务场景中也是非常常见的。

举个非常简单的例子，某黑心出版商决定给旗下一部分书涨价，那么就可以通过如下代码操作：
```java
public void updateBookPrice(Map<Integer, Double> books) throws SQLException, IOException {
    try (Connection conn = DBConnector.getDBConnection()) {
        String query = "UPDATE books SET price = price + ? WHERE id = ?";
        PreparedStatement stat = conn.prepareStatement(query);
        for (Map.Entry<Integer, Double> e: books.entrySet()) {
            stat.setDouble(1, e.getValue());
            stat.setInt(2, e.getKey());
            stat.executeUpdate();
        }
    }
}
```
这样呢，我们就可以使用一个PreparedStatement更新多本书的价格。

**Note #1**: *以前没少用直接拼接字符串的方式执行SQL查询，然而这样做不仅**效率低**，**易出错**，如果没有做好特殊字符的转义等防范措施很容易被人使用SQL注入恶意入侵，**带来极大的安全风险**。因此，在需要绑定参数的时候，一定要记得**使用PreparedStatement**哦～*

**NOTE #2**: *PreparedStatement实例在创建它的Connection实例被关闭后就不可再使用了。不过，很多数据库会自动缓存PreparedStatement，如果一个相同的查询被Prepare两次，数据库会直接重用已编译好的查询策略。因此，多次调用prepareStatement也没有关系啦～*

**Note #3**: *对于复杂的SQL查询，**“If you can do it in SQL, don't do it in Java.”**。为什么要强调这一点，因为有时候会不想写复杂的SQL语句，而通过多次查询多个结果集后，在使用Java代码处理来得到最终结果。（Emmmm....我好像经常冒出这样的念头/。\）然而，这样做效率是十分低的，毕竟这是人家数据库调优存在的意义啊，放着不用他们会很伤心的～*

# 使用CallableStatement调用带输入输出参数的存储过程
对于一些复杂的包含了输入参数甚至输出参数的存储过程（stored procedures），简单的Statement和PreparedStatement就搞不定了，这时，我们就需要亮出第三类Statement——**CallableStatement**。看名字就知道它就是专门应对这类需求的。这里提一点，CallableStatment又是继承自PreparedStatement。可见这三类Statement在职责上是逐层增加的。

举个简单的例子。首先我们在数据库中创建一个procedure备用（实在不想费脑子设定一个有意义的场景了，单纯为了举个🌰，将就看哈～）：

```
DELIMITER $$
CREATE PROCEDURE get_book_publisher(IN book_id INT, OUT title VARCHAR(255), OUT publisher VARCHAR(255))
BEGIN
    SELECT books.title, publishers.name INTO title, publisher
    FROM books, publishers
    WHERE books.id = book_id AND books.publisher_id = publishers.id;
END
$$
DELIMITER ;
```

在上面这个procedure中，我们定义了一个输入参数`book_id`，两个输出参数`title`和`publisher`。意思呢，就是输入书的ID，输出书名和出版商名称。
在Java中调用这个procedure的代码如下所示：
```java
public String getBookPublisherById(Connection conn, int bookId) throws SQLException {
    // 类似与PreparedStatement，使用?作为参数占位符
    String query = "{call get_book_publisher(?, ?, ?)}";
    // 调用prepareCall创建一个CallableStatement实例
    try (CallableStatement stat = conn.prepareCall(query)) {
        // 设置输入参数
        stat.setInt(1, bookId);
        // 注册输出参数并指定类型
        stat.registerOutParameter(2, Types.VARCHAR);
        stat.registerOutParameter(3, Types.VARCHAR);

        stat.executeUpdate();
        // 获取输出参数
        String title = stat.getString(2);
        String publisher = stat.getString(3);
        return title + " - " + publisher;
    }
}
```


# 处理多个结果集
有的时候调用预定义的存储过程会有多个输出结果。（话说这种情况我还真的第一次尝试）
这种情况要怎么处理呢？依旧举例来说。我们在MySQL数据库中再创建一个简单的procedure：
```
DELIMITER $$
CREATE PROCEDURE get_books_publishers()
BEGIN
    SELECT * FROM books;
    SELECT * FROM publishers;
END
$$
DELIMITER ;
```
为了遍历所返回的全部结果集，我们可以这样做：
```java
public void getBooksPublisher(Connection conn) throws SQLException{
    String query = "{call get_books_publishers()}";
    try (Statement stat = conn.createStatement()) {
        // 第一步：调用execute执行查询，如果返回是个ResultSet则为true，如果是Update count或无结果集则为false
        boolean isResult = stat.execute(query);
        boolean done = false;
        while (!done) {
            if (isResult) {
                // 拿到第一个结果集或者数值
                ResultSet rs = stat.getResultSet();
                while (rs.next()) {
                    System.out.println(rs.getInt(1) + " " + rs.getString(2));
                }
            } else {
                int updateCount = stat.getUpdateCount();
                // 如果下一个不是update count则返回值为-1
                if (updateCount < 0) done = true;
            }
            // 如果有多个结果集，调用getMoreResults方法指向下一个结果集，如果没有或不是结果集，返回false
            if (!done) isResult = stat.getMoreResults();
        }
    }
}
```

---
# 使用Statement执行Batch操作
每个Statement，PreparedStatement和CallableStatement实例在被创建的时候，都包含了一个空的list，这个list里面可以添加多个SQL command，然后以Batch的形式提交到数据库。

举个例子就懂啦：
```java
public static void insertBooks(Connection conn) throws SQLException {
    // 使用Batch之前记得先禁用自动提交
    conn.setAutoCommit(false);
    String sql = "INSERT INTO books(title, publisher_id, price) VALUES (?, ?, ?)";
    try (PreparedStatement stat = conn.prepareStatement(sql)) {
        stat.setString(1, "Harry Potter And The Sorcerer's Stone");
        stat.setInt(2, 3);
        stat.setDouble(3, 106.8);
        // 调用addBatch来添加SQL命令
        stat.addBatch();

        stat.setString(1, "Harry Potter and the Chamber of Secrets");
        stat.setInt(2, 3);
        stat.setDouble(3, 142.87);
        stat.addBatch();

        stat.setString(1, "Harry Potter and the Prisoner of Azkaban");
        stat.setInt(2, 3);
        stat.setDouble(3, 124.9);
        stat.addBatch();

        // 三条命令将按顺序执行，并且它们各自影响的行数会依次存放在一个数组中返回
        int[] updateCounts = stat.executeBatch();
        
        // 执行Batch之后，之前添加的命令并不会被清空，这时可以调用clearBatch来清空，在本例中并没有必要，因为后续没有更多操作了。如果后续还要使用batch一定要注意这一点，避免重复提交。
        // stat.clearBatch();
        
        // 将Batch提交到数据库
        conn.commit();
        conn.setAutoCommit(true);
    }
}
```

需要特别注意的是，在Batch可以添加诸如更新、插入、删除语句，以及一些DDL语句比如`CREATE TABLE`、`DROP TABLE`等。但是，不能Batch中添加会返回一个ResultSet对象的语句，比如查询语句，否则会抛异常哦。

---

到此为止，我们着重介绍了使用三种statement执行SQL语句来实现增删改查操作，除此之外，使用查询得到的ResultSet以及它的子类RowSet也是可以对数据集进行增删改的，这又是些什么操作呢？我们有缘再见～

---

* **参考资料**

* [Java Tutorial - JDBC Basics][1]
* *Core Java 2, Volume 2: Advanced Features*


  [1]: https://docs.oracle.com/javase/tutorial/jdbc/basics/index.html