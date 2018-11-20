---
title: async/await 异步队列 vs Promise 异步队列
date: 2018-11-20 12:01:05
tags:
- js
- 异步编程
categories: js 基础
---

我在上一篇[Promise 异步编程](https://www.xqyoung.com/2018/11/20/promise-async-then-chain/)里面，提到了一个面试题：

```javascript
const timeout = ms => new Promise((resolve, reject) => {
    setTimeout(() => {
        resolve();
    }, ms);
});

const ajax1 = () => timeout(2000).then(() => {
    console.log('1');
    return 1;
});

const ajax2 = () => timeout(1000).then(() => {
    console.log('2');
    return 2;
});

const ajax3 = () => timeout(2000).then(() => {
    console.log('3');
    return 3;
});



function mergePromise(ajaxArray) {
//todo 补全函数
}

mergePromise([ajax1, ajax2, ajax3]).then(data => {
    console.log('done');
    console.log(data); // data 为 [1, 2, 3]
});

// 分别输出
// 1
// 2
// 3
// done
// [1, 2, 3]
```

实现方法，我用了 promise ：

```javascript
function mergePromise(ajaxArray) {
  let result = [];
  let p = Promise.resolve();
  ajaxArray.forEach((ajax) => {
    p = p.then(ajax).then((data) => {
      result.push(data)
    })
  });
  return p.then(() => {
    return result;
  })
}
```

为了保持异步队列按顺序调用，我们要维护一条 promise 链，一般就是在函数作用域内生成一个 resolved 的 `Promise.resolve()`，然后在它后面接上 then 链。于是我们还要分析，then 的回调是不是返回一个异步 resolve 的 promise ？后面需不需要再接上一条 then 的尾巴，以接收异步 resolve 的结果？

promise 不好读，因为 promise 基本是 try、catch 和 return 之类的语言原语的替代品。

我用 async/await 改写一下：

```javascript
function mergePromise(ajaxArray) {
  let result = [];
  async function run() {
    for (let fn of ajaxArray) {
      let data = await fn();
      result.push(data);
    }
  }
  return run().then(() => {
    return result;
  });
}
```

在循环里面，我们可以像写同步代码一样写异步代码，await 一个异步结果。

### generator 函数

[生成器函数](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/function*)在执行时能暂停，后面又能从暂停处继续执行。

调用一个生成器函数并不会马上执行它里面的语句，而是返回一个这个生成器的 迭代器 （iterator ）对象。当这个迭代器的 next() 方法被首次（后续）调用时，其内的语句会执行到第一个（后续）出现yield的位置为止，yield 后紧跟迭代器要返回的值。或者如果用的是 yield*（多了个星号），则表示将执行权移交给另一个生成器函数（当前生成器暂停执行）。

```javascript
function *gen(){
    yield 10;
    y=yield 'foo';
    yield y;
}

var g=gen();
console.log(g.next());// 执行 yield 10，返回 10
console.log(g.next());// 执行 yield 'foo'，返回 'foo'
console.log(g.next(10));// 将 10 赋给上一条 yield 'foo' 的左值，即执行 y=10，返回 10
console.log(g.next());// 执行完毕，value 为 undefined，done 为 true
```

更多例子查看 [MDN 生成器函数](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/function*)

#### generator 异步队列

以上面的代码为例，每次调用 `g.next()` 会走到下一个 yield 语句，对 yield 语句后面对表达式求值，并且返回形如 `{value: someValue, done: false}` 的对象，我们希望在生成器函数里面可以像写同步代码一样，yield 一个异步结果。那么 yield 应该返回一个 Promise 实例，通过控制 g.next() 的执行，只有在上一个 yield 返回的 promise resolve/reject 之后，才调用 g.next() 走到下一个 yield 语句。

我们先有一系列的 promise ，它们是要被异步执行的。

```javascript
const asyncPromise = (x) => {
  return new Promise(resolve => {
    setTimeout(() => {
      resolve(x + 1);
    }, 1000)
  })
}

function* gen() {
  const data1 = yield asyncPromise(1);
  console.log(data1);

  const data2 = yield asyncPromise(data1);
  console.log(data2);

  const data3 = yield asyncPromise(data2);
  console.log(data3);
}
```

然后我们需要一个控制函数，它接受 gen 作为函数，控制执行流程。

```javascript
const carryPromise = function(gen) {
  return new Promise((resolve, reject) => {
    const g = gen();
    const next = (data) => {
      const ret = g.next(data);
      // 走到最后一个 yield 语句，最好把最后一个值传递下去
      if (ret.done) {
        resolve(data);
        return;
      }
      // ret.value 其实是一个 promise
      // 这里为刚刚生成的 promise 注册 then 回调，回调的操作是注册下一个 promise
      ret.value.then((data) => {
        next(data);
        // 上面这句换成
        // return next(data);
        // 结果也是一样的，就和 Promise 链式调用一样了
        // 只不过实现上，更偏向控制 g.next() 的移动，来实现 promise 的注册
      })
    }
    next();
  })
}
```

看一下执行结果：

```javascript
carryPromise(gen).then((value) => {
  setTimeout(() => {
    console.log(`end with: ${value}`);
  }, 1000)
})

// 每项输出前间隔 1 s
// 2
// 3
// 4
// end with 4
```

用 generator 实现的异步队列，跟 Promise 的 then 链式回调有些相似，一样是在 promise 的 then 回调里面创建新的 promise ，不一样的是，链式回调的 then 回调里面要 return 这个新的 promise，值才会沿着 then 链传递下去。而 generator 赋给了我们从“外部读取异步结果的能力”，例如 `const data1 = yield asyncPromise(1);` 。

### async/await

首先搞清楚，async 关键字后面跟一个函数，await 关键字后面跟一个表达式，表达式应该是一个 Promise 对象或者任何要等待的值。

当调用一个 async 函数时，会返回一个 Promise 对象。当这个 async 函数返回一个值时，Promise 的 resolve 方法会负责传递这个值；当 async 函数抛出异常时，Promise 的 reject 方法也会传递这个异常值。

async 函数中可能会有 await 表达式，这会使 async 函数暂停执行，等待表达式中的 Promise 执行完成后才会继续执行 async 函数并返回解决结果。

更多关于 async function，查看 [MDN async function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)

用 async/await 改写 generator 实现的异步队列：

```javascript
async function fn() {
  const data1 = await asyncPromise(1);
  console.log(data1);

  const data2 = await asyncPromise(data1);
  console.log(data2);

  const data3 = await asyncPromise(data2);
  console.log(data3);
  return data3;
}

var p = fn().then((value) => {
  setTimeout(() => {
    console.log(`end with ${value}`)
  }, 1000);
});
```

async function 的返回值将被隐式地传递给 Promise.resolve。


> 参考文章 
>
> [Taming the asynchronous beast with ES7](https://pouchdb.com/2015/03/05/taming-the-async-beast-with-es7.html)
>
> [JavaScript 异步队列实现及拓展](https://juejin.im/post/59cce36751882501c14db49c)
