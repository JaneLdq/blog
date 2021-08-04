---
title: 浅谈 OAuth 2.0 (一) - 基本概念
date: 2019-10-21 14:04:34
categories: 技术笔记
tags: 
- OAuth
- 网络安全
- 认证与授权
---

一年之前开过一个关于Web认证相关的坑，只简单地开了个头就不甚了了。时隔一年，正好这一年做的项目一直都跟应用间的认证和授权相关，于是决定重拾旧坑，好好研究一番。

（ˊ_>ˋ 原本就想写个简单的概述，但那样好像没什么意义，而且内容也实在是有点多...）

第一篇就先聊聊 OAuth 诞生的背景和其中的一些基本概念吧。

---
## 为什么会有 OAuth 2.0 的诞生
在传统的客户端-服务器认证模型中，由**客户端**携带**用户**的**认证凭证（credentials）**来访问**服务端**受保护的**资源**，而当有第三方应用想访问服务端资源时，我们最容易想到的方式是什么呢？

让第三方应用与客户端**共享认证凭证**。

而上述这种做法的弊端和局限性也很明显：
* 第三方应用需要自己保存用户凭证（安全隐患非常大，尤其在使用明码的情况下）
* 服务端要支持用户名密码验证机制，而密码本身就存在安全弱点
* 第三方应用获得的访问权限过大，而客户端没有办法限制授权的有效期和范围
* 用户如果要收回对某一第三方应用的授权，就只能修改密码，而这将同时收回所有其他第三方应用的授权
* 只要有一个第三方应用被攻击了，那么用户的认证凭证就泄漏了，那么所有被这个密码保护的数据也将被泄漏。

OAuth 的出现就是为了解决这些问题。OAuth 并不是框架或工具，而是一组基于 HTTP 协议的标准，它定义了一系列与授权场景相关的概念和规则，针对不同的应用场景给定了几类标准的授权过程。下面就让我们一起揭开 OAuth 神秘的面纱吧。

*注：本文将只介绍 OAuth 2.0，OAuth 2.0 和 OAuth 1.0 并不兼容。*

<!--more-->
---
## OAuth 2.0 中的角色
就好比在面向对象的编程中，我们要先抽象出对象，在应用授权的场景中，我们也要先划分出不同的角色。
OAuth 2.0 定义了如下四种角色：

* **Resource Owner** - 资源拥有者，拥有给其他资源请求方授权的能力的角色，通常指用户（下文都简称用户）。
* **Resource Server** - 资源服务器，存储用户资源的服务器。
* **Client** - 客户端，即上面所提到的第三方应用，在用户授权下以用户身份访问受保护资源的应用。
* **Authorization Server** - 授权服务器，在成功认证资源拥有者并获得授权后向客户端发送 Access Token（客户端通过 Access Token 向资源服务器请求资源）。

资源服务器和授权服务器可以是一台服务器，也可以是不同的服务器。

这四种角色之间的主要交互过程如下图所示：
![OAuth 2.0 Roles][1]

简单概括流程如下：
1. 客户端在请求资源前向用户请求 `Authorization Grant`。
2. 客户端拿到用户给的 `Authorization Grant` 后向授权服务器获取用于访问资源的 `Access Token` 授权服务器对客户端进行认证并校验授权，如果合法，则生成 `Access Token` 并返回给客户端。
3. 客户端拿着 `Access Token` 向资源服务器请求受保护的资源，服务器校验 `Access Token`，如果合法，则返回资源。

注：在上图中体现出来是客户端直接向用户请求授权，而更好的方式是由授权服务器作为中介间接授权，详细过程见下文介绍的具体的授权过程。 

上面又提到了两个新的概念，`Authorization Grant` 和 `Access Token`，要讲清楚 `Authorization Grant` 就必须先弄懂 `Access Token`。

---
## Access Token & Refres Token
事实上，OAuth 2.0 中定义了两种 Token，一种是 `Access Token`，另一种是 `Refresh Token`。

### Access Token
`Access Token`，故名思义，就是用来访问受保护资源要用到的令牌。客户端要访问资源服务器上受保护的资源，就必须要有 `Access Token` 作为通行证。`Access Token` 由授权服务器生成。客户端之后获取了用户授权后才能向授权服务器申请 `Access Token`。

OAuth 通过 `Access Token` 在授权过程中增加了一个授权层（Authorization Layer），这样对于资源服务器而言，它只需要能支持 `Access Token` 这一种授权方式就可以了。

*“只要你给我一个合法的 `Access Token`，那我就给你提供你想要的资源，至于这个 `Access Token` 是怎么来的：用户和客户端如何认证、如何授权，我都不 care，就是这么任性”。*

那么问题来了：

🌟 **Access Token 的格式是什么？里面包含了什么内容？**
🌟 **资源服务器如何验证一个 Access Token 是合法的呢？**
🌟 **资源服务器和授权服务器之间的信任如何建立？**

>  An access token is a string representing an authorization issued to the client. ... Tokens represent specific scopes and durations of access, granted by the resource owner, and enforced by the resource server and authorization server. (RFC 6749 #section-1.4 )

从官方定义上来看，`Access Token` 是一个**字符串**，至少要提供关于**客户端的基本信息**（通常是客户端的ID）和该客户端**获得的权限**，权限由一组 `scopes` 表示，并且 `Access Token` 是有有效期的。

至于 Access Token 的具体格式、这些内容究竟是如何获得：是 `Access Token` 自包含的（self-contain）内容呢？还是 `Access Token` 只提供一个标识符，然后利用这个标识符再通过某种方式来获取呢？OAuth 2.0 (RFC6749) 里没有规定。

常见的方式如下：
* **Signed JWT Bearer Token** - 所有需要勇于验证合法性的内容都包含在 token 自身里了，包括但不限于用户信息、客户端信息、scope 等。这种方式要求资源服务器能通过 `Access Token` 的签名（授权服务器进行数字签名）来验证其确实是来自可信任的授权服务器。这种情况可能需要在资源服务器上注册授权服务器的证书。

* **共享数据库** - 对于小型应用，资源服务器和授权服务器通常是同一个服务器，此时它们之间自然不存在信任问题，而且还能直接内部共享 Token 的信息，比如共享数据库。

* **通过 Token Introspection Endpoint** - 这是一个OAuth 2.0 的扩展协议（[RFC7662](https://tools.ietf.org/html/rfc7662)），由授权服务器开放一个用于验证和解析 `Access Token` 的接口供资源管理器调用。

总而言之，OAuth 2.0 没有明确规定资源服务器和授权服务器之间的信任如何建立，这取决于具体应用的安全性需求和开发人员的选择。

---
#### Scopes
我们在前文提到，传统的用户名密码授权方式问题之一就是权限过大，一旦拥有了用户的用户名密码，客户端理论上可以访问所有该用户能访问的资源。然而，这并不是我们想要的。我们希望只给客户端授予必要的最小权限，那么 `scope` 就是 OAuth 中用来达到这一目的存在。

`scope` 由资源服务器来定义，比如最简单的 `sample.read`、`sample.write` 这类，在授权时，用户可以选择只授予某个客户端 `sample.read` 的 `scope` 而给另一个客户端授予 `sample.write` 的权限。

---
### Refresh Token
`Refresh Token` 也是由授权服务器生成的。当一个 `Access Token` 过期或者失效时，客户端可以使用 `Refresh Token` 来获取一个新的 `Access Token`，这个新的 `Access Token` 拥有的 `scope` 范围**小于等于**原来的那个。

`Refresh Token` 是可选项，如果授权服务器生成了 `Refresh Token`，它会与 `Access Token` 一起返回给客户端。

---
## 客户端的分类与注册
我想在介绍 `Authorization Grant` 之前先提一下 OAuth 2.0 中的客户端的分类和客户端注册。

### Public & Confidential 客户端
OAuth 2.0 将客户端分为两类：
* **Confidential** - 指具有自己维护客户端认证凭证（client credentials）安全能力的客户端，比如运行在受访问限制的服务器上的 web 应用，用户只能通过浏览器访问应用提供的HTML用户界面，并不会接触到客户端认证凭证。

* **Public** - 指不具备自己维护客户端认证凭证安全能力的客户端，比如运行在浏览器中的 Javascript 应用或运行在移动设备上的 Native 应用，这些应用的源码都被下载或安装到了浏览器或设备上，认证凭证或协议相关的数据非常容易被获取到，因此是不安全的。

---
### 客户端注册
在整个 OAuth 2.0 授权流程启动之前，客户端需要先向授权服务器注册。“注册”这个操作具体怎么实现，不在 OAuth 2.0 考虑的范围，它只规定了“注册”这个操作需要完成如下任务：
* 指定该客户端的类型，public 或 confidential
* 提供客户端重定向URI （redirect URI，关于这个在后文还会再提到）
* 提供其他授权服务器要求的信息（比如客户端名字、网站、LOGO、描述等）

注册成功后，授权服务器会为客户端生成 `client_id`，如果是 confidential 的客户端，还会生成一组 `client credentials` 用于认证。Client credential 可以是 password，也可以是 public/private key，具体方式取决于授权服务器对安全性的要求。

---
### 客户端认证
OAuth 2.0 没有明确规定客户端的认证方式，这取决于应用的具体实现。
对于采用 password 认证的客户端，可以采用 HTTP Basic 认证的方式，也可以将其作为 POST 请求的 request body 传送给授权服务器。

除此之外，[OpenID Connect](https://openid.net/specs/openid-connect-core-1_0.html) 作为一个基于 OAuth 2.0 的认证协议，其中也定义了多种客户端认证的方式，由于本系列笔记的重点在授权，在这里就不作展开了。

---

好了，到目前为止，我们接收了很多概念，包括：
* OAuth 2.0 中定义的四个角色：**资源拥有者（用户）**、**客户端**、**授权服务器**和**资源服务器**
* 客户端用于请求受保护资源的 **Access Token**，以及用于获取新 Access Token 的 **Refresh Token**
* 用于限定授权范围的 **Scopes**
* 客户端分为两类 - **public** 和 **confidential**
* 我们还知道了**客户端需要提前向授权服务器注册**，授权服务器在生成 Access Token 之前要先进行**客户端认证**。

当脑海中有了这些相对零散的基本概念后，下一步，就是将它们串联在一起，组成 OAuth 2.0 中最核心的部分 —— 四种授权类型。

我们下文见~

---

**参考资料**

* [RFC6749 - The OAuth 2.0 Authorization Framework](https://tools.ietf.org/html/rfc6749)
* [OAuth 2.0 Servers](https://www.oauth.com)
* [What the Heck is OAuth?](https://developer.okta.com/blog/2017/06/21/what-the-heck-is-oauth)
* [stackoverflow - OAuth v2 communication between authentication and resource server](https://stackoverflow.com/questions/6255104/oauth-v2-communication-between-authentication-and-resource-server)
* [stackoverflow - How Resource Server can identify user from token?](https://stackoverflow.com/questions/48770574/how-resource-server-can-identify-user-from-token)

  [1]: /blog/uploads/images/oauth-roles.svg








































