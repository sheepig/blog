---
title: 从 Babel 转译结果看 ES6
date: 2018-11-20 11:54:52
tags:
- js
- ES6
categories: js 基础
---

## ES6 => ES5

转义结果来自babel

### class

```javascript
class Component {
  constructor(props, context){
    this.props = props;
    this.context = context;
  }
}
```

```javascript
"use strict";

function _classCallCheck(instance, Constructor) {
  if (!(instance instanceof Constructor)) {
		throw new TypeError("Cannot call a class as a function");
	}
}

var Component = function Component(props, context) {
  _classCallCheck(this, Component);

  this.props = props;
  this.context = context;
};
```

通过 class 定义的“类”，不能像 `Component(p, c)` 这样创建一个对象。通过 `this` 向对象添加熟悉，和传统构造函数基本一样。

如果有自定义的函数呢？

```javascript
class Component {
  constructor(props, context){
    this.props = props;
  	this.context = context;
  }
  fn() {
    console.log('fn');
  }
}
```

```javascript
'use strict';

var _createClass = function () {
  function defineProperties(target, props) {
    for (var i = 0; i < props.length; i++) {
      var descriptor = props[i];
      descriptor.enumerable = descriptor.enumerable || false;
      descriptor.configurable = true;
      if ("value" in descriptor) descriptor.writable = true;
      Object.defineProperty(target, descriptor.key, descriptor);
    }
  }
  return function (Constructor, protoProps, staticProps) {
    if (protoProps) defineProperties(Constructor.prototype, protoProps);
    if (staticProps) defineProperties(Constructor, staticProps);
    return Constructor;
  };
}();

function _classCallCheck(instance, Constructor) { if (!(instance instanceof Constructor)) { throw new TypeError("Cannot call a class as a function"); } }

var Component = function () {
  function Component(props, context) {
    _classCallCheck(this, Component);

    this.props = props;
    this.context = context;
  }

  _createClass(Component, [{
    key: 'fn',
    value: function fn() {
      console.log('fn');
    }
  }]);

  return Component;
}();
```

Component 变成了一个自执行匿名函数。在这个函数里面，创建一个同名的构造函数 `Component` ，最后返回它。属性的创建方法同前一个例子一样，方法则是通过 `_createClass` 方法创建。

我们把 `Component` 构造函数和一个携带 `fn` 信息的对象数组传给了 `_createClass` 自执行函数（闭包），最后的结果就是通过 `Object.defineProperty` ，把函数（们）挂在了 `Component` 的原型对象（`Component.prototype`）上。class 声明的“类”中的函数，是挂在原型对象上的，由所有实例共享。

```javascript
var c1 = new Component();
var c2 = new Component();

console.log(c1.fn === c2.fn); // true
```

### class extends...

```javascript
class App extends Component {
  constructor(props) {
    super();
    this.props = props;
  }
  fn() {
    console.log('fn');
  }
}
```

```javascript
'use strict';

var _createClass = function () { function defineProperties(target, props) { for (var i = 0; i < props.length; i++) { var descriptor = props[i]; descriptor.enumerable = descriptor.enumerable || false; descriptor.configurable = true; if ("value" in descriptor) descriptor.writable = true; Object.defineProperty(target, descriptor.key, descriptor); } } return function (Constructor, protoProps, staticProps) { if (protoProps) defineProperties(Constructor.prototype, protoProps); if (staticProps) defineProperties(Constructor, staticProps); return Constructor; }; }();

function _classCallCheck(instance, Constructor) { if (!(instance instanceof Constructor)) { throw new TypeError("Cannot call a class as a function"); } }

function _possibleConstructorReturn(self, call) { if (!self) { throw new ReferenceError("this hasn't been initialised - super() hasn't been called"); } return call && (typeof call === "object" || typeof call === "function") ? call : self; }

function _inherits(subClass, superClass) { 
  if (typeof superClass !== "function" && superClass !== null) { 
    throw new TypeError("Super expression must either be null or a function, not " + typeof superClass); 
  } 
  subClass.prototype = Object.create(superClass && superClass.prototype, { 
    constructor: { 
      value: subClass, 
      enumerable: false, 
      writable: true, 
      configurable: true 
    }
  }); 
  if (superClass) Object.setPrototypeOf ? Object.setPrototypeOf(subClass, superClass) : subClass.__proto__ = superClass; 
}

var App = function (_Component) {
  _inherits(App, _Component);

  function App(props) {
    _classCallCheck(this, App);

    var _this = _possibleConstructorReturn(this, (App.__proto__ || Object.getPrototypeOf(App)).call(this));

    _this.props = props;
    return _this;
  }

  _createClass(App, [{
    key: 'fn',
    value: function fn() {
      console.log('fn');
    }
  }]);

  return App;
}(Component);
```

`App` 是一个自执行函数，父类 `Component` 作为参数被传入。 `_inherits` 方法，子类 `subClass.prototype` 继承 `superClass.prototype` 。App 构造函数处理继承父类属性，最后返回 App 构造函数。

### rest 参数

```javascript
function push(array, ...items) {
  items.forEach(function(item) {
    array.push(item);
    console.log(item);
  });
}

var a = [];
push(a, 1, 2, 3)
```

rest 参数使得我们可以传入多余参数，把他们当数组一样处理。（实际上在内部，他们的确被转化成一个数组）。这样就可以不用 arguments 参数。arguments 参数是类数组对象，除了 length 属性和索引元素之外没有任何 Array 属性。

```javascript
"use strict";

function push(array) {
  for (var _len = arguments.length, items = Array(_len > 1 ? _len - 1 : 0), _key = 1; _key < _len; _key++) {
    items[_key - 1] = arguments[_key];
  }

  items.forEach(function (item) {
    array.push(item);
    console.log(item);
  });
}

var a = [];
push(a, 1, 2, 3);
```