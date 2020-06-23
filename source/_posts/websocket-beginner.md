---
title: 初识 WebSocket
date: 2018-11-25 15:03:25
tags:
- 网络
categories: 网络
---

WebSocket 是一种协议，[RFC 6455](https://tools.ietf.org/html/rfc6455) 指出了它的目的：为基于浏览器的应用提供一种机制，使得应用可以在不依赖多条 HTTP 链接的情况（使用 XMLHttpRequest ，iframe 或长轮询）下与服务器进行通信。

### WebSocket 链接过程

接下来用一个简单的例子，了解 WebSocket 链接的建立和通信过程。

Server 端：

```javascript
const WebSocket = require('ws');
 
const wss = new WebSocket.Server({ port: 8080 });
 
wss.on('connection', function connection(ws) {
  ws.on('message', function incoming(message) {
    console.log('received: %s', message);
  });
 
  ws.send('something');
});
```

Client 端：

```javascript
const socket = new WebSocket('ws://localhost:8080');

// Connection opened
socket.addEventListener('open', function (event) {
  socket.send('Hello Server!');
});

// Listen for messages
socket.addEventListener('message', function (event) {
  console.log('Message from server ', event.data);
});
```

#### TCP 三次握手

WebSocket 本质上还是基于 TCP 协议，所以 WebSocket client 和 server 通信的第一步，依然是建立 TCP 链接。这一点和 HTTP 协议的表现差不多。

#### 协议升级

WebSocket 建立链接的握手过程，是要适配基于 HTTP 的服务端软件和中间件的，所以一个端口，可以同时允许一个 HTTP 客户端和一个服务器通信，或者一个 WebSocket 客户端和这个服务器通信。

![](websocket-client.png)

TCP 建立后，客户端发起协议升级请求，下面是请求头部的部分信息。WebSocket 要求 HTTP 版本至少为 1.1 ，并且只支持 GET 请求。

```
GET ws://localhost:8080/ HTTP/1.1
Host: localhost:8080
Connection: Upgrade
Upgrade: websocket
Origin: http://localhost:3000
Sec-WebSocket-Version: 13
Sec-WebSocket-Key: 8NgftYvA8Vyr7cGRS8WdpA==
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
```

 - Upgrade 字段，值为 `websocket` ，表示要升级到 WebSocket 协议。
 - Sec-WebSocket-Version 字段，表示 WebSocket 的版本。如果服务端不支持该版本，需要返回一个 `Sec-WebSocket-Version` header ，里面包含服务端支持的版本号。
 - Sec-WebSocket-Key 字段，与后面服务端响应首部的 Sec-WebSocket-Accept 是配套的，用于告知客户端，服务器愿意初始化 WebSocket 链接。
 - Sec-WebSocket-Extensions 可选字段，表示协议级别上（可能为空）服务器可以使用的扩展。

服务端同意协议升级：

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: JAz1fxUy5f9LbUgeVcToNl0yWzA=
```

#### Sec-WebSocket-Accept 的计算

1. 将客户端的 `Sec-WebSocket-Key` 跟 `258EAFA5-E914-47DA-95CA-C5AB0DC85B11` 拼接。
2. 计算拼接值的 SHA1 哈希值，转换为 Base64 编码。

`Sec-WebSocket-Key` 和 `Sec-WebSocket-Accept` 这一对配套的头部信息有什么用呢？[RFC 6455](https://tools.ietf.org/html/rfc6455#page-7) 是这样解释的：服务端要给客户端一个准信，它已经收到服务端的 WebSocket 握手请求了，针对这个客户端，服务端不接受非 WebSocket 链接，这样可以防止攻击者通过 XMLHttpRequest 或表单提交向 WebSocket 服务器发送伪装过的包。

#### 数据帧格式

WebSocket 客户端、服务端通信的最小单位是帧（frame），由 1 个或多个帧组成一条完整的消息（message）。

发送端：将消息切割成多个帧，并发送给服务端；

接收端：接收消息帧，并将关联的帧重新组装成完整的消息；

##### 帧格式详解

参考 [RFC 6455 5.2](https://tools.ietf.org/html/rfc6455#section-5.2)

```
      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-------+-+-------------+-------------------------------+
     |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
     |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
     |N|V|V|V|       |S|             |   (if payload len==126/127)   |
     | |1|2|3|       |K|             |                               |
     +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
     |     Extended payload length continued, if payload len == 127  |
     + - - - - - - - - - - - - - - - +-------------------------------+
     |                               |Masking-key, if MASK set to 1  |
     +-------------------------------+-------------------------------+
     | Masking-key (continued)       |          Payload Data         |
     +-------------------------------- - - - - - - - - - - - - - - - +
     :                     Payload Data continued ...                :
     + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
     |                     Payload Data continued ...                |
     +---------------------------------------------------------------+
```

**FIN** ，1 比特

指示这个帧是不是消息的最后一个分片（fragment）。1 表示是，0 表示否。

**RSV1，RSV2，RSV3** ，3 比特

这个是跟 WebSocket 扩展有关的，全 0 表示没有使用任何扩展。

**Opcode** ，4 比特

定义了对载荷数据（Payload Data）的解析。它有以下值：

 - `%x0` 表示一个延续帧，本次数据传输采用了数据分片，当前收到的数据帧为其中一个数据分片。

 - `%x1` 表示一个文本帧

 - `%x2` 表示一个二进制帧

 - `%x3-7` 预保留

 - `%x8` 表示链接关闭

 - `%x9` 表示一个 ping 操作

 - `%xA` 表示一个 pong 操作

 - `%xB-F` 预保留

**Mask** ，1 比特

表示是否要对数据载荷进行掩码操作。从客户端向服务端发送数据时，需要对数据进行掩码操作；从服务端向客户端发送数据时，不需要对数据进行掩码操作。

如果服务端接收到的数据没有进行过掩码操作，服务端需要断开连接。

如果Mask是1，那么在Masking-key中会定义一个掩码键（masking key），并用这个掩码键来对数据载荷进行反掩码。所有客户端发送到服务端的数据帧，Mask 都是 1 。

**Payload length**，7 比特，7+16 比特，或 7+64 比特

Payload data 的长度按字节计算，如果 Payload data 的长度为：

 - 0-125 字节，那么 Payload length 就是这个值
 - 126 字节，后续 2 个字节代表一个 16 位的无符号整数，该无符号整数的值为数据的长度
 - 127 字节，后续8个字节代表一个64位的无符号整数（最高位为0），该无符号整数的值为数据的长度

**Masking-key** 0 或 4 字节

所有从客户端传送到服务端的数据帧，数据载荷都进行了掩码操作，Mask 为 1，且携带了 4 字节的 Masking-key。如果 Mask 为 0 ，则没有 Masking-key 。

备注：载荷数据的长度，不包括mask key的长度。

**Payload data** (x+y) 字节

包括了扩展数据、应用数据。其中，扩展数据x字节，应用数据y字节。

#### 数据传递

客户端和服务器建立 WebSocket 链接后，后续的操作都是基于数据帧。WebSocket 根据 `opcode` 来区分操作的类型。比如 0x8 表示断开连接，0x0-0x2 表示数据交互。

下面是[MDN](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API/Writing_WebSocket_servers)的例子：

```
// 文本帧，消息已经发送完毕，没有后续的帧
Client: FIN=1, opcode=0x1, msg="hello"
Server: (process complete message immediately) Hi.

// 文本帧，消息未发送完毕，还有后续的帧
Client: FIN=0, opcode=0x1, msg="and a"
Server: (listening, new message containing text started)
// 分片帧，消息未发送完毕，还有后续的帧，当前的数据帧需要接在上一条数据帧之后
Client: FIN=0, opcode=0x0, msg="happy new"
Server: (listening, payload concatenated to previous message)
// 分片帧，消息发送完毕，没有后续的帧，当前的数据帧需要接在上一条数据帧之后
Client: FIN=1, opcode=0x0, msg="year!"
Server: (process complete message) Happy new year to you too!
```

#### 断开链接

网络断开或页面关闭，会导致 WebSocket 关闭。如果通信双方长时间（默认 60 s）无数据交互，链接也会断开。断开的操作其实是 TCP 四挥手。

如果希望客户端和服务端保持稳定的链接，可以定时向对方发送心跳包。

 - 发送方->接收方：ping
 - 接收方->发送方：pong

ping、pong 的操作，对应的是 WebSocket 的两个控制帧，opcode 分别是 0x9、0xA 。

有个工具[browser-sync](https://www.browsersync.io/) 就用到 WebSocket ，它可以做到“一端更新，多端同步”的效果。其实就是在你开发的页面注入了一段脚本，在这个脚本里面启用了 WebSocket ，和本地服务通信。所以可以实现：浏览器->服务端->其他客户端 的刷新机制。

### 细节

#### 掩码算法

ENCODED：原码

MASK：掩码键

掩码、反掩码都是采用以下算法：

```javascript
var DECODED = "";
for (var i = 0; i < ENCODED.length; i++) {
    DECODED[i] = ENCODED[i] ^ MASK[i % 4];
}
```

一下转载自[WebSocket协议：5分钟从入门到精通](https://www.cnblogs.com/chyingp/p/websocket-deep-in.html)

WebSocket 协议中，数据掩码的作用是增强协议的安全性。但数据掩码并不是为了保护数据本身，因为算法本身是公开的，运算也不复杂。除了加密通道本身，似乎没有太多有效的保护通信安全的办法。

那么为什么还要引入掩码计算呢，除了增加计算机器的运算量外似乎并没有太多的收益。

答案还是两个字：安全。但并不是为了防止数据泄密，而是为了防止早期版本的协议中存在的代理缓存污染攻击（proxy cache poisoning attacks）等问题。

在正式描述攻击步骤之前，我们假设有如下参与者：

 - 攻击者、攻击者自己控制的服务器（简称“邪恶服务器”）、攻击者伪造的资源（简称“邪恶资源”）
 - 受害者、受害者想要访问的资源（简称“正义资源”）
 - 受害者实际想要访问的服务器（简称“正义服务器”）
 - 中间代理服务器

攻击步骤一：

1. 攻击者浏览器 向 邪恶服务器 发起WebSocket连接。根据前文，首先是一个协议升级请求。
协议升级请求 实际到达 代理服务器。
2. 代理服务器 将协议升级请求转发到 邪恶服务器。
3. 邪恶服务器 同意连接，代理服务器 将响应转发给 攻击者。
4. 由于 upgrade 的实现上有缺陷，代理服务器 以为之前转发的是普通的HTTP消息。因此，当协议服务器 同意连接，代理服务器 以为本次会话已经结束。

攻击步骤二：

1. 攻击者 在之前建立的连接上，通过WebSocket的接口向 邪恶服务器 发送数据，且数据是精心构造的HTTP格式的文本。其中包含了 正义资源 的地址，以及一个伪造的host（指向正义服务器）。（见后面报文）
2. 请求到达 代理服务器 。虽然复用了之前的TCP连接，但 代理服务器 以为是新的HTTP请求。
3. 代理服务器 向 邪恶服务器 请求 邪恶资源。
4. 邪恶服务器 返回 邪恶资源。代理服务器 缓存住 邪恶资源（url是对的，但host是 正义服务器 的地址）。

到这里，受害者可以登场了：

1. 受害者 通过 代理服务器 访问 正义服务器 的 正义资源。
2. 代理服务器 检查该资源的url、host，发现本地有一份缓存（伪造的）。
3. 代理服务器 将 邪恶资源 返回给 受害者。
4. 受害者 卒。

附：前面提到的精心构造的“HTTP请求报文”。

```
Client → Server:
POST /path/of/attackers/choice HTTP/1.1 Host: host-of-attackers-choice.com Sec-WebSocket-Key: <connection-key>
Server → Client:
HTTP/1.1 200 OK
Sec-WebSocket-Accept: <connection-key>
```

最初的提案是对数据进行加密处理。基于安全、效率的考虑，最终采用了折中的方案：对数据载荷进行掩码处理。

需要注意的是，这里只是限制了浏览器对数据载荷进行掩码处理，但是坏人完全可以实现自己的WebSocket客户端、服务端，不按规则来，攻击可以照常进行。

但是对浏览器加上这个限制后，可以大大增加攻击的难度，以及攻击的影响范围。如果没有这个限制，只需要在网上放个钓鱼网站骗人去访问，一下子就可以在短时间内展开大范围的攻击。

> 参考
> [WebSocket协议：5分钟从入门到精通](https://www.cnblogs.com/chyingp/p/websocket-deep-in.html)
> [RFC 6455](https://tools.ietf.org/html/rfc6455)