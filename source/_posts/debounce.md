---
title: debounce 防抖
date: 2018-11-20 14:55:36
tags:
- js
- 面试题
categories: js 基础
---

函数去抖背后的基本思想是指，某些代码不可以在没有间断的情况连续重复执行。第一次调用函数，创建一个定时器，在指定的时间间隔之后运行代码。当第二次调用该函数时，它会清除前一次的定时器并设置另一个。如果前一个定时器已经执行过了，这个操作就没有任何意义。然而，如果前一个定时器尚未执行，其实就是将其替换为一个新的定时器。目的是只有在执行函数的请求停止了一段时间之后才执行。
比如连续滚动。
```javascript
function debounce (method, wait, context) {
    clearTimeout(method.tId);
    method.tId = setTimeout(function(){
        method.call(context);
    }, wait);
}
```
对`scroll`事件测试：
```javascript
function handle() {
    console.log('scroll');
}
window.addEventListener('scroll', debounce(handle, 1000));
```
连续滚动（每次间隔时间小于1000ms，不会触发`handle`方法，因为每次触发都会取消上一次的延时事件，然后重新设置延时1000ms。只有两次`scroll`事件间隔超过1000ms，前一次的才会被触发。
### debounce更加常规的写法：返回一个闭包
```javascript
function debounce(method, wait) {
    var timer = null;
    console.log('debounce initialized');
    return function() {
        var context = this;
        var args = arguments;
        // 清空上一次定时器
        clearTimeout(timer);
        timer = setTimeout(function(){
            method.apply(context, args);
        }, wait);
    }
}
```
测试：
```javascript
var box = document.getElementById('box');
function handle() {
    console.log('hello');
}
var fun = debounce(handle,1000);
box.addEventListener('scroll',fun);
```
脚本加载后，`debounce initialized`只会被打印一次。`box`的`scroll`事件的触发有1000ms防抖。

:point_right: [demo in jsfiddle](https://jsfiddle.net/xqy_young/Ls2jaxvp/3/?utm_source=website&utm_medium=embed&utm_campaign=Ls2jaxvp)
