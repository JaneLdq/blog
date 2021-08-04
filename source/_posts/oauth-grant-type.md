---
title: 浅谈 OAuth 2.0 (二) - 授权类型
date: 2019-10-24 10:27:06
categories: 技术笔记
tags: 
- OAuth
- 网络安全
- 认证与授权
---

在上一篇笔记中，我们了解了一些 OAuth 2.0 中的概念，其中最重要的就是 OAuth 2.0 的四个角色：**资源拥有者（用户）**、**客户端**、**授权服务器**和**资源服务器**。在本篇笔记里，我们将介绍 OAuth 2.0 中的授权类型。

针对不同的授权场景，OAuth 2.0 定义了 4 种授权类型：
* **Authorization Code**
* **Implicit**
* **Resource Owner Password Credentials**
* **Client Credentials**

这里的应用场景主要指的是客户端的类型，一方面指 public 或 confidential，另一方面指具体种类，比如web 应用、native 移动应用、不包含浏览器的设备，又或者是客户端也是一个 server。根据不同的情况，客户端获取 Access Token 的过程也不尽相同。

除此之外，OAuth 2.0 还支持自定义扩展授权类型。

下面我们就分别来看看这四种授权类型的详细定义吧。

<!--more-->
---
## Authorization Code
Authorization Code（授权码）这一授权类型基于重定向的过程，因此在整个授权过程中，有多个步骤要求 User Agent（通常指浏览器，这一组件在上图中没有体现）的参与。它既可以用来获取 Access Token，也可以用来获取 Refresh Token。

先看图：
![Authorization Code Grant][1]

1. 客户端通过 User Agent 向授权服务器发起获取授权码的请求，在请求中会带上 `client_id`, `scope`, `state` 和一个在获取授权码后要跳转到的 `redirect_uri`（请记住这个URL，我们还会再见到它好几次）。
2. 授权服务器将 User Agent 重定向到用户认证页面，对用户进行认证，并授权给客户端，然后授权服务器将 User Agent 重定向到刚刚客户端在请求中指定的那个 `redirect_uri`，并附加上授权码和客户端在请求中带上的`state`。（用户也可以在这一步拒绝授权，那么在这一步将返回 Error Response）。
3. 客户端带着`code`，`redirect_uri` 和 `client_id` 向授权服务器请求 Access Token，在这一步，如果客户端之前注册时有生成 Client Credential，那么授权服务器会对客户端进行认证，并且还要校验这个请求中的 `redirect_uri` 和获取授权码的请求中的 `redirect_uri` 是否一致。
4. 如果客户端认证成功，其他校验也都通过了，那么授权服务器就会返回一个 Access Token，有可能还附带一个 Refresh Token。
5. 客户端拿着 Access Token 向资源服务器请求受保护的资源。

在这一授权类型中，**授权服务器充当了客户端和用户之间的媒介，在整个过程中，用户只与授权服务器交互完成认证，因此用户的认证信息不需要与客户端共享。**

---
## Implicit
一种为浏览器客户端应用简化的授权流程。注意，这里的浏览器客户端是指指类似与运行在浏览器中的 Javascript 应用，而非通过浏览器访问、实际部署在远程服务器上的应用。
在 Implicit 这类授权中，客户端使用脚本语言直接请求授权，跳过了“获取授权码”这个步骤，Access Token 将作为 `redirect_uri` 的一部分返回给客户端。

Implicit 授权过程如图所示：
![Impilicit Grant][2]
1. 客户端向授权服务器发送授权请求。
2. 授权服务器重定向到用户认证界面，用户通过认证并同意授权。
3. 授权服务器将生成的 Access Token 放到客户端的 `redirect_uri` 的 fragment 中，并重定向到该 URL。
4. 客户端使用从重定向 URL 中获得的 Access Token 向资源服务器请求资源。

Implicit 授权的过程比授权码类型简单，但简单的流程也暴露了更多的安全隐患：
* 因为在 Implicit 授权中，Access Token 直接暴露在 URL 中，这意味着重定向到包含着 Access Token 的 URL 时，这个携带者 Access Token 的 URL 会被记录到浏览器的浏览历史中，相当于暴露给了浏览器。如果有心人企图利用浏览器历史获取 Access Token，那么就会对受保护的资源造成威胁。因此，一般授权服务器给同构 Implicit 方式获取的 Access Token 的设定的有效期都比较短。

Implicit 的安全性没有授权码类型强，那么为什么 OAuth 2.0 中还要定义这一类型呢？

### 为什么会有 Implicit 类型？
Implicit 类型主要用于授权服务器不支持 [**CORS - 跨域资源共享**](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing) 的情况。因为在授权码授权类型中，客户端需要向授权服务器发送一个POST请求，如果授权服务器不支持 CORS，那么这个请求就会被拒绝，在这种情况下，授权码类型不再适用。

时至今日，在绝大多数服务器都支持 CORS，并且浏览器能自动在 HTTP 请求中添加 CORS 首部的情况下，我们要尽量避免使用 Implicit 授权类型。

📹 **相关视频**
[What's going on with the OAuth 2.0 Implicit flow?](https://www.youtube.com/watch?v=CHzERullHe8)（这个视频对 Implicit 授权的介绍还挺全面的，值得一看）

---
## Resource Owner Password Credentials
Resource Owner Password Credentials（用户密码认证）授权，即客户端直接使用用户的用户名密码进行认证获得 Access Token，这种方式要求客户端能够获取用户的用户名密码。

这种授权类型只应该在高度信任客户端的情况下使用，比如当客户端是操作系统的一部分，或者其他认证类型不可行的时候。

授权过程如图所示：
![Resource Owner Password Credentials Grant][3]
1. 用户向客户端提供用户名和密码。
2. 客户端利用用户的用户密码向授权服务器请求授权。
3. 授权服务器验证通过后返回 Access Token。

即使这种情况用到了用户的用户密码，也应该只是在第一次请求时用于获取 Access Token 和 Refresh Token， 并将 Token 设定为长生命周期的 Token 以便于后续操作，这样可以避免在客户端保存用户密码。

通常，这一类型是用来将使用直接认证的客户端（比如使用 HTTP Basic 或 Digest 认证）迁移到 OAuth 服务器，也就是说原来这些客户端要保存用户名密码，现在转为保存 Token。

---
## Client Credentials
在这一类型中客户端以自己的名义，而不是以用户的名义向资源服务器请求资源，因此在获取 Access Token 时客户端直接采用自己的认证信息进行认证就可以了。这一模式从本质上来讲，客户端自身就是“资源拥有者”，因此并不存在是授权问题。

授权过程如下图所示：
![Client Credentials Grant][4]
1. 客户端向授权服务器进行认证。在上一篇笔记中我们有提到 OAuth 没有规定要如何认证客户端，关于客户端的认证见另一个基于 OAuth 发展的协议 OpenID Connect。
2. 客户端认证通过后，授权服务器向客户端发送 Access Token。

---
## 协议端点（Endpoints）
通过分析以上四种授权过程，主要用到了如下端点（Endpoints）：

两个授权服务器端点：
* **Authorization endpoin** - 用于从用户那儿获取授权。
* **Token endpoint** - 客户端使用此端点来获取 Access Token。所有拥有 credentials 的客户端在调用此端点时都要先经过授权服务器的认证。

一个客户端端点：
* **Redirection endpoint** - 获取用户的授权后，授权服务器通过此端点将授权返回给客户端。这个重定向端点一般在客户端注册时提供给授权服务器，也可以在发送授权请求时使用 `redirect_uri` 参数指明。

---

到目前为止，关于 OAuth 2.0 基本的理论知识已经说了很多，虽然在那些 RFC 文档里还有很多细节和扩展内容，但作为「浅析」系列，我想我们还是先把这些抽象的概念转换为能跑起来的简单例子，实际感受一下 OAuth 2.0 中这几个角色的交互过程会比较有意思。

So，我计划用 Spring Security OAuth 来实现一个 OAuth 2.0 授权场景。
下文见～

**参考资料**
* [RFC6749 - The OAuth 2.0 Authorization Framework](https://tools.ietf.org/html/rfc6749)
* [Is the OAuth 2.0 Implicit Flow Dead?](https://developer.okta.com/blog/2019/05/01/is-the-oauth-implicit-flow-dead)

  [1]:/blog/uploads/images/oauth2-code-grant.png
  [2]:/blog/uploads/images/oauth2-implicit-grant.png
  [3]:/blog/uploads/images/oauth2-resource-owner-credentials-grant.png
  [4]:/blog/uploads/images/oauth2-client-credentials-grant.png