---
title: js 作用域相关问题
date: 2018-11-20 11:53:03
tags:
- js
- 面试题
categories:
- js 基础
---

## 作用域、闭包相关问题集锦

### IIFE

[MDN IIFE（ 立即调用函数表达式）](https://developer.mozilla.org/zh-CN/docs/Glossary/%E7%AB%8B%E5%8D%B3%E6%89%A7%E8%A1%8C%E5%87%BD%E6%95%B0%E8%A1%A8%E8%BE%BE%E5%BC%8F)

判断以下执行结果：

```javascript
var env = 'outer';
(function(){
    console.log(env);
    var env = 'inner';
})();
```

```javascript
var env = 'outer';
(function(){
    var env = 'inner';
    console.log(env); // inner
})();
```

### 扩展 console.log 方法

扩展 console.log 方法，每个输出前增加一个自增序号。

```javascript
function myLog (fn) {
    id = 0;
    return function() {
        id++;
        var fn_args = [].slice.call(arguments);
        fn_args.unshift(id+':');
        fn.apply(null, fn_args);
    }
}

console.log = myLog(console.log);

var num = 2;
console.log('foo');       // 1: foo
console.log('bar');       // 2: bar
console.log('num:', num); // 3: num: 2
```