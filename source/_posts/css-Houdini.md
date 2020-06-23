---
title: CSS Houdini
date: 2018-11-21 00:10:49
tags:
- css
- 浏览器
categories: css 进阶
---

借用[谷歌开发者文档](https://developers.google.com/web/updates/2016/05/houdini)的比喻，CSS Houdini 允许你自己当一名魔术师——传统流程中，你改变了 css 的一个样式，可能整个页面的样式都大变样，但是你只能看着浏览器呈现这场魔法盛宴，自己无法参与。CSS Houdini 赋予了你魔术师的本领，你也可以参与浏览器处理样式和布局的流程。

Houdini 是 W3C 新成立的一个任务小组，它的终极目标是实现 css 属性的完全兼容。Houdini 提出了一个前无古人的的设想：开放 CSS 的 API 给开发者，开发者可以通过这套接口自行扩展 CSS，并提供相应的工具允许开发者介入浏览器渲染引擎的样式和布局流程中。

### 规范

#### Worklets

此规范定义了一个 API ，规范了渲染流程中的各个阶段运行的独立于主 javascript 执行环境的脚本。Worklets 的概念和 Web Worker 相似，它们允许你引入脚本文件并执行特定的 JS 代码。Worklets 和 Web Worker 有很多概念上的重叠，那么搞出一个 Worklets 是否多此一举？

Worklets ，后缀 `-let` 已经表面了它的特性“小”。存在这种可能：脚本中某些代码片段，在每一帧都要跑一次。Web Worker 其实比较笨重，并不鼓励大量使用。举个例子，为一张四百万像素的图片的每一个像素启用一个 Web Worker 将是不妥当的。

Worklets 定义了一个方法集合，这些方法的特性由 Worklets 的类型预定义了，开发者所能执行的操作类型被限定，这样就保证了性能。

#### CSS Paint API

Paint API 在 Chrome 65 默认开启支持，它也被称为 “Houdini’s paint worklet”，它允许 Web 开发人员使用 JavaScript 自定义 CSS `image`（背景，边框，内容），自定义的 `image` 会响应样式和尺寸的变化。 

下面是[谷歌开发者文档](https://developers.google.com/web/updates/2018/01/paintapi)给的一个例子。

在 Worklets 脚本文件中，先定义一个 CheckerboardPainter 的类，里面规定了画棋盘的方法。然后用 `registerPaint` 方法注册一个 paint worklet 类。这个 paint worklet 名为 checkerboard 。

```javascript
// checkerboard.js
class CheckerboardPainter {
  paint(ctx, geom, properties) {
    // Use `ctx` as if it was a normal canvas
    const colors = ['red', 'green', 'blue'];
    const size = 32;
    for(let y = 0; y < geom.height/size; y++) {
      for(let x = 0; x < geom.width/size; x++) {
        const color = colors[(x + y) % colors.length];
        ctx.beginPath();
        ctx.fillStyle = color;
        ctx.rect(x * size, y * size, size, size);
        ctx.fill();
      }
    }
  }
}

// Register our class under a specific name
registerPaint('checkerboard', CheckerboardPainter);
```

在主 javaScript 环境中，用 `CSS.paintWorklet.addModule('checkerboard.js')` 加载这个 paint worklet ，然后在 css 属性值里面，可以调用 `paint(checkerboard)` ，就可以定义自己要的 css 背景/边框/内容 效果。

```javascript
<!-- index.html -->
<!doctype html>
<style>
  textarea {
    background-image: paint(checkerboard);
  }
</style>
<textarea></textarea>
<script>
  CSS.paintWorklet.addModule('checkerboard.js');
</script>
```

[在线 demo](https://googlechromelabs.github.io/houdini-samples/paint-worklet/checkerboard/)

#### Animation Worklet

Animation Worklet 是 Compositor Worklet 演化而来的。详细的资料看[这里](https://dassur.ma/things/animworklet/)。

Compositor Worklet 允许开发者书写在合成器线程（compositor thread）。这将保证代码在每一帧运行，并且开启了一些新的可能性，例如强制实现一些和不是绑定在时间上的动画，比如输入或滚动位置。

如果合成器线程阻塞了——因为代码计算量太大或者低效——那么整个页面将无法响应，动画也会卡住。

Animation Worklet 不运行在合成器线程“上”，而是在“尽力而为”的基础上与它同步运行。这意味着，如果你在你的 worklet 中堵塞了，动画允许“溜走”。

预计 Chrome 71 首次尝试支持 Animation Worklet，我写这篇文的时候，Chrome 71还未发布。Animation Worklet 的例子可以看[
Houdini's Animation Worklet](https://developers.google.com/web/updates/2018/10/animation-worklet)。有 [polyfill](https://github.com/web-animations/web-animations-js) 提供一样的 API，但不提供一样的性能。

#### Layout Worklet

开发者可以自己定义一个布局，自定义盒子内元素的布局，然后用类似 `display: layout(myLayout)` 这样的规则应用自己的布局。这样的好处是：一方面当有新的布局出现的时候可以借助这个 API 进行 polyfill 就不用担心没有实现的浏览器不兼容，另一方面可以发挥想象力实现自己想要的布局，这样在布局上可能会百花齐放了，而不仅仅使用W3C给的那几种布局。

#### Typed CSSOM

在 js 中修改一个 css 属性，可谓要翻山越岭：

```javascript
$('#someDiv').style.height = getRandomInt() + 'px';
```

我们做了数学计算，把一个数字转成字符串并附加一个单位，浏览器再解析这个字符串，得到一个数字，再把这个数字交回给 CSS 引擎。

有了类型化 CSS 对象模型，我们可以基于 StylePropertyMap 操作元素。

```javascript
<div style="width: 200px;" id="div1"></div>
<div style="width: 300px;" id="div2"></div>
<div id="div3"></div>
<div style="margin-left: calc(5em + 50%);" id="div4"></div>
var w1 = $('#div1').styleMap.get('width');
var w2 = $('#div2').styleMap.get('width');
$('#div3').styleMap.set('background-size',
  [new SimpleLength(200, 'px'), w1.add(w2)])
$('#div4')).styleMap.get('margin-left')
  // => {em: 5, percent: 50}
```

> 参考文章
> [Houdini：CSS 领域最令人振奋的革新
](https://zhuanlan.zhihu.com/p/20939640)
> [Houdini: Demystifying CSS](https://developers.google.com/web/updates/2016/05/houdini)
>