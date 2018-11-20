---
title: 浏览器事件机制
date: 2018-11-20 15:00:51
tags:
- 浏览器
- 优化
categories: 浏览器
---

要知道浏览器是怎么接收输入（滚动，鼠标移动，input 事件，对浏览器来说都是输入），需要明白在页面的呈现的哪个步骤，对事件作出响应。

简单讲一下页面呈现步骤：

浏览器的 render process 构建了 DOM Tree，CCSOM tree ，它知道文档的结构以及每个节点的样式，但仅有这些信息不足以渲染一个页面。这时需要 layout process 去定位每个节点的几何信息（形状，位置），由 render process 提供的 DOM tree 和 CSSOM tree ，构建 layout tree （或者说 render tree）。下一步交给 painting process ，主线程遍历 layout tree ，创建画板记录（paint records）。画板记录就像是 painting process 中的一本笔记，记录着类似“先画背景，然后是文字，然后是矩形”的东西。

现在浏览器知道了文档的结构，节点的样式，页面的几何信息，还有绘制顺序。把这些信息转化为屏幕上的像素的过程，叫做光栅化。

比较傻瓜的方法是在浏览器的视口（viewport）里面做部分光栅化。如果用户滚动页面，把这个光栅往下挪，填充空缺的部分。Chrome 刚发布的那会就是用这种方式做光栅化的，现代浏览器则会启用一个更复杂的进程：compositing（合成）。

compositing 进程把页面分为多个图层（layer），分隔地将它们光栅化，最后在合成器线程（compositor thread）中组成页面。图层可能很大，比如占据整个页面的长度，所以合成器线程会把它们分成一个个小瓦片，发送到多个光栅线程（raster threads）。光栅线程将每个瓦片光栅化（像素化），并储存在 GPU 内存中。

> 在 chrome devTools 里面的 performance 可以看到，渲染一个页面的时候有多个 raster threads 参与。

以上大概就是页面 layout --> paint --> composite 的过程，更多细节可以看 google developers 的文章 [
Inside look at modern web browser (part 3)](https://developers.google.com/web/updates/2018/09/inside-browser-part3)

---

好了，终于可以讲浏览器对输入的处理了：

第一个接收到用户手势（触屏，点击，滑动...）的是浏览器进程（UI 线程？），浏览器进程会把这个手势的事件类型，坐标发送到渲染进程。渲染进程找到事件目标，执行对应的监听。

执行 JavaScript 是主线程（main thread）的主要工作，当一个页面经由合成器线程合成后，合成器线程会把绑定了监听事件的区域标记为 “Non-Fast Scrollable Region” 。如果注册的事件在这个区域上触发，合成器线程确保把事件发送到主线程。如果事件发生在这个区域外，合成器线程继续合成新的帧，而不用等待主线程。

我们知道事件代理模式。事件可以冒泡，那么可以在最最底部的元素上绑定事件，比如如下代码：

```javascript
document.body.addEventListener('touchstart',  event => {
    if (event.target === area) {
        event.preventDefault();
    }
} );
```

看起来只需要注册一次事件，就可以代理所有发生在它子代上面的事件，似乎很诱人。但是在浏览器的角度，整个页面被标记为 non-fast scrollable region ，这意味着，尽管你的应用可能不在意页面一些区域的输入，合成器线程还是需要和主线程通信，每次都要等待输入。所以合成器的“顺畅滚动“能力用不上了。

可以向事件监听传入参数 `passive: true` ，这告诉浏览器：你还是想要在主线程上监听事件，但是合成器线程仍继续合成帧。

```javascript
document.body.addEventListener('touchstart',  event => {
    if (event.target === area) {
        event.preventDefault()
    }
 }, {passive: true} );
```

### 事件触发的三个阶段

本节部分转载自： https://yuchengkai.cn/docs/zh/frontend/browser.html#%E4%BA%8B%E4%BB%B6%E6%9C%BA%E5%88%B6

事件触发有三个阶段

 - window 往事件触发处传播，遇到注册的捕获事件会触发
 - 传播到事件触发处时触发注册的事件
 - 从事件触发处往 window 传播，遇到注册的冒泡事件会触发

事件触发一般按照上面的顺序，但是有例外：如果为同一个元素同时注册了捕获事件和冒泡事件，事件会按照注册的顺序触发。

```javascript
// 以下会先打印冒泡然后是捕获
node.addEventListener('click',(event) =>{
    console.log('冒泡')
},false);
node.addEventListener('click',(event) =>{
    console.log('捕获 ')
},true)
```

### 注册事件

一般用 [EventTarget.addEventListener()](https://developer.mozilla.org/zh-CN/docs/Web/API/EventTarget/addEventListener)，当然 ie 老古董自行用 `onEvent = function(){ ... }` 方式。

addEventListener 函数的第三个参数可以是布尔值，也可以是对象。对于布尔值 useCapture 参数来说，该参数默认值为 false 。useCapture 决定了注册的事件是捕获事件还是冒泡事件。缺省情况下注册冒泡事件。

对于对象参数来说，可以使用以下几个属性

 - capture，布尔值，和 useCapture 作用一样
 - once，布尔值，值为 true 表示该回调只会调用一次，调用后会移除监听
 - passive，布尔值，表示永远不会调用 [preventDefault](https://developer.mozilla.org/en-US/docs/Web/API/Event/preventDefault)

capture 和 once 都比较好理解。下面讲一下 preventDefault ：

preventDefault 告诉浏览器：如果此事件没有需要显式处理，那么它默认的动作也不要做（因为默认是要做的）。此事件还是继续传播，除非碰到事件侦听器调用stopPropagation() 或stopImmediatePropagation()，才停止传播。

这里我们可以看到 preventDefault 的作用：开发者可以在监听 touch 和 wheel 事件时指定 `{passive:true}` 来表明他们不会调用preventDefault。也就是说，默认的滚动事件还是要做的，而且永远不会被禁用，从而获得更好的滚动性能。归根结底是合成器线程无需等待主线程，继续合成帧。

更多阅读移步 [Passive event listeners](https://github.com/WICG/EventListenerOptions/blob/gh-pages/explainer.md) [翻译版](https://futu.im/posts/2017-06-06-passive-event-listeners/)

### 事件代理

如果一个节点中的子节点是动态生成的，那么子节点需要注册事件的话应该注册在父节点上

```javascript
<ul id="ul">
    <li>1</li>
    <li>2</li>
    <li>3</li>
    <li>4</li>
    <li>5</li>
</ul>
<script>
    let ul = document.querySelector('##ul')
    ul.addEventListener('click', (event) => {
        console.log(event.target);
    })
</script>
```

事件代理的方式相对于直接给目标注册事件来说，有以下优点

 - 节省内存
 - 不需要给子节点注销事件