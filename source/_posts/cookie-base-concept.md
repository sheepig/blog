---
title: cookie 基础概念
date: 2018-11-20 14:58:46
tags:
- 网络
- 客户端储存
categories: 网络
---

## cookie

我们常说的 cookie ，也称为 HTTP Cookie ，它其实是 HTTP 头部中的一段信息，记录客户端和服务器端的交互中的一些信息。它会在浏览器下次向同一服务器再发起请求时被携带并发送到服务器上。通常，它用于告知服务端两个请求是否来自同一浏览器，如保持用户的登录状态。Cookie 使基于无状态的 HTTP 协议记录稳定的状态信息成为了可能。

### Set-Cookie

[RFC 6265](https://tools.ietf.org/html/rfc6265#section-4.1) 规定，服务端可以在 HTTP 响应头部设置 cookie。格式：

```
Set-Cookie:<cookie>
```

cookie 以一对键-值对开始，后面跟着若干属性-值对。属性的详细信息查看 [MDN Set-Cookie](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Set-Cookie)。

| Attribute | value | detail |
| ----------| ----- | ------ |
| Expires | 符合 HTTP-date 规范的时间戳 | 若不设置，说明这是一个会话期 cookie |
| Max-Age | cookie 存活的时间，单位是秒。 | 优先级比 Expires 高|
| Domain | cookie 可以送达的主机名 | 假如没有指定，那么默认值为当前文档访问地址中的主机部分（但是不包含子域名）|
| Path | 指定一个 URL 路径，这个路径必须出现在要请求的资源的路径中才可以发送 Cookie 首部 | 字符  %x2F ("/") 可以解释为文件目录分隔符，此目录的下级目录也满足匹配的条件（例如，如果 path=/docs，那么 "/docs", "/docs/Web/" 或者 "/docs/Web/HTTP" 都满足匹配的条件）。如果没有设置这个值，用户代理会默认使用 HTTP 请求 URI 的 path |
| Secure | 只有在浏览器认为当前通道是“安全”的，才会把带有 Secure 选项的 cookie 发送到服务端 | 浏览器认为的安全，通常是在 HTTPS 下 |
| HttpOnly | 设置了 HttpOnly 属性的 cookie 不能使用 JavaScript 经由  Document.cookie 属性、XMLHttpRequest 和  Request APIs 进行访问，以防范跨站脚本攻击（XSS）|  |

一个 Set-Cookie 首部字段，只能设置一个 cookie，如果要设置多个 cookie ，直接用多个 Set-Cookie 字段。

```
Set-Cookie:<cookie1>
Set-Cookie:<cookie2>
```

### cookie的读写

原生 node 的 res.setHeader 和 res.writeHead ，如果直接对 Set-Cookie 赋值，会出现覆盖的情况。其实也好理解，对对象的某个属性重复写入值，只有最后一个会保留。

```javascript
res.setHeader("Set-Cookie", "yang=ozz; Max-Age=500");
```

要在 HTTP 头部加入多个 Set-Cookie 字段，需要对响应流拼接多个 Set-Cookie 选项。为了节省时间，这里用 express 。

设置以下 cookie

```
Set-Cookie: yan=eee; Max-Age=600; Path=/; Expires=Invalid Date
Set-Cookie: agg=sss; Max-Age=600; Path=/; Expires=Invalid Date
```

通过 document.cookie 或直接在浏览器 Application 面板中，可以直接访问，修改 cookie 。

```
document.cookie // "yan=eee; agg=sss"
document.cookie = "yan=www";
document.cookie // "agg=sss; yan=www"
```

document.cookie 修改 cookie 只能一个个修改，格式和 Set-Cookie 一样。没有指定修改的属性保持不变，修改的值便覆盖原来的值。

#### HttpOnly

HttpOnly 只能由服务端设置，本地的设置不会生效。而且设置了 HTTPOnly 的 cookie 无法通过 document.cookie 访问。

```
Set-Cookie: yan=eee; Max-Age=600; Path=/; Expires=Invalid Date; HttpOnly
Set-Cookie: agg=sss; Max-Age=600; Path=/; Expires=Invalid Date
```

```
document.cookie // "agg=sss" 
```

### cookie 删改

修改 cookie

要想修改一个cookie，只需要重新赋值就行，旧的值会被新的值覆盖。但要注意一点，在设置新cookie时，path/domain这几个选项一定要旧cookie 保持一样。否则不会修改旧值，而是添加了一个新的 cookie。

删除 cookie

删除一个cookie 也挺简单，也是重新赋值，只要将这个新cookie的expires 选项设置为一个过去的时间点就行了。但同样要注意，path/domain/这几个选项一定要旧cookie 保持一样。

原生的 cookie 的读写接口都不太友好，一般需要封装方法。

 - 读取 cookie

document.cookie 的结果是一串字符串，是以“; ”分隔开的 cookie 键值对。

```javascript
function getCookie(name) {
    var ary = document.cookie.match(new RegExp("(^| )" + name + "=([^;]*)(;|$)"));
    // 第二个捕获组就是 key=value; 的 value
    return ary[2] ? ary[2] : null;
}

getCookie('agg');
```

```javascript
// options
// |-- expires  自设置起过期秒数
// |-- callback 设置成功后回调
function setCookie(name, value, options) {
    options = options ? options : {};
    var exp = new Data();
    var expire = "";
    if (options.expires) {
        exp.setTime(exp.getTime + expires * 1000);
        expire = exp.toGMTString();
    }
    document.cookie = name + "=" + escape(value) + ";path=/;expires=" + expire;
    options.callback && options.callback();
}
```