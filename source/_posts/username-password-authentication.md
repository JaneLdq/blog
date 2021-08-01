---
title: Web认证那些事儿（二）用户名密码认证
date: 2018-10-18 20:58:43
categories: 技术笔记
tags: 
- 网络安全
- 认证与授权
---

我们就从用户名密码认证说起。

认证过程很简单：当客户端访问受保护的API时，需要提供用户名和密码，服务端验证通过后返回请求的资源，否则拒绝访问。

HTTP协议本身就提供了两种基本认证方式。
这两种认证方式的基本流程是一个质询过程：

1. 客户端发送一个HTTP请求给服务器
2. 服务器并不会立即响应请求，而是“认证质询”响应，要求对方提供认证信息
3. 客户端再次发起请求并附上认证信息（比如用户名密码）
4. 如果认证成功，则服务器正常响应，否则再次质询或返回错误

<!--more-->
就像下面这样：
![HTTP Authentication][1]
下文介绍的基本认证和摘要认证的差别主要发生在(2)(3)步。

# HTTP基本认证 - HTTP Basic Authentication
Basic认证遵循如上质询流程，当服务器返回401响应后，客户端的请求需要带上用户认证信息：将用户名和密码以`:`接起来，然后用Base64编码得到一串字符。然后在HTTP请求中添加一个`Authorization`首部，格式为`Basic YourEncodededBase64String`。

举个例子，假设用户名为test，密码为123456，`/auth`API使用HTTP Basic认证，那么我们访问这个API的HTTP请求如下：
```http
GET /auth HTTP/1.1
Authorization: Basic dGVzdDoxMjM0NTY=
Host: localhost:8080
```
上面的`dGVzdDoxMjM0NTY=`这串字符就等于`test:123456`使用Base64编码的结果。

* HTTP基本认证的安全问题
    * Base64实际上并没有对用户名密码做任何加密操作，实际数据等同于“明文”
    * 即使密码是以更难解码的方式加密的，第三方用户仍然可以捕获被修改过的用户名和密码，并且重复使用这个密码进行重放攻击
    * 如果用户大量使用相同密码，很容易被交叉盗用
    * 没有提供任何针对代理和中间节点如man-in-the-middle攻击的防护措施，它们可能在没有修改认证首部的前提下篡改报文的其他部分

因此，如果选择使用基本认证，一般都会与加密数据传输（HTTPS）配合使用。

---
# HTTP摘要认证 - HTTP Digest Authentication
HTTP摘要认证也遵循质询过程，当比基本认证更为安全。不同于基本认证直接传输密码，摘要认证的宗旨是“绝不传输密码”。
不传输密码那传输什么？“摘要”咯。
何为摘要？摘要就是对用户名密码和一些其他信息进行MD5转换后得到的值，就好比把食物吃进肚子里经过一番消化得到的产物。

虽然不传输密码，但是对于黑客而言，只要截取了密码“摘要”根本不用管密码是什么，直接拿来用就好了。为了避免这样的情况发生，在摘要认证中，服务器在返回401质询响应时，除了会带上realm值（这个值是什么呢请自行查询）外，还会带上一个特殊的值——**nonce**。客户端在生成摘要时带上nonce，这个值会经常发生变化（可能是每毫秒，或者是每次认证都变化），以此避免重放攻击。

> The word nonce means "the present occasion" or "the time being." In a computer-security sense, the nonce captures a particular point in time and figures that into the security calculations.

简单放一个客户端应答质询的请求例子：
```http
GET /dir/index.html HTTP/1.0
Host: localhost
Authorization: Digest username="Mufasa",
                     realm="testrealm@host.com",
                     nonce="dcd98b7102dd2f0e8b11d0f600bfb0c093",
                     uri="/dir/index.html",
                     qop=auth,
                     nc=00000001,
                     cnonce="0a4f113b",
                     response="6629fae49393a05397450978507c4ef1",
                     opaque="5ccc069c403ebaf9f0171e9517f40e41"
```

在这个请求里，可以看到`Authorization`首部后跟的“Digest ...”，然后接了参数，这些参数的具体含义这里就不过多介绍了，关于更完整的例子，可以戳这里[Digest Authentication Example With Explanation][2]

虽然摘要认证比基本认证稍微安全一点，但也只是一点点而已，缺陷还是很多的，比如：

* 面对Man-in-the-Middle攻击十分脆弱。比如攻击者可以中途质询响应，要求客户端使用基本认证，从而窃取密码。
* HTTP的多重认证机制，由于没有要求客户端选择最强的认证机制，所以很难保证认证效果。
* 使用摘要认证的客户端会用服务器提供的nonce来生成响应。但如果中间有一个被入侵的或恶意的代理在拦截流量（或者有个恶意的原始服务器），就可以很容易地为客户端的响应计算提供nonce。在这种情况下，使用已知密钥（入侵者提供的nonce）来计算响应可以简化密码分析过程。

---
# Form-based认证
虽然HTTP协议中定义了两种标准的认证方式，但是由于其局限性和不安全性，在日常开发中，最常见的是被称之为Form-based的认证方式，它并没有一个规范的标准定义，所以也没有标准实现。因为其主要实现方式是基于HTML的表单，因此被通称为Form-based认证。

下面就是一个使用表单实现的非常简单的HTML登录页面了：
```html
<!doctype html>
<body>
    <form method="post" action="/login">
        <label>Username</label>
        <input type="text" name="username" required/>
        <label>Password</label>
        <input type="password" name="password" required/>
        <input type="submit" value="Login"/>
    </form>
</body>
```

Form-based认证的基本流程如下：
1. 用户请求访问某个受保护的资源
2. 服务器发现用户尚未认证，返回一个包含需填写认证信息的表单的HTML页面
3. 用户填写用户名和密码并提交给服务器
4. 服务器验证用户信息，如若成功则为当前用户创建一个“Session”，并将session id放到cookie中，之后这个用户的所有请求都会带上这个cookie，这样服务器就可以拿到cookie中的session id将其与服务器端保存在内存里的session id比较进行认证。
5. 当用户退出登录时，Session被销毁，服务器端保存的session id被移除。之后的请求需要重新进行验证。

使用HTTP的Form-based认证中所有的认证信息也都是明文传输的，在这一点上Form-based的安全性并不比Basic认证高，仍然需要与加密传输（比如HTTPS）配合使用。Form-based认证的优势主要在于认证成功后服务器端维护了与用户之间的会话信息，无论是用户主动退出登录还是一定时间后服务器可认为会话超时，在“登出”操作上Form-based认证会比HTTP Basic认证处理得更安全。

说到底，Form-based只是表明这种认证方式使用了HTML的表单作为其实现中的一环，至于整个认证过程的安全性是与整个认证机制的实现息息相关的，比如是否使用HTTPS传输，服务器如何验证，Session的管理等等。

到此为止，我们介绍了三种使用用户名密码进行认证的方式，其中两种是HTTP协议定义的标准认证方式——HTTP基本认证和HTTP摘要认证，第三种是非标准但更加常见的Form-based认证。

---
在Form-based认证流程中，我们提到认证成功后，服务器会通过Session来维持会话的状态。这一类用户认证又被称之为**基于Session的认证**。

# 基于Session的认证

Session的信息一般是存储在服务器的内存数据库中的，这种管理方式在可扩展性或者说分布式环境中具有一定的局限性。一个需要处理大量用户请求的服务会有很多服务器同时运转，如果一个用户的第一个请求发给了服务器A，并且通过了服务器A的认证，那么当前这个session就被保存在服务器A的内存中；过了几分钟，用户发送了第二个请求，由于有负载均衡管理等机制，这个请求可能被分给了另一台服务器B，而服务器B中并没有用户的认证信息，这是用户又会被要求重新登录。这种操作显然很不科学。

另一方面，现在微服务应用广泛，很多服务都以REST API的形式开放出来，REST请求都是无状态的（Stateless），如果用session实现没什么意义，用户每调用一个API都会被要求认证，非常不友好。

那么，针对这些应用场景，有一种更合适的认证方式，那就是——**基于Token的认证**。不过本片笔记就到这里收尾啦，下一篇我们再详细介绍基于Token的认证方式。

未完待续~

---

本期日常：为什么病娇不是生病了之后会变得特别娇气的意思呢？嘤。

---
**参考资料**

* 《HTTP权威指南》
* [Digest Access Authentication - Wikipedia][3]
* [Session vs Token Based Authentication][4]


  [1]: http://static.zybuluo.com/JaneL/o8n479smuovgl2sak11xr2uk/HTTP%20Authentication.png
  [2]: https://en.wikipedia.org/wiki/Digest_access_authentication#Example_with_explanation
  [3]: https://en.wikipedia.org/wiki/Digest_access_authentication
  [4]: https://medium.com/@sherryhsu/session-vs-token-based-authentication-11a6c5ac45e4