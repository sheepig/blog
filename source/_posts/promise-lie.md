---
title: 解析 Lie 库 Promise 源码
date: 2018-11-20 11:56:44
tags:
- js
- 异步编程
categories:
- 源码解析
---

## Promise

### Promise/A+ 

以下内容译自 Promise/A+ 规范。

Promise 代表一个异步操作的最终结果。操作一个 promise 的主要方式是通过它的 `then` 方法，该方法接收一个回调，回调参数是 promise 的最终值或这个 promise 转为 fulfilled 失败的原因。

#### 术语

 - “promies” 是一个对象或函数，并有一个符合规范的 `then` 方法
 - “thenable” 是一个定义 `then` 方法的对象或函数
 - “value” 是任何合法的 JavaScript 值（包括 `undefined` ，一个 thenable ，或一个 promise）
 - “exception” 是用 `throw` 声明抛出的值
 - “reason” 指明一个 promise 为什么被 rejected 。

#### 条件

##### Promise 的状态

一个 promise 只能处于三种状态中的一种：`pending`, `fulfilled` 或 `rejected` 。

当 promise 处于 pending 时：

 - 可能转换为 fulfilled、 rejected 之一

当 promise 处于 fulfilled 时：

 - 不可转换为其他状态
 - 必须有一不可更改的 value

当 promise 处于 rejected 时：

 - 不可转换为其他状态
 - 必须有一个不可更改的 reason

这里所说的“不可更改”，指的是自身不变性（比如 `===` ），但不需要遵循深层不可变性。

------

##### `then` 方法

promise 必须提供一个 `then` 方法，用以访问当前的/最终的 value 或 reason 。

`then` 方法接收两个参数：

```javascript
promise.then(onFulfilled, OnRejected)
```

`onFulfilled` 和 `onRejected` 都是可选参数

 - 如果 `onFulfilled` 不是函数，忽略
 - 如果 `onRejected` 不是函数，忽略

如果 `onFulfulled` 是函数

 - 它必须在 `promise` 转换为 fulfilled 之后调用，第一个参数是 `promise` 的 value
 - 不可以在 `promise` 转换为 fulfilled 之前调用
 - 不可以被调用一次以上

如果 `onRejected` 是函数

 - 它必须在 `promise` 转换为 rejected 之后调用，第一个参数是 `promise` 的 reason
 - 不可以在 `promise` 转换为 rejected 之前调用
 - 不可以被调用一次以上

当且仅的[执行上下文](https://es5.github.io/#x10.3)栈只包含[平台代码](https://promisesaplus.com/#notes) `onFulfilled` 和 `onRejected` 被调用

`onFulfilled` 和 `onRejected` 必须以函数的形式调用（不能有 `this` ，严格模式下 `this` 为 `undefined` ，非严格模式下 `this` 指向 `global` 对象）。

`then` 可能在同个 promise 上多次调用

 - 如果/当 promise 转换为 fulfilled ，所有对应的 `onFulfilled` 回调会按照 `then` 发起的顺序调用。
 - 如果/当 promise 转换为 rejected ，所有对应的 `onRejected` 回调会按照 `then` 发起的顺序调用。

`then` 必须返回一个 promise

```javascript
promise2 = promise1.then(onFulfilled, onRejected);
```

 - `onFulfilled` 或 `onRejected` 之一返回一个 value ，则执行 Promise Resolution Procedure `[[Resolve]](promise2, x)` 
 - `onFulfilled` 或 `onRejected` 之一抛出一个异常 `e` ，`promise2` 必须被 rejected ，并且将  `e` 作为 reason 传入
 - 如果 `onFulfilled` 不是一个函数而且 `promise1` 转换为 fulfilled ，`promise2` 必须转换为 fulfilled ，并且将 `promise1` 的 value 传入。
 - 如果 `onRejected` 不是一个函数而且 `promise1` 转换为 rejected ，`promise2` 必须转换为 rejected ，并且将 `promise1` 的 reason 传入。

------

##### Promise Resolution Procedure

**Promise Resolution Procedure** 是一个抽象的操作，输入是一个 promise 和一个 value ， 我们用 `[[Resolve]](promise, x)` 表示，如果 `x` 是 thenable 的，它做此尝试：在 `x` 表现得像一个 promise 的前提下，让  `promise` 采用 `x` 的状态( `state` )。否则将 `promise` 的状态转换为 fulfilled ，`x` 作为 value 。

执行 `[[Resolve]](promise, x)` ，会执行以下步骤：

如果 `promise` 和 `x` 指向同个对象，以一个 `TypeError` 为 reason ，拒绝这个 `promise` 。

1. 如果 `x` 是一个 promise ，采用它的 state

 - 如果 `x` 处于 pending 中，`promise` 必须保持 pending 直至 `x` 转换为 fulfilled 或 rejected 
 - 如果/当 `x` 转换为 fulfilled ，以相同的 value ，履行这个 `promise` （fulfill `promise` with the same value）
 - 如果/当 `x` 转换为 rejected ，以相同的 reason ，拒绝这个 `promise` （reject `promise` with the same reason）


2. 如果 x 是一个对象或者函数

 - 令 `then` = `x.then`
 - 如果访问 `x.then` 会抛出一个异常 `e` ，以 `e` 为 reason 拒绝这个 promise 
 - 如果 then 是一个函数，将它的 `this` 绑定为 `x` 并调用它，第一个参数是 `resolvePromise` ，第二个参数是 `rejectedPromise` 。

```javascript
then.call(x, resolvePromise, rejectedPromise)
```
如果 `resolvePromise` 调用，调用返回值是 `y` ，执行 `[[Resolve]](promise, y)`

如果 `rejectedPromise` 调用，调用返回值是 `r` ，以 `r` 为由拒绝 `promise` 

如果 `resolvePromise` `rejectedPromise` 都被调用，或者以相同参数多次调用，则只采用第一次调用，忽略其他调用

如果调用 then 抛出一个异常 e ：

如果 `resolvePromise` `rejectedPromise` 都已经被调用，忽略 e 

否则，以 `e` 为 reason 拒绝这个 `promise` 

 - 如果 `then` 不是一个函数，以 `x` 履行这个 `promise` （fulfill promise with x）

3. 如果 x 既不是函数也不是对象，以 `x` 履行这个 `promise` （fulfill promise with x）

### 解析 lie 库的 Promise 实现

#### 状态

Promise 任何实例 promise （接下来用 promise 指代 Promise 实例）在一个时刻，只能是三种状态之一：

```javascript
var REJECTED = ['REJECTED'];
var FULFILLED = ['FULFILLED'];
var PENDING = ['PENDING'];
```

#### Promise 构造函数

```javascript
var p1 = new Promise(function (resolve, reject) {
	console.log(1);
	resolve(4);
	console.log(2)
});
p1.then(num => {
	console.log(num);
})

console.log(3);
```

输出结果是 1234 。入参的匿名函数似乎是立刻执行的（ 1，2 最先被打印），但是 resolve 之后没有立刻 then ，而是先执行完所有同步代码（打印 3 ），再去调用一个回调（打印 4 ）。

看一下 Promise 的构造函数

```javascript
function Promise(resolver) {
  if (typeof resolver !== 'function') {
    throw new TypeError('resolver must be a function');
  }
  this.state = PENDING;
  this.queue = [];
  this.outcome = void 0;
  if (resolver !== INTERNAL) {
    safelyResolveThenable(this, resolver);
  }
}
```

新创建的 promise 的状态，会被初始化为 PENDING 。入参 resolver 就是 `function(resolve, reject) { // ... }` 这样的一个函数，看一下 `safelyResolveThenable(this, resolver)` 做了什么。


```javascript
function safelyResolveThenable(self, thenable) {
  // Either fulfill, reject or reject with error
  var called = false;
  function onError(value) {
    if (called) {
      return;
    }
    called = true;
    handlers.reject(self, value);
  }

  function onSuccess(value) {
    if (called) {
      return;
    }
    called = true;
    handlers.resolve(self, value);
  }

  function tryToUnwrap() {
    thenable(onSuccess, onError);
  }

  var result = tryCatch(tryToUnwrap);
  if (result.status === 'error') {
    onError(result.value);
  }
}
```

这里的 thenable 就是传进来的 resolver 。可以看到 thenable 被调用：`var result = tryCatch(tryToUnwrap);` ，
tryCatch 其实是一个帮助函数，封装了 `func(value)` 的返回结果。

```javascript
function tryCatch(func, value) {
  var out = {};
  try {
    out.value = func(value);
    out.status = 'success';
  } catch (e) {
    out.status = 'error';
    out.value = e;
  }
  return out;
}
```

thenable 执行环境中，以下三个语句按顺序执行。

```javascript
console.log(1);
resolve(4);
console.log(2)
```

重点看一下 `resolve(4)` ，打断点，最终来到 handlers.resolve 

```javascript
handlers.resolve = function (self, value) {
  var result = tryCatch(getThen, value);
  if (result.status === 'error') {
    return handlers.reject(self, result.value);
  }
  var thenable = result.value;

  if (thenable) {
    safelyResolveThenable(self, thenable);
  } else {
    self.state = FULFILLED;
    self.outcome = value;
    var i = -1;
    var len = self.queue.length;
    while (++i < len) {
      self.queue[i].callFulfilled(value);
    }
  }
  return self;
};
```

这里 `self = p1` ，`value = 4` ，value 不是一个 thenable 类型（如果是 thenable 的处理？）。最后把 self 的 state 设为 FULFILLED ，把 outcome 赋值为 value ，返回 self 。

其实执行完 `resolve(4)` ，p1 的状态已经变为 FULFILLED ，并且有一个不可变的 value 。

#### Promise.prototype.then

then 方法的最主要作用，是访问一个 promise 最终的 value，或者 reason 。

```javascript
Promise.prototype.then = function (onFulfilled, onRejected) {
  // 如果 onFulfilled, onRejected 不是函数，忽略，同时实现透传，即
  // Promise.then().then(num => {console.log(num);})
  if (typeof onFulfilled !== 'function' && this.state === FULFILLED ||
    typeof onRejected !== 'function' && this.state === REJECTED) {
    return this;
  }

  // 返回结果是一个 promise ，为了和当前的 promise（即 this） 区分，命名为 promise2
  var promise2 = new this.constructor(INTERNAL);
  if (this.state !== PENDING) {
  	// 只有 promise 的 state 不是 PENDING 时，才会调用 onFulfilled 或者 onRejected
  	// 当状态为 FULFILLED 时，调用 onFulfilled ，并把 promise 的 value 传递给 onFulfilled
  	// 当状态为 REJECTED 时，调用 onRejected ，并把 promise 的 reason 传递给 onRejected
    var resolver = this.state === FULFILLED ? onFulfilled : onRejected;
    unwrap(promise2, resolver, this.outcome);
  } else {
  	// promise 的 state 为 PENDING ，把 promise2，onFulfilled 和 onRejected 包装成一个 QueueItem，
  	// 压入 promise 的 queue
    this.queue.push(new QueueItem(promise2, onFulfilled, onRejected));
  }
  return promise;
}
```

#### nextTick 实现

then 方法有链式传递，前一个 promise 的 value 或 reason 会传递给下一个 promise ， `promise.then(onFulfilled, onRejected)` ，真正在操作 value 和 reason 的其实是新的 promise（promise2）。

```javascript
function unwrap(promise, func, value) {
  immediate(function () {
    var returnValue;
    try {
      // 新的 promise 的状态是变为 FULFILLED 还是 REJECTED，和它处理的是 value 还是 reason 没有必然联系，
      // 也就是说它的状态转换不受前一个 promise 的状态影响，它只根据前一个 promise 提供的 value/reason，分别调用自己不同的处理方法
      // 如果处理成功，比如成功处理前一个 promise 传过来的 reason ，那么新的 promise 会被 resolve
      // 否则 reject
      returnValue = func(value);
    } catch (e) {
      return handlers.reject(promise, e);
    }
    if (returnValue === promise) {
      handlers.reject(promise, new TypeError('Cannot resolve promise with itself'));
    } else {
      handlers.resolve(promise, returnValue);
    }
  });
}
```

`immediate` 注册一个异步方法，使得：当且仅当执行上下文只有平台代码的时候，执行 then 的回调 onFulfilled 或 onRejected 。类似 setTimeout 0 ，但又不仅仅如此。

```javascript
function immediate(task) {
  if (queue.push(task) === 1 && !draining) {
    scheduleDrain();
  }
}
```

其中 `queue` `draining` 是闭包中可以访问的变量。queue 用以保存回调，以便按序执行，draining 标记当前是否有回调执行。

```javascript
var draining;
var queue = [];
```

只有在当前仅有一个任务 task ，并且之前的任务已经全部执行完毕（draining === false），才会调用 `scheduleDrain()` 。

```javascript
var Mutation = global.MutationObserver || global.WebKitMutationObserver;

{
  // 浏览器环境 IE11+ 支持，使用 MutationObserver
  if (Mutation) {
    var called = 0;
    var observer = new Mutation(nextTick);
    var element = global.document.createTextNode('');
    observer.observe(element, {
      characterData: true
    });
    scheduleDrain = function () {
      element.data = (called = ++called % 2); // called 的值在 0/1 之间切换
    };
  // Node.js 环境，使用 MessageChannel
  } else if (!global.setImmediate && typeof global.MessageChannel !== 'undefined') {
    var channel = new global.MessageChannel();
    channel.port1.onmessage = nextTick;
    scheduleDrain = function () {
      channel.port2.postMessage(0);
    };
  // 低版本浏览器，使用一个 script 标签，当它插入文档中，它的 readystatechange 事件会触发
  } else if ('document' in global && 'onreadystatechange' in global.document.createElement('script')) {
    scheduleDrain = function () {

      // Create a <script> element; its readystatechange event will be fired asynchronously once it is inserted
      // into the document. Do so, thus queuing up the task. Remember to clean up once it's been called.
      var scriptEl = global.document.createElement('script');
      scriptEl.onreadystatechange = function () {
        nextTick();

        scriptEl.onreadystatechange = null;
        scriptEl.parentNode.removeChild(scriptEl);
        scriptEl = null;
      };
      global.document.documentElement.appendChild(scriptEl);
    };
  // 最后什么都不行，用 setTimeout 0
  } else {
    scheduleDrain = function () {
      setTimeout(nextTick, 0);
    };
  }
}
```

scheduleDrain 的根本目的，是尽可能较快，异步触发 nextTick 。以下是几种方案：

使用 [MutationObserver](https://developer.mozilla.org/zh-CN/docs/Web/API/MutationObserver) 监控一个空文本节点，每次调用 scheduleDrain 之后会把 called 的值从 0 改为 1（或从 1 改回 0）。改变这个 DOM 节点会触发
`new Mutation(nextTick)` 注册的回调。这种回调的执行顺序会比 setTimeout 快。

使用[MessageChannel](https://developer.mozilla.org/zh-CN/docs/Web/API/MessageChannel)，port1 的 message 事件触发时候，调用 nextTick 回调。scheduleDrain 执行时，通过 `port2.postMessage` 触发 port1 的 message 事件。

低版本浏览器用一个动态插入的 script 标签，它的 readystatechange 事件会触发，触发的结果就是立即执行 nextTick 。这种“事件触发调用 nextTick” 的思想和前面两种方法很接近，但是用 script 标签要注意，在 readystatechange 事件最后要移除 script 标签。

最后，如果以上三种方案都不支持，用 setTimeout 0 。


```javascript
function nextTick() {
  draining = true;
  var i, oldQueue;
  var len = queue.length;
  while (len) {
    oldQueue = queue;
    queue = [];
    i = -1;
    while (++i < len) {
      oldQueue[i]();
    }
    len = queue.length;
  }
  draining = false;
}
```

nextTick 按 queue 中 回调的顺序，逐个执行回调，执行完之后把 draining 重置为 false 。

看一下这段代码的执行结果：

```javascript
var p1 = new Promise(function (resolve, reject) {
	console.log(1);
	resolve(4);
	Promise.resolve(3).then( num => { console.log(num); });
}).then().then(num => {
	console.log(num);
})

console.log(2);
```

结果是 1234 。细节如下：

1. Promise 的入参是同步执行的，所以先打印 1 。

2. p1 先 resolve ，p1 状态 FULFILLED ，outcome = 4；

3. 新建一个 Promise 实例，立刻把它的状态变为 RESOLVED ，注册 then 的成功回调。

4. 注册 p1 的回调

5. 执行同步代码

6. 清空 queue 。

#### 异步 resolver

上面例子的中，调用 `resolve` 的时候是同步的，如果调用 `resolve` 是异步的，那么在 `then` 方法中，promise 的状态仍为 PENDING ，这时候 promise.queue 就派上用场了。

![](resolver.png)

只是把异步 resolver 和 promise 的信息包装成一个 QueueItem ，压入 promise.queue 。

定时器结束后，promise 的状态才变成 RESOLVED ，此时才注册 then 回调。

![](async-resolver.png)

### Promise 的其他接口

Promise A+ 其实没有规定 Promise 实现的其他接口。常用的 `promise.catch` `Promise.resolve` 其实都是基于 `then` 去扩展的。

#### Promise.prototype.catch

```javascript
Promise.prototype["catch"] = function (onRejected) {
  return this.then(null, onRejected);
};
```
 
所以 `Promise.prototype.catch` 是 `this.then(null, onRejected)` 的别名。以下这两种写法本质是一样的。

```javascript
var p1 = new Promise((resolve, reject) => {
    setTimeout(() => {
        reject('opps!');
    }, 1000)
});

p1.catch((err) => {
    console.log(err);
})

// 等同于
p1.then(() => {}, (err) => {
    console.log(err);
})

```

#### Promise.prototype.finally

finally 方法用于指定不管 Promise 对象最后状态如何，都会执行的操作。该方法是 ES2018 引入标准的。

finally 其实是 `then` 方法的特例，`onFulfilled` `onRejected` 的回调是同一个。

```javascript
Promise.prototype["finally"] = function (callback) {
  if (typeof callback !== 'function') {
    return this;
  }
  var p = this.constructor;
  // finally 本质还是 then 方法
  return this.then(resolve, reject);

  function resolve(value) {
    function yes () {
      return value;
    }
    // 利用 Promise.resolve 把一个新的 promise 转换成 RESOLVED，注册它的成功回调
    // 这里调用 callback 时，和 `this` 没有联系。callback 的操作，应该是与状态无关的，不依赖于 Promise 的执行结果。
    return p.resolve(callback()).then(yes);
  }
  function reject(reason) {
    function no () {
      throw reason;
    }
    return p.resolve(callback()).then(no);
  }
};
```

#### Promise.resolve && Promise.reject

这两个方法比较接近，所以放在一块讲。Promise.resolve 多了一层判断，如果入参已经是一个 promise ，直接返回它。这两个方法本质都是新建一个 promise ，并且一定在未来某一时刻（可能是立刻），把它的状态转换为 FULFILLED 或 REJECTED 。

```javascript
Promise.resolve = resolve;
function resolve(value) {
  if (value instanceof this) {
    return value;
  }
  return handlers.resolve(new this(INTERNAL), value);
}

Promise.reject = reject;
function reject(reason) {
  var promise = new this(INTERNAL);
  return handlers.reject(promise, reason);
}
```

在这里有必要重新看看 `handlers.resolve` 方法。`handlers.resolve(new this(INTERNAL), value)` 的 value 可以是一个 thenable 对象。即

```javascript
let thenable = {
    then: function(resolve, reject) {
        resolve(2)
    }
}
Promise.resolve(thenable).then((v => {
    console.log(v)
}));
```

这里 resolve 是同步的，promise 创建后，会调用 then 方法。

Promise.resolve 并不是总返回一个 resolved 对象。在 resolve 的时候其实用了 try catch 封装。catch 错误的时候会返回一个 rejected 的 promise 。

#### Promise.all && Promise.race

Promise.all 返回的是一个 promise 。

```javascript
Promise.all = all;
function all(iterable) {
  var self = this;
  if (Object.prototype.toString.call(iterable) !== '[object Array]') {
    return this.reject(new TypeError('must be an array'));
  }

  var len = iterable.length;
  var called = false;
  if (!len) {
    return this.resolve([]);
  }

  // values 数组，每个值是 iterable[i] resolve 的返回值。这个数组传递给 promise 的回调函数
  var values = new Array(len);
  var resolved = 0;
  var i = -1;
  // Promise.all 返回值是一个 promise
  var promise = new this(INTERNAL);

  while (++i < len) {
    allResolver(iterable[i], i);
  }
  return promise;
  function allResolver(value, i) {
    // n 次 Promise.resolve(iterable[i])
    // 第 i 次 Promise.resolve(iterable[i])，因为 iterable[i] resolve，把它的返回值赋给 values[i] 
    // 等到所有 iterable[i] 都被 resolved ，也就是符合条件 ++resolved === len && !called
    // 把 values 数组传递给 promise 回调
    self.resolve(value).then(resolveFromAll, function (error) {
      if (!called) {
        // 一旦有一个 iterable[i] 被rejected ，reject promise，并把 iterable[i] rejected 的 reason 传递过去
        called = true;
        handlers.reject(promise, error);
      }
    });
    // 如果 iterable[i]
    function resolveFromAll(outValue) {
      values[i] = outValue;
      if (++resolved === len && !called) {
        called = true;
        handlers.resolve(promise, values);
      }
    }
  }
}
```

Promise.race 也会做 n 次 `Promise.resolve(iterable[i])` ，不同的是，第一个 `iterable[i]` 状态改变就会停止后面的 Promise.resolve ，并把 resolve/reject 结果传给 promise 。

>参考

>[从一道Promise执行顺序的题目看Promise实现](https://fed.renren.com/2018/03/10/promise/)
>
>[InterviewMap-Promise 实现](https://yuchengkai.cn/docs/zh/frontend/#promise-%E5%AE%9E%E7%8E%B0)
>
>[Promises/A+](https://promisesaplus.com/)
>
>[详解JavaScript中的Event Loop（事件循环）机制](https://zhuanlan.zhihu.com/p/33058983)