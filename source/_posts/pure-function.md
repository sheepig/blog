---
title: 纯函数
date: 2018-11-20 11:50:34
tags: 
- js
- 函数式编程
- 面试题
categories: js 基础
---

## 纯函数

> 参考[JS函数式编程-Pure Happiness with Pure Functions](https://mostly-adequate.gitbooks.io/mostly-adequate-guide/ch03.html)

这里不探讨一些概念，直接看纯函数的应用。以下是来自原书的例子。

### 小试身手

#### 输入缓存

实现以下要求：

```javascript
squareNumber(4);
//=> 16

squareNumber(4); // 从缓存中读取输入值为 4 的结果
//=> 16

squareNumber(5);
//=> 25

squareNumber(5); // 从缓存中读取输入值为 5 的结果
//=> 25
```

实现：

```javascript
var memorize = function(fn) {
    const cache = {};

    return (...args) => {
        const argStr = JSON.stringify(args);
        cache[argStr] = cache[argStr] || fn(...args);
        return cache[argStr];
    }
}

var squareNumber = memorize(x => x * x);
```