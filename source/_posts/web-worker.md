---
title: 【转载】浅谈 Web Worker
date: 2018-11-20 15:12:03
tags:
- js
- 浏览器
categories: 浏览器
---

> 转载自[浅谈HTML5 Web Worker](https://juejin.im/post/59c1b3645188250ea1502e46)，略有修改
> 
> 参考[Web Workers 的基本信息 HTML5 Rocks](https://www.html5rocks.com/zh/tutorials/workers/basics/)


## 浅谈 Web Worker

[Web Worker](https://developer.mozilla.org/zh-CN/docs/Web/API/Web_Workers_API) 使得一个 Web 应用程序可以在与主执行线程分离的后台线程中运行一个脚本操作。这样做的好处是可以在一个单独的线程中执行费时的处理任务，从而允许主（通常是UI）线程运行而不被阻塞/放慢。

[规范](https://html.spec.whatwg.org/multipage/workers.html#workers)中介绍了两种 Worker ：专用 Worker 和 共用 Worker。本文只涉及专用 Worker，并在全文中将其称为“Web Worker”或“Worker”。

### 创建 worker

```javascript
var myWorker = new Worker('task.js');

myWorker.onmessage = function(e) {
    console.log('in main:', e.data)
}
myWorker.postMessage({
    msg: 'from main'
})
```

脚本 task.js 在 worker 线程运行。

```javascript
var i = 0;

function timer(wait) {
    setTimeout(() => {
        console.log('in worker:', i);
        postMessage(i);
    }, wait);
}

timer(4000);

onmessage = function(e) {
    console.log(e.data)
}
```

在 task.js 中，global 是一个 [DedicatedWorkerGlobalScope](https://developer.mozilla.org/zh-CN/docs/Web/API/DedicatedWorkerGlobalScope) 对象，setTimeout 时和最外部的 this 指向这个对象，所以这里没有指定调用 possMessage ，onmessage 的对象（其实就是 this.postMessage）。

创建 Worker 总需要新建一个脚本吗？不一定，可以用 [URL.createObjectURL()](https://developer.mozilla.org/zh-CN/docs/Web/API/URL/createObjectURL) 创建 URL 对象，实现内嵌 Worker

```javascript

var myTask = `
    var i = 0;
    function timedCount(){
        i = i+1;
        postMessage(i);
        setTimeout(timedCount, 1000);
    }
    timedCount();
`;

var blob = new Blob([myTask]);
var myWorker = new Worker(window.URL.createObjectURL(blob));
```

### Worker 通信

主页面与 Worker 之间的通信是通过 postMessage() 方法和 onmessage 事件。通信可以由任一方发起，另一方接收。

#### 简单数据传递

在主页面与 Worker 之间传递的数据是通过拷贝，而不是共享来完成的。传递给 Worker 的对象需要经过序列化，接下来在另一端还需要反序列化。页面与 Worker 不会共享同一个实例，最终的结果就是在每次通信结束时生成了数据的一个副本。

也就是说，**Worker 与其主页面之间只能单纯的传递数据，不能传递复杂的引用类型**：如通过构造函数创建的对象等。并且，传递的数据也是经过拷贝生成的一个副本，在一端对数据进行修改不会影响另一端。

#### 通过可转让对象传递数据

前面介绍了简单数据的传递，其实还有一种性能更高的方法来传递数据，就是通过可转让对象将数据在主页面和Worker之间进行来回穿梭。可转让对象从一个上下文转移到另一个上下文而不会经过任何拷贝操作。这意味着当传递大数据时会获得极大的性能提升。和按照引用传递不同，一旦对象转让，那么它在原来上下文的那个版本将不复存在。该对象的所有权被转让到新的上下文内。例如，当你将一个 ArrayBuffer 对象从主应用转让到 Worker 中，原始的 ArrayBuffer 被清除并且无法使用。它包含的内容会(完整无差的)传递给 Worker 上下文。

```javascript
var uInt8Array = new Uint8Array(1024*1024*32); // 32MB
for (var i = 0; i < uInt8Array .length; ++i) {
  uInt8Array[i] = i;
}

console.log(uInt8Array.length); // 传递前长度:33554432

var myTask = `
    onmessage = function (e) {
        var data = e.data;
        console.log('worker:', data); // ArrayBuffer(33554432)
    };
`;

var blob = new Blob([myTask]);
var myWorker = new Worker(window.URL.createObjectURL(blob));
myWorker.postMessage(uInt8Array.buffer, [uInt8Array.buffer]);

console.log(uInt8Array.length); // 传递后长度:0
```

### importScripts()

Worker 线程能够访问一个全局函数 [improtScripts()](https://developer.mozilla.org/zh-CN/docs/Web/API/WorkerGlobalScope/importScripts) 来引入脚本，该函数接受 0 个或者多个 URI 作为参数。

此示例将 script1.js 和 script2.js 加载到了 Worker 中：

worker.js：

```javascript
importScripts('script1.js');
importScripts('script2.js');
// 可以访问 script1.js script2.js 中的全局变量
```

也可以写成单个导入语句：

```javascript
importScripts('script1.js', 'script2.js');
```

### 适用于 Worker 的功能

由于 Web Worker 的多线程行为，所以它们只能使用 JavaScript 功能的子集：

 - navigator 对象
 - location 对象（只读）
 - XMLHttpRequest
 - setTimeout()/clearTimeout() 和 setInterval()/clearInterval()
 - 应用缓存
 - 使用 importScripts() 方法导入外部脚本
 - 生成其他 Web Worker

Worker 无法使用：

 - DOM（非线程安全）
 - window 对象
 - document 对象
 - parent 对象

### 终止 terminate()

在主页面上调用 terminate() 方法，可以立即杀死 worker 线程，不会留下任何机会让它完成自己的操作或清理工作。另外，Worker 也可以调用自己的 close() 方法来关闭自己

```javascript
// 主页面调用
myWorker.terminate();

// Worker 线程调用
self.close();
```

### 处理错误

当 worker 出现运行时错误时，它的 onerror 事件处理函数会被调用。它会收到一个实现了 ErrorEvent 接口名为 error 的事件。相关界面中包含用于找出错误内容的三个实用属性：filename - 导致错误的 Worker 脚本的名称；lineno - 出现错误的行号；以及 message - 有关错误的实用说明。

该事件不会冒泡，并且可以被取消；为了防止触发默认动作，worker 可以调用错误事件的 preventDefault() 方法。

```javascript
<output id="error" style="color: red;"></output>
<output id="result"></output>

<script>
  function onError(e) {
    document.getElementById('error').textContent = [
      'ERROR: Line ', e.lineno, ' in ', e.filename, ': ', e.message].join('');
  }

  function onMsg(e) {
    document.getElementById('result').textContent = e.data;
  }

  var worker = new Worker('workerWithError.js');
  worker.addEventListener('message', onMsg, false);
  worker.addEventListener('error', onError, false);
  worker.postMessage(); // Start worker without a message.
</script>
```

示例：workerWithError.js 尝试执行 1/x，其中 x 未定义。

```javascript
// workerWithError.js：
self.addEventListener('message', function(e) {
  postMessage(1/x); // Intentional error.
});
```

### Worker 使用场景

 - 预先抓取和/或缓存数据以便稍后使用
 - 突出显示代码语法或其他实时文本格式
 - 拼写检查程序
 - 分析视频或音频数据
 - 背景 I/O 或网络服务轮询
 - 处理较大数组或超大 JSON 响应
 - `<canvas>` 中的图片过滤
 - 更新本地网络数据库中的多行内容

### 例子

MDN 几个例子

[基本的dedicated worker示例](https://github.com/mdn/simple-web-worker)

[基本的 shared worker示例](https://github.com/mdn/simple-shared-worker)
