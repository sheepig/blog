---
title: throttle 节流
date: 2018-11-20 14:57:03
tags:
- js
- 面试题
categories: js 基础
---

函数节流（throttle）：当持续触发事件时，保证一定时间段内只调用一次事件处理函数。节流通俗解释就比如我们水龙头放水，阀门一打开，水哗哗的往下流，秉着勤俭节约的优良传统美德，我们要把水龙头关小点，最好是如我们心意按照一定规律在某个时间间隔内一滴一滴的往下滴。
### 时间戳方法
```javascript
/**
 * 时间戳方法
 * @param   {function} method [节流的方法]
 * @param   {number}   delay  [间隔执行时间]
 * @returns {function}        []
 */
function throttle(method, delay) {
    var prev = Date.now();
    return function() {
        var context = this;
        var args = arguments;
        var now = Date.now();
        if (now - prev > delay) {
            method.apply(context, args);
            prev = Date.now();
        }
    }
}
```
看一下用法
```javascript
var box = document.getElementById('box');
function handle() {
    console.log('hello');
}
var fun = throttle(handle,1000);
box.addEventListener('scroll',fun);
```
时间戳方法，一般第一次`scroll`就会触发`method`。（除非脚本加载完1s内，`box`就触发了`scroll`事件。。。）最后一次`scroll`不会触发`method`（假设高频事件，最后两次事件挨着）:ghost:[demo online](https://jsfiddle.net/xqy_young/Lzk9a7r4/)

### 定时器方法
```javascript
/** 定时器方法
 * @param   {function} method [节流的方法]
 * @param   {number}   delay  [间隔执行时间]
 * @returns {function}        []
 */
function throttle(method, delay) {
    var timer = null;
    return function() {
        var context = this;
        var args = arguments;
        if (!timer) {
            timer = setTimeout(function() {
                method.apply(context, args);
                timer = null;
            }, delay);
        }
    }
}
```
使用定时器之后，无论中间有没有中断`scroll`事件，每次的第一次触发都会延迟1000ms。当最后一次停止触发后，由于定时器的delay延迟，可能还会执行一次函数。
:ghost:[demo online](https://jsfiddle.net/xqy_young/j698yg5z/)

### 定时器+时间戳
节流中用时间戳或定时器都是可以的。更精确地，可以用时间戳+定时器，当第一次触发事件时马上执行事件处理函数，最后一次触发事件后也还会执行一次事件处理函数。
```javascript
/**
 * 定时器+时间戳
 * @param   {function} method [节流的方法]
 * @param   {number}   delay  [间隔执行时间]
 * @returns {function}        []
 */
function throttle(method, delay) {
    var prev = Date.now();
    var timer = null;
    return function() {
        var now = Date.now();
        var remain = delay - (now - prev);
        var context = this;
        var args = arguments;
        if (remain <= 0) {
            method.apply(context, args);
            prev = Date.now();
        } else {
            timer = setTimeout(function(){
                method.apply(context, args);
            }, delay)
        }
    }
}
```
