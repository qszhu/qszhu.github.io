---
layout: post
title: 用Node.js实现SOCKS代理 
date: 2020-09-30 20:00:00 +0800
tags: [node.js, socks]
---

### 1. 背景

Node.js生态中存在很多包含原生模块的包，在安装时要么需要下载预编译好的文件，要么需要在宿主机上编译。无论哪种情况都会拖慢CI/CD的速度。编译应该很好理解，为什么下载也会拖慢速度？因为node原本只支持安装时编译，为了加快安装过程很多包都会在安装脚本中尝试下载与宿主系统匹配的预编译文件。当这些文件处于访问困难的网站上时下载速度就会很慢，甚至导致下载失败而fallback到编译的手段。即使是使用国内的npm源也是这样，因为这些包的安装脚本中的下载地址通常都是写死的，不受npm源的影响。

一直以来我都是通过设置代理的方式来加速下载，不过这些代理软件都不是js实现的。基于Atwood定律[1]，我决定尝试用Node来实现相同的功能。

### 2. 分析

代理协议首选SOCKS，有以下优点：

* 支持广泛。无论是操作系统还是浏览器，一般都可以设置SOCKS代理
* 工作在会话层。这意味着对于应用层是透明的，可以支持所有的应用层协议，如http/https等
* 协议简单。SOCKS4的协议描述[2]只有一屏。协议简单意味着实现也简单
* 目前常用的协议是v4a[3]和v5[4]，客户端可以将要访问的域名发送给代理，由代理进行解析。这可以避免很多本地DNS失效的问题

而使用Node.js实现有些额外的好处。因为Node中的Socket是一个Duplex流，这意味着核心的代理代码只有两行：

```javascript
downstreamSocket.pipe(upstreamSocket)
upstreamSocket.pipe(downstreamSocket)
```

即把下游客户端和上游服务器的读写流分别进行管道对接。同时因为是Node.js的流，所以自带了back pressure处理，根据测试[5]可以进一步减少GC和提升性能。

### 3. 实现要点

实现当然远不止两行，流程大致如下：

* 建立连接后读取第一个字节，来决定使用哪个版本的协议
* v4的协议在第一次握手时就会建立上游的连接；v5是在第二次才连接，需要区分当前连接进行到了哪一步（使用状态机）
* 握手结束就可以对接上下游了。不过用`pipe()`的话需要自行处理错误和释放资源，可以使用`pump`[6]来简化工作。Node v10之后则可以用`pipeline()`

### 4. 改进

实现以后部署在远程服务器上就可以当成代理使用了。不过仍会遇到问题。因为SOCKS协议虽然是二进制的，但仍旧是明文的，恶意的中间人在识别出协议后可对其进行监控、干扰甚至篡改，这就需要对传输的数据进行加密。为此可以引入本地的反向代理，在发送数据之前先对数据进行加密，收到数据之后先解密，像是这样：

![结构](/assets/images/2020-09-30/arch.jpg)

这就需要我们实现一些自定义流。在反向代理中可以实现两个Transform流，一个加密，一个解密，然后代码就可以这样写：

```javascript
downstream.pipe(cipher).pipe(upstream)
upstream.pipe(decipher).pipe(downstream)
```

在代理中则可以通过包装socket实现自定义的Duplex流，在从socket中读取后解密，在写入socket前加密。

其他一些实现要点：

* 留意back pressure机制，例如当`write()`返回`false`时就需要暂时停止写入，等待写入流的`drain`事件后再恢复
* 自定义Readable流时错误通过`destroy()`传播，自定义Writable流时错误通过`callback()`传播

详情见官方文档[7]。

### 5. 小结

* 要想解决本文开头的问题，还需满足以下条件：
  * 由于npm不支持SOCKS代理，需要有一个能使用SOCKS代理的HTTP代理。可以选择现成的npm包，或是自行实现
  * 具备高速线路的远程服务器
* Node.js的流是一种很方便的抽象。印象中似乎没有其他语言的运行时有提供带back pressure的流，应该可以说是Node的特色功能吧。网上相关的资料似乎也比较少，官方文档也比较简略。可能是因为用起来太简单了，大家都觉得没什么好说的吧🤪

### 参考资料
* [1] https://en.wikipedia.org/wiki/Jeff_Atwood
* [2] https://www.openssh.com/txt/socks4.protocol
* [3] https://www.openssh.com/txt/socks4a.protocol
* [4] https://www.ietf.org/rfc/rfc1928.txt
* [5] https://nodejs.org/en/docs/guides/backpressuring-in-streams/
* [6] https://github.com/mafintosh/pump
* [7] https://nodejs.org/api/stream.html#stream_api_for_stream_implementers