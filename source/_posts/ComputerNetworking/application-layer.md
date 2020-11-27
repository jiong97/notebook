---
title: 应用层
date: 2020-11-16 19:47:23
categories: [ComputerNetworking]
excerpt: '应用层是计算机网络的最上层，提供了应用间的通讯方式。它通过下层协议到达远程主机。上层协议往往不管下层是如何传输的，它只要调用下层协议提供的服务'
---
计算机网络是分层的，上层协议往往不管下层是如何传输的，它只要调用下层协议提供的服务。在网络的两个主机间的不同进程上通讯，使用的是操作系统提供的Socket接口。应用通过Socket来使用下层协议的功能，相当于两个通讯实体间是对等的。同过IP地址表示不同主机，使用端口（port）表示一台主机上的不同进程。不同协议需要不同的传输服务。

各种开放协议定义了交互规则。传输层提供了各种服务：

- 可靠性，如HTTP
- 时延
- 带宽
- 安全性

# HTTP

Web页面有一系列对象组成，如HTML文件、图片、视频，通过URL地址寻址（Uniform Resource Locator）。

> scheme://host:port/path

如 http://www.yoursite.com/index.html ，其中http是协议，www.yoursiet.com 是主机名，/index.html是路径名。

http定义了客户端与服务器间请求、响应的方式。客户端请求时首先发起tcp连接，然后通过socket访问tcp。用户进程通过socket访问tcp，从socket发送http请求，和接受http响应。

HTTP1.0版本中使用非持久性连接，每个tcp连接最多传输一个对象，并且该请求响应结束后会断开tcp连接，也就是说只能传输一个请求报文和响应报文。1.1默认使用持久性连接，且允许传输多个对象。持久性连接消耗两个RTT，一个用于建立连接，一个用于请求文件。持久性连接支持流水线的性质，客户端可以发出多个请求，而不用等待前面未响应的请求（HTTP1.1）。

## HTTP报文格式

### HTTP请求报文

![HTTP请求报文格式](/images/computer-networking/application-layer/http-request-format.png)

![HTTP请求例子](/images/computer-networking/application-layer/http-request-message.png)

第一行为请求行（request line），其他的为头部行。Connection: close告诉服务器不使用持久性连接，在该请求完成后断开连接。使用GET是实体为空，使用POST是，可以用实体来提交表单内容。

### HTTP响应报文

![HTTP响应报文格式](/images/computer-networking/application-layer/http-response-format.png)

![HTTP响应报文例子](/images/computer-networking/application-layer/http-response-message.png)

它包含状态行、首部行、实体。首部行中，Date是产生和发送该响应报文的时间日期；Last-Modified为对象最后一次修改时间，这对本地缓存和缓存服务器至关重要；Content-Length指示被发送对象的字节数；Content-Type为实体对象类型。

- 200 OK
- 301 Moved Permanently：请求对象被永久转移，新URL在响应报文的Location首部中
- 304 Not Modified
- 400 Bad Request：通用错误码，表示请求不被服务器理解
- 404 Not Found
- 505 HTTP Version Not Supported：服务器不支持该HTTP版本

## Cookie

虽然HTTP是无状态协议，但是可以通过cookie来记录状态。在请求和响应报文中各有一个cookie头部。

响应报文可以在头部中增加Set-cookie来添加cookie，如Set-cookie: 2333，浏览器收到HTTP响应后会在cookie文件中添加，改行包含主机名和头部内容。每当继续请求同样的站点时，浏览器都会从cookie文件提取该内容，并放到请求报文的cookie头部。

## Web缓存/代理服务器

Web cache也叫代理服务器（proxy server），使用来代替初始web服务器来满足HTTP请求的实体。代理服务器保存有最近请求的副本。代理服务器是可配置的，一旦配置之后，所有请求都会被定向到代理服务器。

![请求资源位于缓存服务器](/images/computer-networking/application-layer/web-cache.png)

当浏览器向代理服务器发送请求后，proxy首先检查是否存储了该对象的副本，如果有，proxy就向浏览器返回该对象。如果proxy中没有该对象，则proxy向原始服务器发送HTTP请求，proxy得到响应后，在本地创建一份副本，并返回给浏览器（复用之间建立的TCP连接）。proxy可以有效降低公网访问的流量强度。

## 条件GET

proxy中的副本有可能是旧的，保存在服务器上的对象已经被修改过。通过在HTTP报文中包含If-Modified-Since头部，来保证获取的对象是新的。

当proxy第一次向服务器发送请求后，得到的结果会包含Last-Modified头部的响应。

![包含Last-Modified头部的响应报文](/images/computer-networking/application-layer/Last-Modified.png)

proxy缓存该数据，也存储了最后修改时间。当下次浏览器向proxy访问时，proxy会首先向服务器发送条件GET，If-Modified-SInce的值恰好为上次服务器响应的Last-Modified的值。

![包含修改日期的请求报文](/images/computer-networking/application-layer/If-modified-since.png)

会得到类似响应：

![资源没有过期，状态码为304](/images/computer-networking/application-layer/If-modified-since-response.png)

如果状态码为304，说明该缓存没有过期。

# DNS

域名系统（Domain Name System）用来将主机名（host）转换为IP地址。DNS是分层的分布式数据库。DNS运行在UDP之上，使用53端口。

DNS还提供别名服务，一台名为a.yoursite.com的主机，可能有两个别名b.yoursite.com和c.yoursite.com，其中a.yoursite.com称为规范主机名（canonical hostname），使用DNS可以获得主机别名名对应的规范主机名和IP地址。

DNS提供邮件别名服务，一个公司的邮件服务器的主机名可能非常复杂，邮件应用程序可以调用DNS，对提供的邮件别名经行解析，以获得规范主机名及IP，这样邮件服务器就能与Wen服务器同名。

DNS还提供负载分配（load distribution）的功能。一个站点可以分布在多个主机上，每个主机对于多个IP，多个IP地址可以对应一个主机名，DNS服务器存储着这些IP地址集合。当客户端发送DNS请求时，总是以整个IP地址集合作为响应，但在每个回答中循环这些地址的次序。因为客户端总是向排在第一位的IP发送请求。

## 工作原理

如果某些应用程序需要将主机名转换为IP，则需调用DNS客户端，用户主机DNS接受到请求后，向网络发送DNS查询报文。所有DNS请求和回答报文都使用53端口发送。经过若干毫秒后，主机DNS接收到DNS回答报文。这个过程十分复杂，由分布在全球的大量DNS服务器完成。

DNS是一个分布式数据库，其架构如下所示：

![DNS服务器的层次体系](/images/computer-networking/application-layer/hierarchy-of-DNS-servers.png)

没有一台DNS拥有因特网所有主机的映射，该映射分布在所有DNS服务器上。大致有三种DNS服务器：根服务器、顶级域{top level domain，TLD）服务器、权威服务器。假定用户要决定 www.amazon.com 的IP地址。客户端首先与根服务器之一联系，之后它将获得com域名的TLD服务器的IP地址。该客户端之后与这些TLD服务器之一联系，TLD将返回amozon.com的权威服务地址。最后，客户端与该权威服务器之一取得联系，它将返回主机名 www.amozon.com 的IP地址。

在公共可访问主机的组织机构必须提供可访问的DNS记录。一个组织的权威DNS服务器收藏了这些DNS记录。一个机构可以选择实现自己的权威DNS服务器来保存这些记录，也可以付费将这些记录存储在某个服务提供商一个权威DNS服务器中。

TDL服务器并不总是知道权威服务器的IP地址，相反，它只是知道中间某个DNS服务器，该中间服务器依次才能知道用于该主机的权威服务器。

## 迭代查询与递归查询

![迭代查询](/images/computer-networking/application-layer/interaction-DNS.png)

![递归查询](/images/computer-networking/application-layer/recursive-DNS.png)

从实践上，请求主机到本地DNS服务器是递归的，其余查询是迭代的。

## DNS缓存

在一个请求链中，当某个DNS服务器接收到一个回答时，它能将该回答缓存到本地。DNS服务器通常会在一段时间后丢弃缓存。

## 记录和报文

### RR

共同实现DNS分布式数据库的所有DNS服务器存储了从主机名到DNS映射的资源记录（Resource Record，RR）。DNS回答报文包含一条或多条RR。

RR是一个包含下列字段的四元组：

> (Name, Value, Type, TTL)

TTL决定了该记录的生存时间，它决定了RR应该从缓存删除的时间。Name和Value的值取决于Type。

- Type=A，则Name是主机名，Value是该主机对应的IP。因此，一条A记录是一个主机到IP的映射
- Type=NS，则Name是个域（foo.com），Value是一个知道如何获取该域中主机IP的权威DNS服务器的主机名。这个记录用来沿着查询链路由DNS查询。(foo.com, dns.foo.com, NS)
- Type=CNAME，Value是别名为Name的主机对应的规范主机名。该记录用来向客户端提供一个主机名对应的规范主机名。(foo.com, a47563743.bar.foo.com, CNAME)
- Type=MX,，则value是别名为NAME的邮件服务器的规范主机名。MX允许邮件服务其具有简单的别名。一个公司的邮件服务器和其他服务器可以使用相同的别名。为了获得邮件服务器的规范主机名，DNS客户端应该请求一个MX记录

即使某台DNS服务器不是某个主机的权威服务器，它也能缓存A记录。如果一个DNS服务器不是某主机的权威服务器，那么该主机将包含一条NS记录，还有一条A记录。该A记录提供了NS记录的Value字段中DNS服务器的IP地址。

### DNS报文

DNS查询与回答报文有相同的格式。

![DNS报文格式](/images/computer-networking/application-layer/DNS-message-format.png)

- 前12字节是首部区域，标识符是一个用标识查询的16比特数，这个标识会被复制到回答报文中，以便客户端匹配请求和回答。标志（flag）有若干标志位，1比特用于标识查询（0）和响应（1）。1比特用于在回答报文标识是否为权威服务器。1比特用于在查询查询报文中标识是否递归查询。还有4个有关数量的字段，这些字段指出首部后的4类数据区域出现的数量。
- 问题区包含正在进行的查询信息。分为两个字段。名字字段指出正在被查询的主机名。类型字段指出正在被查询的问题类型（A、NX等）
- 权威区域包括其他权威服务器的记录
- 附加区域包含其他有用的记录

### 工具

nslookup工具可以向任何DNS服务器发送任何DNS查询报文。

## DNS注册

向注册机构注册域名是，需向该机构提供所有权威服务器名和IP地址。该机构将确保一个NS记录和一个A记录输入TLD服务器。如(foo.com, dns.foo.com, NS)、(dns.foo.com, 222.222.222.1, A)。

还必须保证Web服务器www的A记录，邮件服务其的MX记录在权威服务其中。
