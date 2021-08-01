---
title: 浅谈网络安全（三）- 密钥的管理与分发
date: 2020-04-18 21:33:20
categories: 技术笔记
tags:
- 网络安全
- 密钥分发
- 数字证书
---

我们可以说网络通信的安全性几乎是建立在密钥的安全性上，因此如何管理密钥和分发密钥是一个非常重要的话题。下文中我们将一一介绍如下概念：

* **密钥分发中心 (KDC, Key Distribution Center)**
* **数字证书 (Certificate)**
* **认证中心 (CA, Certification Authority)**
* **公钥基础设施 (PKI, Public Key Infrastructure)**

<!--more-->
---
# 密钥分发中心
为了安全地分发密钥，我们可以设立一个大家都信任的**密钥分发中心 (KDC, Key Distribution Center)**，它负责 7*24 为用户提供密钥分发服务。 

## 对称密钥分发
在应用对称密钥加密的场景中，KDC 通常要负责为通信双方生成一个临时的会话密钥，大致流程如下图所示：
![KDC][1]
1. Alice 和 Bob 事先在 KDC 中注册了自己的**主密钥 (master key)**，主密钥用于加密与 KDC 的通信
2. 当 Alice 想与 Bob 通信时，她将自己和 Bob 的标识发送给 KDC
3. KDC 生成一个临时的会话密钥 K<sub>ab</sub>，并将其返回给 Alice，同时这个响应中还包括一个 **Titket**，这个 **Ticket** 包含了 会话密钥 K<sub>ab</sub> 和 Alice 和 Bob 的标识，并**用 Bob 的主密码 K<sub>b</sub>加密**，整个响应用 Alice 的主密码 K<sub>a</sub>加密。
4. Alice 解出收到的响应，获得会话密钥 K<sub>ab</sub>，同时将 Ticket 转发给 Bob，由于 Alice 并不知道 Bob 的主密码，所以 Alice 并不知道 Ticket 的内容
5. Bob 收到 Ticket 并用自己的主密码 K<sub>b</sub> 解开，知道是 Alice 想与其通信，同时也获得了会话密钥 K<sub>ab</sub>

## Kerberos
[Kerberos](https://en.wikipedia.org/wiki/Kerberos_(protocol%29) 是目前最主流的对称密钥分配协议，除了分配密钥之外，Kerberos 也是一个认证协议。在这里就先不详细介绍了。

---
# 数字证书与认证中心
对于非对称密钥分发，有比 KDC 更适合的方案，首先非对称密钥通常不需要临时生成，并不要求 KDC 时刻在线，我们需要的只是某种机制能证明某个公钥确实属于它所生声明的某个实体（这个实体可以是个人、公司或其他组织）。
事实上，确实有一个专门的组织负责证明公钥所属权，它就是**认证中心 (Certification Authority)**。

认证中心如何工作呢？
Alice 可以带着她的公钥去找 CA 请求认证，同时 Alice 要向 CA 提供一些身份信息（比如身份证）用于认证。如果通过了认证，那么 CA 会给她颁发一个 **证书**，证书主要包含如下信息：

* Alice 的名字
* Alice 的公钥
* CA 的名称
* 证书本身的数字签名（CA 用自己的私钥对证书的哈希值签名）
* 签名用到的哈希算法

证书的基本目标就是将一个实体的名字与一个公钥绑定，证书本身是公开的。Alice 可以将她获得的证书放到自己的网站，当 Bob 想与 Alice 建立安全通信时，就可以从 Alice 的网站获取她的证书，并从中得到 Alice 的公钥。这时，如果有个第三者 X 将 Alice 的证书换成了自己的，那么 Bob 可以很快发现证书上持有者的名字并不是 Alice。如果这个 X 只替换了公钥，而保留其他信息，Bob 仍然可以通过计算证书的哈希值，并将其与使用 CA 的公钥解密签名获得的值进行比对发现问题（X 没有 CA 的私钥，无法伪造签名）。

举个实例，下图是维基百科网站的证书，可以看到这个证书是由 `DigiCert SHA2 High Assureance Server CA` 颁发的，从上自下依次是持有者的信息、颁发者的信息、持有者公钥和数字签名的详细信息：
![certificate of *.wikipedia.org][2]

## X.509 证书
如果每个实体申请的证书格式都不一样，那么管理不同格式的证书会成为非常痛苦的事情，因此，人们设计了一个统一的证书格式标准，也就是 **X.509**。
因此，当我们提起 **X.509 证书** 时，即表示这个证书满足 X.509 格式标准。(想当初，我还以为X.509是什么很神秘的东西呢，毕竟又是X又是奇怪的数字)

更多关于 X.509 的信息，见 [X.509 - Wiki](https://en.wikipedia.org/wiki/X.509)

---
# 公钥基础设施
CA 负责颁发证书，但不可能全球用户的证书都由一个 CA 来颁发。显而易见，我们需要多个 CA，但多个 CA 怎么组织呢？由谁负责呢，事关全球范围，谁独揽大权都不合适。为了解决这些问题，**公钥基础设施 (Public Key Infrastructure)** 应运而生。

> A public key infrastructure (PKI) is a set of roles, policies, hardware, software and procedures needed to create, manage, distribute, use, store and revoke digital certificates and manage public-key encryption. (from wikipedia)

一种简单的 PKI 形式是 CA 层级结构，最顶层的 CA，又被称为 Root CA，它的责任是证明第二级的 CA，第二级的 CA 下面可能还有第三级的 CA，那么二级 CA 则负责证明三级 CA，三级 CA 才是最终负责为用户颁发证书的 CA。

![Intermediate CA][3]
![Root CA][4]

上图维基百科的证书为例，可以看到维基百科的证书是由 `DigiCert SHA2 High Assurance Server CA` 颁发，它的类型属于 **Intermediate certificate authority**，而这个 CA 的证书又是由 `DigiCert High Assurance EV Root CA` 颁发，它的类型则是 **Root certificate authority**，仔细看它的颁发者也是它自己，说明这是一个自签的 CA (self-signed CA)。

当浏览器想验证这个网站的证书时，它会从网站自身的证书开始逐级向上验证，向这样自底向上回溯到根的证书链常被称为**信任链 (chain of trust)**。

正如我们之前提到的，谁来独揽运行根 CA 都不合适，因此现实世界中，全球范围内有多个根 CA，每个根 CA 有自己的子 CA。目前大约有 100 多个根 CA 的证书是被预先安装到浏览器中的，它们也被称为**信任锚 (trust anchor)**。

对于 PKI 我的理解是它实际上是一个比较泛的概念，到目前为止也没有一个绝对的标准，我只找到了了一个链接 [PKI Technical Standards](http://www.oasis-pki.org/resources/techstandards/)，在这个链接下罗列了与 PKI 相关的诸多标准，将来有缘再见啦～

---

终于写到了证书，CA 这些概念，回想起来，这部分概念在读书的时候确实没怎么在意过，不甚理解。然而这两年在工作中却时常要打交道，毕竟生产环境不像自己搭个玩具，不同系统之间为了能够安全的通信，系统之间的信任必须先建立起来。
前阵子为了做一个 PoC, 当时为了建立某个本地应用与远程系统之间的信任，先是创建了一个自签的CA，然后用这个 CA 签发了本地应用的证书，为了让远程系统信任本地应用，又将本地自签的 CA 的证书导入了远程系统。一顿操作下来，没少与这些概念打交道，倒是正好学习了一波。

只是这波整理下来，又是一个大坑，只能先把坑都留在这里了。

---

**参考资料**
* 计算机网络，第五版
* Computer Networking: A Top-Down Approach, 7th Edition
* [Public key infrastucture](https://en.wikipedia.org/wiki/Public_key_infrastructure)


[1]:/uploads/images/kdc.svg
[2]:/uploads/images/wiki-cert.png
[3]:/uploads/images/intermediate-ca.png
[4]:/uploads/images/root-ca.png