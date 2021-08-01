---
title: 「认识 Tomcat」先导篇之 Servlet 是什么?
date: 2020-08-12 20:13:25
categories: 技术笔记
tags: 
- Java
- Servlet
---

从读书到工作，用 Java 写过一些 Web 应用，由于大多项目都是基于 Spring 开始，而 Spring 作为一个非常成熟的开发框架，为开发者隐去了很多底层细节，导致自己现在对于 Java Web 的基本运作原理的理解远不够深刻。为了弄明白 Java Web 到底是怎么一回事，我决定从 Tomcat 入手，好好捋一捋。

为什么是 Tomcat 呢？我们来看看 Tomcat 的官方介绍：
> The Apache Tomcat® software is an open source implementation of the Java Servlet, JavaServer Pages, Java Expression Language and Java WebSocket technologies.

还有什么能比 Tomcat 更合适用来学习 Java Web 相关的知识吗？

那么，在我们深入了解 Tomcat 之前，先来认识一下 Java Web 中最基础的内容 —— Servlet。
<!--more-->

---
# Servlet 与 Servlet Container
**Servlet** 这个单词看起来有点别扭，但知道它从何而来就很好理解了：
**Servlet 即 "Server Applet"**， 意思就是**运行在 Web Server 上的小程序 (Applet)**，这些小程序接收并响应来自 Web 客户端的请求。

**Servlet Container**, 顾名思义，是装载这些 Servlet 的容器，并负责管理 Servlet 的生命周期。Tomcat 就是一款非常成熟的 Servlet Container，本系列笔记后续的内容将围绕 Tomcat 是如何实现 Servlet Container 的功能展开。在此之前，我们还是再聊聊 Servlet 吧。

在 Java 中，Servlet 声明为接口，主要定义如下，非常简洁：
```java
package javax.servlet;
public interface Servlet {
    void init(ServletConfig config);
    void service(ServletRequest req, ServletResponse res);
    void destroy();
    String getServletInfo();
    ServletConfig getServletConfig();
}
```
最重要的 `init()`, `service()` 和 `destroy()` 三个方法也体现了 Servlet 的生命周期：
1. **初始化**
2. **处理请求，返回响应**
3. **被销毁**

下面我们稍微展开介绍一下这段代码中涉及到的另外三个接口 `ServletConfig`, `ServletRequest` 和 `ServletResponse`。

---
# ServletConfig 与 ServletContext
在 Servlet 初始化时 Servlet Container 会传入一个 `ServletConfig` 对象，提供一些初始化参数和一个 `ServletContext` 对象。
下面是 `ServletConfig` 接口的声明，也很简洁：
```java
public interface ServletConfig {
    ServletContext getServletContext();
    String getServletName();
    String getInitParameter(String name);
    Enumeration<String> getInitParameterNames();
}
```

`ServletContext` 是一个非常重要的接口，Container 提供者负责提供该接口的实现。Servlet 通过 `ServletContext` 与 Servlet Container 通信。通过 `ServletContext`，Servlet 可以记录日志、获取资源的URL、与同一个 context 下的其他 Servlet 共享属性等。

下面是 `ServletContext` 声明的部分接口：
```java
public interface ServletContext {
    String getContextPath();
    // 记日志
    void log(String msg);
    // 获取资源
    URL getResource(String path);
    InputStream getResourceAsStream(String path);
    // 获取和配置初始化参数
    String getInitParameter(String name);
    boolean setInitParameter(String name, String value);
    // 获取和配置 Attributes
    Object getAttribute(String name);
    void setAttribute(String name, Object object);
    // 添加和配置 Servlets
    ServletRegistration.Dynamic addServlet(String servletName, String className);
    // 添加和配置 Filters
    FilterRegistration.Dynamic addFilter(String filterName, String className);
    // 添加和配置 Listeners
    void addListener(String className);
    // ...
}
```
Web 服务器上的每一个 Web 应用都关联了一个 `ServletContext` 实例。

---
# ServletRequest 与 ServletResponse

`ServletRequest` 包含了客户端请求的所有信息。Servlet Container 在收到客户端请求后，负责将其封装成 `ServletRequest` 实例作为入参传递给 Servlet 的 `service` 方法。特定的协议请求可以继承该接口并扩展提供协议相关的数据。`javax.servlet.http.HttpServletRequest` 就继承了该接口并封装了 HTTP 协议特定的数据，比如 HTTP Header，HTTP Session 等。

`ServletResponse` 定义了一组接口用于 Servlet 响应请求。`ServletResponse` 的创建也有 Servlet Container 负责，并在调用 Servlet 的 `service` 方法时作为入参传入。

下面是一张简单概括 Servlet, ServletConfig 和 ServletContext 关系的 UML 图：
![Java Servlet][1]

---

**参考资料**
* Java Servlet Specification 4.0

[1]:/uploads/images/java-servlet.svg