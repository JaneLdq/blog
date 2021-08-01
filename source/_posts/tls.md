---
title: HTTPS 究竟是如何保障安全通信的？
date: 2020-04-30 09:49:51
categories: 技术笔记
tags:
- 网络安全
- HTTPS
- TLS
---

我们都知道 HTTP 协议本身是不提供任何安全保障的，HTTP 的报文只是普通的文本，在传输中报文的内容是完全公开的。在没有安全机制的保障下，基于 HTTP 的通信很容易受到诸如窃听、man-in-the-middle 这类攻击。HTTPS (Hypertext Transport Protocol Secure) 作为 HTTP 的扩展，由[RFC 2818](https://tools.ietf.org/html/rfc2818) 正式定义，也被称为 HTTP over TLS，它在 HTTP 与 TCP 之间引入了 TLS (Transport Layer Security)，其目的就是为 HTTP 协议提供安全服务，防御上述攻击。

那么，TLS 究竟是什么？它是如何为 HTTP 通信提供安全保障的呢？在接下来的内容中我们将逐步揭开 TLS 的神秘面纱，最终你会发现，原来都是些“熟面孔”。

<!--more-->

---
# HTTP, TLS 与 TCP
TLS (Transport Layer Security) 实现在 TCP 协议之上，也常被看作是 TCP 协议的增强 (enhancement)，为应用层协议提供安全服务。TLS 的前身是 SSL (Secure Sockets Layer)，所以有很多文章提到 HTTPS 的 "S" 也指 SSL。TLS 实际上就是 SSL 的新版本。

当我们使用 HTTP 协议时，HTTP 请求在 TCP 连接上传输，而是用 HTTPS 协议时，HTTP 请求在 TLS 连接上传输，TLS 连接建立在 TCP 连接之上。三者的关系如下图所示：

![HTTP & TLS & RCP][1]

当 TCP 连接建立之后，TLS 将通过一系列认证、协商流程建立起 TLS 连接。当发送方的 HTTP 请求传给 TLS 后，TLS 对 HTTP 请求加密并传递给下层的 TCP 连接，接收方的 TLS 将 TCP 连接传上来的数据解密得到原始的 HTTP 请求交给上层的 HTTP 处理。
在这个过程中，TLS 连接的建立意味着通信的双方通过了身份认证，而传输的内容则由 TLS 提供的一系列加密验证机制保证了数据的保密性和完整性。

具体的实现待下文一一道来。

---
# TLS 1.2
TLS 从 SSL 发展而来，历经 TLS 1.0, 1.1, 1.2，最新的 TLS 版本是由 2018 年 [RFC 8446](https://tools.ietf.org/html/rfc8446) 更新的 TLS 1.3。虽然有很多服务器开始往 1.3 迁移，但目前使用最多的还是 TLS 1.2，所以我们先从 TLS 1.2 介绍。

TLS 由两个子协议组成：
* **TLS 记录协议 (TLS Record Protocol)** - 定义 TLS 消息的字段（包括长度、描述、内容），负责原始数据的切分、压缩、MAC 计算、加密和传输，并对接收到的数据执行反向操作，即解密、验证、解压和组装。TLS 记录协议提供了实现安全通信所需的两大要素：
    * **保密性**：TLS 连接中传输的消息都会采用对称密钥加密
    * **完整性**：TLS 将使用 MAC 验证数据的完整性（*还记得 MAC - Message Authentication Code 吗～*）
* **TLS 握手协议 (Handshake Protocol)** - 建立在 TLS 记录之上，用于在实际的数据传输开始前，完成通信双方的身份认证、加密算法的协商和密钥交换。通过 TLS 握手协议我们获得了 TLS 记录协议需要的安全参数（加密算法和密钥），同时实现了安全通信的第三个要素的**身份认证**。

---
## TLS 1.2 握手过程
TLS 握手协议承担了创建 TLS 会话的重任，成功握手意味着通信双方信任的建立，以及对通信信道安全的保障。这究竟是如何做到的呢？
下图展示了 TLS 握手的过程，详细解释见下文：

![TLS 1.2 Handshake][2]

1. 客户端向服务器发送 `ClientHello`，请求发起 TLS 会话
    * 在 `ClientHello` 中包含了客户端使用的协议版本 (protocol version)、会话 ID(session id)、客户端支持的加密套件列表（cipher suites)、压缩方法列表 (compression methods) 和一个随机数（random, 实际上是个结构体，包含了一个随机数和时间戳），还可能包含一组扩展 (extensions)。
2. 服务器向客户端发送一组消息，依次为：
    * `ServerHello`，其中包括了服务器的协议版本、会话 ID、加密套件（从客户端提供的加密套件列表中选择出来的）、压缩方法（从客户端提供的列表中选择出来的）和服务器随机数，以及一组扩展。
    * `Certificate`，包括了服务器的证书链，默认证书格式为 X.509，用于客户端对服务器的身份验证。
    * `ServerKeyExchange`，可选项，与协商选定的密钥交换方法有关。只有在 `Certificate` 消息中包含的信息不足以用于后续 premaster secret 的交换时才需要发送这条消息，其内容也与密钥交换方式相关。
    * `CertificateRequest`，可选项，只有在服务器需要对客户端进行身份验证时才发送。
    * `ServerHelloDone`，表示服务器 Hello 消息的结束
3. 当客户端收到服务器发送的 `ServerHelloDone` 消息后，会响应一组消息，依次为：
    * `ClientCertificate`，可选项，如果服务器发送了 `CertificateRequest`，那么客户端会发送此条消息，将客户端的证书链发送给服务器用于身份认证
    * `ClientKeyExchange`，这一步是交换密钥的关键，这一步之后，服务器和客户端都会获得相同的 **premaster secret**。然后双方将这个 **premaster secret** 和之前在`ClientHello` 和 `ServerHello` 中交换的两个随机数作为**伪随机数函数 PRF** (pseudorandom function) 的输入参数生成 **master secret**。这个 master secret 会被划分为两类四个密钥用于往返两个方向的消息的加密和校验，分别为 `client write MAC key`, `server write MAC key`, `client write encryption key` 和 `server write encryption key`。
        * 此条消息的内容跟选定的公开密钥算法相关。举个例子，如果是RSA算法，则内容是经过公钥加密的 premaster secret；如果是DH算法，则是用于生成 premaster secret 的信息。
    * `CertificateVerify`，可选项。？
    * `ChangeCipherSpec`，客户端通过发送这条消息告知服务器客户端这边已经准备好使用之前协商好的加密套件加密数据并传输了，与此同时，客户端会将上述协商好的 `Cipher Spec` 从 **pending** Cipher Spec 拷贝到 **current** Cipher Spec 做好准备。
    * `Finished`，紧随 `ChangeCipherSpec` 之后的 `Finished` 消息是第一条使用协商的加密套件保护的消息.
4. 服务器收到客户端的 `ChangeCipherSpec` 消息后，也会准备好使用协商好的加密套件，并发送一条服务端的 `ChangeCipherSpec` 的消息给客户端，并使用新的加密套件发送服务端的 `Finished` 消息。当服务器收到客户端的 `Finished` 消息后，使用对称密钥解密消息，验证 MAC。
5. 客户端收到服务器的 `Finished` 消息后，也要使用对称密钥解密消息，验证 MAC，如果一切顺利，则握手完成，可以开始发送应用数据。

握手完成后，后续通信的安全性实际上依赖于 premaster secret 的安全性。通过分析上述握手过程，我们可以看到用于生成 master secret 的三个随机数：客户端随机数、服务器随机数和 premaster secret 中虽然前两个存在被窃听的可能，但 premaster secret 是通信双方完成身份验证（建立信任）后加密传输的，理论上这个随机数是不可能被窃听的。因此，只要 premaster secret 不被破解，master secret 便是安全的，整个通信也是安全的。

---
## Change Cipher Spec 协议
上面提到的 `ChangeCipherSpec` 消息实际上是由一个独立的 Change Cipher Spec 协议定义的，它的数据包中只有一个字节的数据，其值为 1。它的作用就是用于告知对方自己已经切换到之前协商好的加密套件（Cipher Suite）的状态，准备使用之前协商好的加密套件加密数据并传输了。

---

了解了 TLS 握手的大致流程，不难发现，其中应用了大部分我们在《浅谈网络安全》笔记中聊到的很多技术，因此开头我便说，TLS 中包含了诸多“熟面孔”：
* 服务器与客户端的身份认证利用**数字证书**实现
* 共享密钥的采用**非对称密钥加密**
* 应用数据的传输采用**对称密钥加密**
* 数据的完整性采用 **MAC** 验证

通过综合使用以上技术，TLS 为上层应用协议提供具备数据**保密性**、**完整性**和**身份认证**的安全通信服务。

---

**参考资料**
* [RFC 5246](https://tools.ietf.org/html/rfc5246)
* [HTTPS - wiki](https://en.wikipedia.org/wiki/HTTPS)
* [Transport Layer Security - wiki](https://en.wikipedia.org/wiki/Transport_Layer_Security)

[1]:/uploads/images/http-tls-tcp.svg
[2]:/uploads/images/tls-2-handshake.svg