---
title: Web认证那些事儿（三）Token-based认证
date: 2018-10-23 22:15:26
categories: 技术笔记
tags: 
- 网络安全
- 认证与授权
---

本期日常：昨天单抽出了小奶狗超开心，兴奋到决定填坑一篇，结果太激动脑子不太愿意正常运转，只好今天接着填了\_(:з」∠)\_

---

在上篇笔记中，我们提到在某些场景下Session-based认证并不是很合适，比如：

1. 如果用户访问量很大，每个用户的session信息都需要保存在服务器端，内存消耗会很大
2. 如果采用分布式部署，创建session的服务器可能和后续处理请求的服务器不同，需要考虑多台服务器session共享的问题
3. 如果不同应用之间要共享认证信息，可能还要考虑跨域问题

除此之外，还有另一个需要考虑的问题：最初用于维持会话的session id一般都会放在cookie中跟随每次请求传到服务端。这对于使用浏览器进行的网页访问是没有问题的，但是，现在很多平台都是直接对外开放API，这些API可能被Web应用请求，也可能被Native移动端应用请求，对于这些应用而言，cookie就不那么友好了。

于是乎，随着需求的不断变化，Token-based认证成为了一个冉冉升起的新星～

<!--more-->
Token-based认证的最大特点就是**服务器实现无状态化**，这是如何实现的呢？
当用户第一次认证成功后，把必要的用户验证信息和一些其他有用的信息（比如token有效期）使用服务端密钥签名后生成一个token返回给客户端。之后每次客户端的每次请求都带上这个token，所有用于用户认证的需要的信息都在这个token（self-contained）中，服务器只需要验证token是否合法即可。

一个合法的token必须满足如下条件：

1. 是由服务端签名过的
2. 用户认证信息是合法的
3. 是在有效期内的

Token-based认证除了使得服务端实现无状态化外，还有另一个优势——**便于扩展（Scalable）**。
正由于认证过程是无状态的，在分布式部署的场景中，任何一台服务器都可以根据用户请求中携带的token进行验证，而不再有session共享的问题。

一张简单对比图：
![session-based vs token-based authentication][1]

## JSON Web Token
在上面这张图中的Token-based认证，我们看到用户认证成功之后，服务器返回了一个token，下一次请求的时候呢，http带上了这样一个首部`Authorization Bearer ....`，这种实现方式呢，是现在Token-based认证中最常用的一种实现——JSON Web Token，简称JWT。

JWT本身非常灵活，理论上，你可以在里面放任何信息，只要它满足合法的JSON格式。关于JWT更多信息，这里不做详细说明，请移步官方文档[JWT Introduction][2]

另外，[Cookies vs. Tokens: The Definitive Guide][3]这篇文章里有一个使用JWT的例子，可以参考一下～

之前我也有用Java写过一个简单的认证服务，用的是[jsonwebtoken.io][4]这个库。

比如说这样一个POST请求：
![jwt auth request][5]
验证成功后，后台会返回一个token，然后使用这个token就可以调用被保护的API并通过认证了，像下面这样：
![Screen Shot 2018-10-23 at 10.25.41 PM.png-112kB][6]

---

OK，我们就这样简单介绍了Token-based认证，概念还是很好理解的，之后我们在介绍更为复杂的单点登录或者第三方登录时，还会经常见到Token的身影的～

未完待续~

---

**参考资料**

* [3种web会话管理的方式][7] - 这篇文章对于认证方式分得更细，还比较全面和详细。
* [Session Authentication vs Token Authentication][8]
* [The Most Common Authentication Methods in Web Application Development.][9]


  [1]: http://static.zybuluo.com/JaneL/kr3x579373jec10t3zsh8686/session-based%20vs%20token-based%20authentication.png
  [2]: https://jwt.io/introduction/
  [3]: https://dzone.com/articles/cookies-vs-tokens-the-definitive-guide
  [4]: https://java.jsonwebtoken.io/
  [5]: http://static.zybuluo.com/JaneL/ost2sj5pougv8wj8nbcxzm85/Screen%20Shot%202018-10-23%20at%2010.23.14%20PM.png
  [6]: http://static.zybuluo.com/JaneL/u4gmyz7g7knii5e2r5a3pj98/Screen%20Shot%202018-10-23%20at%2010.25.41%20PM.png
  [7]: https://www.cnblogs.com/lyzg/p/6067766.html
  [8]: https://security.stackexchange.com/questions/81756/session-authentication-vs-token-authentication
  [9]: https://medium.com/@pavithraranathunga/the-most-common-authentication-methods-in-web-application-development-35845ecb1da0