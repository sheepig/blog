---
title: css line-height & vertical-align
date: 2019-01-05 21:48:19
tags:
- css
categories: css 基础
---

写样式到时候经常遇到单行文字垂直居中，对于高度已知的块级元素，我们知道一种简单写法：

```css
line-height: 30px;
height: 30px;
```

设定行高，到底改变了什么？

### 字体的度量

先从英文字母开始。刚开始学ABCD的时候，我们都知道练习本上面有四条线。

![](https://upload.wikimedia.org/wikipedia/commons/thumb/3/39/Typography_Line_Terms.svg/410px-Typography_Line_Terms.svg.png)

我们知道一些字母，比如小写的 a、e，是完全写在中间两条线之间的。写大写字母的时候，比如 A，B，它们的最顶部不会顶到第一条线，那样就很难看了——老师一般是这么说的，但是实际上，AB这样的大写字母，本身就不“顶天”。

这里我们先明确两个概念：

 - `Capital Height` 大写字母高度
 - `x-height` 字母 x 的高度

字体的高度和具体字符的高度不是一个概念，英文练习本上面，英文字符大多有上下留白。

现在又有一个问题，字体的高度是怎么算的？我们平时用 px 去设定 font-size ，对不同字体的结果一样吗？

------

传统金属字块，我们可以看到一个字符是摆在一个方格里面的。同种字体，所有字符的方格是一样的，不同字体的区别在于方格的大小，以及字符在方格中上下“留白”大小。

![](https://designwithfontforge.com/en-US/images/MetalTypeZoomIn.JPG)

我们来看字体的度量

 - 字体定义其 [em square](https://designwithfontforge.com/en-US/The_EM_Square.html)，也称为 "em size" 或 "UPM" ，是字符所处的容器。对于 OpenType 字体，这个值通常是 1000 ；对于 TrueType 字体，这个值是 2 的指数，通常设定为 1024 或 2048 。

 - 字体块对外高度由 ascender、descender 和 line gap决定。
 - 具体字符由 capital height、x-height 决定

以 Arial 字体为例，以下是它的参数。

![](font.jpg)

设定 `font-size: 100px` ，`<p></p>`块最终高度是多少？

```html
<p>skdjdAfd</p>
```

![](outcome.png)

结果是 115px ，也就是说 Arial 字体 line-height normal 值为 1.15。计算为 `(ascent + descent + line gap)/em-square` ，line gap（线间距）不是所有字体都有。

ascent+descent区域，我们称为内容区域（content-area）

没有人为设定字体行高的时候，content-area + line-gap（有些字体为 0 ） 为 line-height。

到这里可以确定，开篇提到的那种居中方法，居中的其实是 content-area 。

### line-box

当 `<p>` 元素呈现在屏幕上，它根据它的宽度可以有很多线。每一行是由一个或多个行内元素（HTML标签元素或匿名内联元素文本内容）组成，专业术语称为行盒（line-box）。line-box的高度是基于它的子元素高度的。浏览器为每个行内元素计算的高度都是line-box（子元素的最高点到最低点）。因此line-box的总高度足以包含所有子元素（默认情况下）。


### line-height 的值

line-height normal 的值，依据不同字体去计算，如果是数字或百分比，相对字体的大小计算。改变 line-height 可能是一件危险的事，所有字体都有自己的安全行高，如果 line-height 过小，字符可能超出 line-box 。

![](chop.jpg)



> 参考
[深入了解CSS字体度量，行高和vertical-align](https://www.w3cplus.com/css/css-font-metrics-line-height-and-vertical-align.html)