---
title: DNS 两三事
date: 2020-03-01 10:40:30
categories: 技术笔记
tags: 
- 网络
- DNS
---

喜迎 2020 第一篇笔记！突然觉得没有赶在四年一遇的 2 月 29 日写，有点遗憾呢。

最近这几个月摸了不少鱼，也挖了一点点坑，总想着把一个主题全部过一遍再回顾写笔记，结果就是永远都没有开始，因为不会的实在是太多了，怎么都学不完呀，所以，还是边看边整理吧～

---

当初在搭这个博客时，少不了要为域名配置 DNS，按着教程在 DNSPod 上注册了个账号，配了几条记录。至于为什么要做这些，倒真是一直没有深究过。
最近在重新学习网络相关的内容，就拿 **DNS - Domian Name System** 开头吧。

## 为什么需要 DNS
理论上，任何程序通过 IP 地址就可以访问网络资源，然而这类地址实在是不便于人类记忆。因此，机智的人类引入了可读性更好、更易于记忆的**主机名 (hostname)** 。主机名不仅解决了可读性的问题，它将机器名字与机器地址分离，这样即使更换了机器地址发生了变化，也不会影响用户通过主机名访问资源。

由于网络本身只认识 IP 地址，所以就需要某种机制**将主机名映射为 IP 地址**，由此 DNS 应运而生。

<!--more-->
## 什么是 DNS
DNS (Domain Name System) 域名系统，本质上 DNS 定义了一种层次化的、基于域的命名方案，并采用一个分布式数据库系统加以实现。
其主要用途就是将**主机名映射成 IP 地址**。

DNS主要由三大部分组成：
* **The Domain Name Space and Resource Records - 域名空间和资源记录**
* **Name server - 域名服务器**
* **Resolver - 解析器**

---
## 域名空间与资源记录

### 域名空间
DNS 域名空间可以表示为一棵树，从根节点开始，依次是顶级域名、二级域名、三级域名依次类推。树的叶子代表没有子域的域，一个叶节点域可能包含一个主机，也可能包含多个主机。

![domain-name-space][1]

**顶级域名**由ICANN（Internet Corporation fro Assigned Name and Numbers，Internet名字与数字地址分配机构)专门管理。
每个域可以自己控制如何分配它下面的子域。为了创建一个新域，创建者必须得到包含该新域的上级域的许可。

---
### 域名资源记录
每个域都有一组与它相关的资源记录(Resource Record, RR)，这些记录组成了DNS数据库。DNS 的基本功能就是将域名映射为资源记录。

每条资源记录都是一个五元组`(Domain_Name, TTL, Class, Type, Value)`
* **DomainName** - 域名，指出这条记录适用于哪个域，通常每个域有多条记录
* **TTL** (Time-to-live) - 生存期，指明这条记录的稳定程度，与缓存机制相关
* **Class** - 类别，对于Internet信息，它总是IN
* **Type** - 类型，DNS有很多类型
* **Value** - 值，类型决定了值的内容

一台主机最常见的资源记录即它的 IP 地址，除此之外还有其他类型的资源记录。

#### 常见资源记录类型
|类型 | 示例<br>(Domain name, TTL, Class, Type, Value) | 值的含义 |
|---|---|---|
|A | (liaodanqi.me., 600, IN, A, `185.199.111.153`)| 该域名对应的主机 IPv4 地址 |
|AAAA | (google.com., 300, IN, AAAA, `2404:6800:4003:c03::65`)| 该域名对应的主机 IPv6 地址 | 
|NS | (liaodanqi.me., 86400, IN, NS, `f1g1ns1.dnspod.net.`) | 该域名及其子域对应的域名服务器 |
|CNAME | (news.sina.com.cn., 60, IN, CNAME, `spool.grid.sinaedge.com.`) | 该别名对应的规范名（canonical name) |
|MX | (sina.com., 60, IN, MX, `10 freemx2.sinamail.sina.com.cn.`) | 用于接受该特定域名的电子邮件的主机名 | 

除此之外，随着 DNS 协议的扩充和完善，还有很多其他类型的 DNS 资源记录，完整列表可见 [List of DNS record types](https://en.wikipedia.org/wiki/List_of_DNS_record_types)

---
## 域名服务器
在上一小节我们提到 DNS的域名空间是一棵层级结构的树，为了更好的提供服务，整个命名空间被划分为多个不重叠的区域 (Zones) ，每个区域有一个或多个域名服务器关联，这些服务器是持有该区域数据库的主机。通常情况下，通常一个区域有一个**主域名服务器**和多个**辅域名服务器**：
* 主服务器：从自己的磁盘文件读入有关域名信息
* 辅服务器：从主服务器获取域名信息

按照服务器所覆盖的区域，可将其分为三类：
* **Root Servers 根域名服务器** - 负责返回顶级域 (TLD) 的权威域名服务器的相关信息。
    * 为了能访问到根域名服务器，每个域名服务器必须有一个或多个根域名服务器的信息，这个信息通常放在某个配置文件中（包含了根服务器的 NS 和 A 记录），在 DNS 服务器启动时加载到 DNS 缓存。
    * 根域名服务器一共有13个，命名相当随意，从 `a.root-servers.net` 到 `m.root-servers.net`。值得强调的一点是，这里的“13个”是指逻辑上的，由于整个Internet的运作都依赖于根服务器，每个根服务器都对应多个物理机器，它们分布在世界各地（比如上海就有一个），以提供强大的域名解析服务。关于根服务器的信息可以在[root-servers.org](https://root-servers.org/)查到，截止到 2020 年 3 月 1 日，一共有 1091 个由 12 家独立的组织运营的根域名服务器。

* **Top-Level Domain (TLD) Servers 顶级域服务器** - 负责返回对应顶级域下的权威域名服务器的相关信息。

* **Authoritative DNS Servers 权威 DNS 服务器** - 每个对外提供公共访问的主机（例如Web服务器和邮件服务器）的机构 (Organizations)，都必须提供相应的主机名到IP地址的映射，即 DNS 资源记录。这些机构的权威 DNS 服务器保存这些 DNS 记录。每个机构可以选择实施自己的权威 DNS 服务器，也可以付费将这些记录在某个服务提供商的权威 DNS 服务器中。

还有一类特殊的 DNS 服务器，即**Local DNS Servers - 本地 DNS 服务器**。本地 DNS 服务器并不严格属于服务器的层次结构.每个ISP（比如Residential ISP /Institutional ISP ）都有本地 DNS 服务器，包括很多大学、公司都有自己的 DNS 服务器。当主机连接到 ISP 时，ISP 为主机提供其一台或多台本地 DNS 服务器的 IP 地址。本地 DNS 服务器离用户较近，一般不超过几个路由器的距离。

除此之外，现在还有很多平台提供了公共免费的域名解析服务，比如 Google 的 `8.8.8.8` 和 `8.8.4.4` 还有 Cloudflare 的 `1.1.1.1` 等。

---
### 域名解析
接下来让我们一起来看看在域名解析过程中，这几类域名服务器各自扮演了怎样角色。

以访问`home.sina.com`为例，我们可以用 `dig` 命令，在控制台输入`dig home.sina.com +trace`，这条命令会将 DNS 解析的过程都打印出来，我们一段段来看。

1. 首先，主机发起放向本地 DNS 服务器发起查询，如果本地 DNS 服务器没有相关记录，则会向根域名服务器发起查询。在下图中，我们可以看到主机的本地 DNS 服务器是127.0.0.1，按照前面提到的，每个 DNS 服务器都有一份根域名服务器的名单。
![get root server info][2]

2. 本地 DNS 服务器得到根域名服务器地址后，将查询发给根服务器。在下图中，可以看到本地 DNS 服务器将请求发给了 `e.root-servers.net` 这个根服务器，并得到了 `.com` 这个顶级域名的 TLD 服务器的相关信息。
![get tld server info][3]

3. 本地 DNS 服务器拿到 TLD 服务器的地址，再将整个查询发给它，TLD 服务器返回新浪的权威 DNS 服务器信息，如下图所示，本地 DNS 服务器将请求发给了 `m.gtld-servers.net`，并得到了`sina.com`这个域名的域名服务器信息：
![get authoritative server info][4]

4. 接下来，本地 DNS 服务器向新浪的某个权威 DNS 服务器发送请求，如果该 DNS 服务器包含了查询域名的 IP 的地址，则返回相关RR；否则该 DNS 服务器将返回更底层的 DNS 服务器地址，本地 DNS 服务器向下一层 DNS 服务器发送查询请求，最终获得域名对应的 IP 地址或者查询失败。
在本例中，我们成功得到了 `home.sina.com` 的IP地址，如下图所示：
![get ip address][5]

5. 本地 DNS 服务器将查询到的IP地址返回给主机。

至此，域名解析结束，整个过程如下图所示：
![dns resolution][6]

---
### 查询机制
我们看到上图图示中标示了两种查询机制，结合这个例子说明如下：
* **递归查询** - 主机发送查询请求给本地 DNS 服务器后，由本地 DNS 服务器代替该主机以 DNS 客户端的身份向其他域名服务器发送查询，即本地 DNS 服务器包揽了后续查询，直到得到它所需的结果，这个结果是**完整结果**，即要么是对应的资源记录，要么报错。这个机制被称为递归查询。
* **迭代查询** - 本地 DNS 服务器向根域名服务器发送请求后，根域名服务器返回**部分结果**，这个结果可能是报错、也可能是对应的资源记录，还可能只是告诉本地 DNS 服务器下一步该向谁发起查询。然后本地 DNS 服务器要自己继续负责后续查询。这个机制被称为迭代查询。

---
## 解析器
到目前为止，我们提到域名解析都是以主机为单位，而实际上真正发起 DNS 查询的应该是运行在主机上的用户程序（比如Email, FTP, TELNET）。那么，用户程序究竟如何向域名服务器发起请求呢？这就需要依赖 DNS 的第三大组件 —— **解析器 (Resolver)**。
解析器一般以子程序 (Subroutine) 或系统调用 (System call) 的形式作为接口给应用程序调用，应用程序传参给解析器，解析器返回结果，因此用户程序和解析器之间不需要协议。

解析器必须能够访问至少一个域名服务器并使用该域名服务器的信息直接响应查询或拿到中间结果向其他域名服务器继续查询。

举个例子，Linux 中的解析程序由一组C库中的系统例程组成，通过 `/etc/resolv.conf` 配置文件为解析程序提供域名服务器的信息，如果配置文件不存在，那么解析器将向运行在本机上的域名服务器的发起查询。下面这段代码中的 `getaddrinfo` 是一个可用来解析域名的库函数，将其编译运行，将打印出`google.com` 的 IP 地址。
```c++
#include <stdio.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>
#include <arpa/inet.h>
int main(void) {
  struct addrinfo* addr;
  int result = getaddrinfo("google.com", NULL, NULL, &addr);
  if (result != 0) {
    printf(stderr, "getaddrinfo: %s\n", gai_strerror(result));
    return 1;
  }
  struct sockaddr_in* internet_addr = (struct sockaddr_in*) addr->ai_addr;
  printf("google.com is at: %s\n", inet_ntoa(internet_addr->sin_addr));
  return 0;
}
```
更多关于 `getaddrinfo` 可见 [Linux Programmer's Manual - GETADDRINFO(3)](http://man7.org/linux/man-pages/man3/getaddrinfo.3.html)。

当解析器完成某个请求后，会将收到的结果缓存起来，这样下一次收到同样的请求时，可以直接从本地缓存返回结果，以提高响应速度。当本地缓存中没有结果或者缓存失效时，解析器才会向外部 DNS 服务器发起请求。

### DNS缓存
缓存是必要的，TTL是服务于缓存的。

**Authoritative Record 权威记录** - 由管理该记录的权威部门提供，因此总是正确的。
**Cached Record 缓存记录** - 有可能是过时的。

当缓存记录遇上权威记录，毫无疑问，弃缓存而选权威。

---
## 彩蛋
最后，安利一个很可爱的网站，[How DNS Works](https://howdns.works/)，以漫画的形式生动形象地介绍了什么是DNS，超级萌！:D


<!--
### DNS基于UDP
DNS采用UDP作为传输层协议，DNS消息通过UDP数据包发送，格式简单，只有查询和响应。
为了应对服务器关闭或者丢包的情况，如果在很短的时间内没有响应返回，则DNS客户端必须重复查询请求；如果重复一定次数后仍然失败，则尝试域内另一台域名服务器。

每个查询报文都包含16位标识符，这个标识符会被复制到响应中，以便域名服务器将响应与查询匹配。
-->

---

**参考资料**
* Computer Networking: A Top-Down Approach, 7th Edition
* 计算机网络，第五版
* [DNS Root Servers - What are they and Are They Really Only 13?](https://securitytrails.com/blog/dns-root-servers)
* [RFC 1034 - DOMAIN NAMES - CONCEPTS AND FACILITIES](https://tools.ietf.org/html/rfc1034)
* [RFC 1035 - DOMAIN NAMES - IMPLEMENTATION AND SPECIFICATION](https://tools.ietf.org/html/rfc1035)
* [What is DNS](https://ns1.com/resources/what-is-dns)
* [What does getaddrinfo do](https://jameshfisher.com/2018/02/03/what-does-getaddrinfo-do/)

  [1]:/blog/uploads/images/domain-name-space.jpg
  [2]:/blog/uploads/images/dns-root-server.png
  [3]:/blog/uploads/images/dns-tld-server.png
  [4]:/blog/uploads/images/dns-auth-server.png
  [5]:/blog/uploads/images/dns-ip.png
  [6]:/blog/uploads/images/dns-resolution.svg