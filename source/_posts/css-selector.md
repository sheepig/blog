---
title: css 选择器优先级计算
date: 2018-11-20 11:06:59
tags: 
- css
- 优化
categories: css 基础
---

## css 选择器

> 主要内容：
> css 选择器优先级，一些优化。

### 样式优先级

#### 样式规则来源

 - 外部样式表或样式元素中的 CSS 规则

```javascript
p {color:blue}
```

 - inline 样式属性及类似内容

```javascript
<p style="color:blue" />
```

 - HTML 可视化属性（映射到相关的样式规则）

```javascript
<p bgcolor="blue" />
```

#### 样式表层叠顺序（优先级从低到高）

1. 浏览器声明
2. 用户普通声明
3. 作者普通声明
4. 作者重要声明
5. 用户重要声明

#### CSS 特异性

一个（或一组）选择器规则的优先级计算。用四个数字表示 `a-b-c-d` 。

 - 如果声明来自于 `style` 属性，而不是带有选择器的规则，则记为 1，否则记为 0 (= a)
 - 计算选择器中 ID 属性（ID attributes，例 `#example` ）的个数 (= b)
 - 计算选择器中其他属性（other attributes，包括属性选择器 attributes selectors， 例 `[type="radio"]` ；类选择器 class selector ，例 `.example`）和伪类（pseudo-classes，例 `:hover`）的个数 (= c)
 - 计算选择器中元素名称（element name，例 `h1`）和伪元素（pseudo-elements，例 `::before`）的个数 (= d)

粗略看，优先级（从高到低）大致如下：

1. !important
2. style
3. id 选择器
4. 类选择器（`.clas`），属性选择器（`[type="radio"]`），伪类选择器（`:hover`）
5. 类型选择器（`h1`），伪元素（`::before`）

多组选择器组合，可以计算其最终优先级 `a-b-c-d` 

```javascript
*             {}  /* a=0 b=0 c=0 d=0 -> specificity = 0,0,0,0 */
li            {}  /* a=0 b=0 c=0 d=1 -> specificity = 0,0,0,1 */
li:first-line {}  /* a=0 b=0 c=0 d=2 -> specificity = 0,0,0,2 */
ul li         {}  /* a=0 b=0 c=0 d=2 -> specificity = 0,0,0,2 */
ul ol+li      {}  /* a=0 b=0 c=0 d=3 -> specificity = 0,0,0,3 */
h1 + *[rel=up]{}  /* a=0 b=0 c=1 d=1 -> specificity = 0,0,1,1 */
ul ol li.red  {}  /* a=0 b=0 c=1 d=3 -> specificity = 0,0,1,3 */
li.red.level  {}  /* a=0 b=0 c=2 d=1 -> specificity = 0,0,2,1 */
#x34y         {}  /* a=0 b=1 c=0 d=0 -> specificity = 0,1,0,0 */
style=""          /* a=1 b=0 c=0 d=0 -> specificity = 1,0,0,0 */
```

比如，有以下规则：

```javascript
li {
    background: #333;
    font-size: 30px;
}
ul li {
    background: #ccc;
}
```

`ul li` 的特异性要优先于 `li` ，所以冲突的属性 `background` 由 `ul li` 定义。最终页面的 li 元素的 `background` 值为 `#ccc` ，`font-size` 值为 `30px` 。

### 样式书写顺序

规范的书写顺序更易于后期的维护。常见书写顺序有这几种：

 - 按首字母排序
 - 按 `property: value` 长度
 - 按 `Outside In` 规则【推荐】

#### Outside In

按一个 CSS 属性对它周边的元素和自身的影响程度，由大到小排序。

```javascript
/* SASS INHERITANCE */
@extend
@mixin, e.g. clearfix

/* GENERATED CONTENT */
content

/* POSITION AND LAYOUT */
position
z-index
top
bottom
left
right

/* Flexbox properties */
float
clear

/* DISPLAY AND VISIBILITY */
display
opacity
transform

/* CLIPPING */
overflow
clip

/* ANIMATION */
animation
transition

/* BOX MODEL (FROM OUTSIDE IN) */
margin
box-shadow
border
border-radius
box-sizing
width
height
padding

/* BACKGROUND */
background
cursor

/* TYPOGRAPHY */
font-size
line-height
font-family
font-weight
font-style
text-align
text-transform
word-spacing
color

/* PSEUDO-CLASSES & PSEUDO-ELEMENTS (NESTED RULES) */
:hover
:focus
:active
:before
:after
:first-child
:last-child
```

### 伪类和伪元素

伪类和伪元素的优先级，跟类和元素的优先级映射是一样的。伪类 > 伪元素。

[css2.1 selector 章节](https://www.w3.org/TR/CSS2/selector.html#pseudo-elements)说明了为什么引入伪类和伪元素。

> CSS introduces the concepts of pseudo-elements and pseudo-classes to permit formatting based on information that lies outside the document tree.

意思就是，引入伪类和伪元素的目的，是格式化文档树以外的信息。

伪类用于当已有元素处于的某个状态时，为其添加对应的样式，这个状态是根据用户行为而动态变化的。比如说，当用户悬停在指定的元素时，我们可以通过:hover来描述这个元素的状态。虽然它和普通的css类相似，可以为已有的元素添加样式，但是它只有处于dom树无法描述的状态下才能为元素添加样式，所以将其称为伪类。

伪元素用于创建一些不在文档树中的元素，并为其添加样式。比如说，我们可以通过:before来在一个元素前增加一些文本，并为这些文本添加样式。虽然用户可以看到这些文本，但是这些文本实际上不在文档树中。关于伪元素使用场景，可以看[Effective前端4：尽可能地使用伪元素](https://www.yinchengli.com/2016/10/30/using-before-after/)


> 参考
> 
> [浏览器工作原理 CSS解析](https://www.html5rocks.com/zh/tutorials/internals/howbrowserswork/#CSS_parsing)
> 
> [Cascading Style Sheets Level 2 Revision 1 (CSS 2.1) Specification](https://www.w3.org/TR/CSS2/cascade.html#specificity)
> 
> [HOW TO ORGANIZE CSS @ 9ELEMENTS](https://9elements.com/css-rule-order/)
> 
> [Effective前端4：尽可能地使用伪元素](https://www.yinchengli.com/2016/10/30/using-before-after/)
> 
> [AlloyTeam——总结伪类与伪元素](http://www.alloyteam.com/2016/05/summary-of-pseudo-classes-and-pseudo-elements/)
