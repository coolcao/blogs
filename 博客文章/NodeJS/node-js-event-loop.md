---
title: 理解nodejs的事件循环
date: 2016-12-22 00:11:50
tags: [nodejs,事件循环]
categories:
- 技术博客
- 原创
---

事件循环机制是nodejs非常非常重要的知识，从网上找的各种资料，却又各种“不同”。
有的文章，从js的执行栈，到事件机制，异步调用，一直讲到事件循环，但是到了事件循环本身的时候，却讲解的又十分含糊，扔张图上去，配两行文字说明，完了。而且，有的图还都不怎么相同，导致看完下来，还是十分模糊，这都说了些啥。。。
我将这些资料整理一遍，梳理了一下，加上自己的理解，成此文。
至于准不准确，其实，我也没底，真的。如果有哪位大神看到有错误的地方，劳烦指出，不胜感激。

<!--more-->

## 什么是事件循环
js是单线程的，但是js的运行时底层的C++ API却是多线程的。
对于浏览器而言，是web API，对于nodejs而言，是libuv库。

先理解几个概念：
### 栈：
函数调用形成了一个堆栈帧。
```js
function f(b){
  var a = 12;
  return a+b+35;
}

function g(x){
  var m = 4;
  return f(m*x);
}

g(21);
```
调用 g 的时候，创建了第一个 堆栈帧 ，包含了 g 的参数和局部变量。当 g 调用 f 的时候，第二个 堆栈帧 就被创建、并置于第一个 堆栈帧 之上，包含了 f 的参数和局部变量。当 f 返回时，最上层的 堆栈帧 就出栈了（剩下 g 函数调用的 堆栈帧 ）。当 g 返回的时候，栈就空了。

[栈的运行示例](http://localhost:4000/2016/12/18/nodejs_at_scale/understanding-the-nodejs-event-loop/)
[运行动画](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)

### 堆
对象被分配在一个堆中，一个用以表示一个内存中大的未被组织的区域。

### 队列
一个 JavaScript 运行时包含了一个待处理的消息队列。每一个消息都与一个函数相关联。当栈为空时，从队列中取出一个消息进行处理。这个处理过程包含了调用与这个消息相关联的函数（以及因而创建了一个初始堆栈帧）。当栈再次为空的时候，也就意味着消息处理结束。

### 事件循环
js运行时至少有两个线程：主线程，工作线程
主线程用于解释执行你写的js代码，工作线程用于循环的从消息队列取消息并执行。

### 简单的事件循环模型图

![chrome](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/20160922091924733.png)

![nodejs](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/nodejs_event_loop.png)

好了，这是最简单的关于事件循环的介绍。
可是，看完依然很蒙圈。
异步事件，包括，网络请求，异步IO，定时器，浏览器的话还有用户的事件，比如点击，拖拽事件等，nodejs经常被面试的两个是：nextProcess和setImmediate

那么这些事件的所有回调任务，都扔到同一个队列吗？
当然不是，实际上，针对不同的事件，有不同的任务队列。比如，会有io队列，定时器队列，等等等等。

同时，任务还分为宏任务，微任务。

微任务的例子：
* process.nextTick
* promise
* Object.observe

宏任务的例子：
* setTimeout
* setInterval
* setImmediate
* I/O

> One go-around of the event loop will have exactly one task being processed from the macrotask queue (this queue is simply called the task queue in the WHATWG specification). After this macrotask has finished, all available microtasks will be processed, namely within the same go-around cycle. While these microtasks are processed, they can queue even more microtasks, which will all be run one by one, until the microtask queue is exhausted.

> What are the practical consequences of this?

> If a microtask recursively queues other microtasks, it might take a long time until the next macrotask is processed. This means, you could end up with a blocked UI, or some finished I/O idling in your application.

> However, at least concerning Node.js's process.nextTick function (which queues microtasks), there is an inbuilt protection against such blocking by means of process.maxTickDepth. This value is set to a default of 1000, cutting down further processing of microtasks after this limit is reached which allows the next macrotask to be processed)

> So when to use what?

> Basically, use microtasks when you need to do stuff asynchronously in a synchronous way (i.e. when you would say perform this (micro-)task in the most immediate future). Otherwise, stick to macrotasks.

> Examples

> macrotasks: setTimeout, setInterval, setImmediate, I/O, UI rendering
> microtasks: process.nextTick, Promises, Object.observe, MutationObserver

这是[stackoverflow](http://stackoverflow.com/questions/25915634/difference-between-microtask-and-macrotask-within-an-event-loop-context)上一个关于这个问题的一个回答。

根据WHATVG规范，在事件循环的一个周期中，应该从宏任务队列中处理正好一个（宏）任务。 在所述宏任务完成之后，所有可用的微任务将在相同的周期内被处理。 当这些微任务正在被处理时，它们可以入队更多的微任务，这些微任务将一个接一个地运行，直到微任务队列耗尽。

这里的宏任务，就是我们经常提到的任务队列中的任务，而微任务，提之甚少。
在《你不知道的js》这本书中，作者有提到，ES6规范新提了一个`Job Queue`工作队列的概念，这个概念和微任务很相似。

因此，这里的再补一个图：
![宏任务/微任务](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/the-Node-js-event-loop.png)

具体的代码实例，可以参看[我的博客中翻译的一篇文章](http://localhost:4000/2016/12/18/nodejs_at_scale/understanding-the-nodejs-event-loop/)

到这里，可能深入点了，但是还不够。
事件循环取队列中任务时，顺序如何？具体怎么取，怎么执行呢？

```txt
   ┌───────────────────────┐
┌─>│        timers         │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     I/O callbacks     │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     idle, prepare     │
│  └──────────┬────────────┘      ┌───────────────┐
│  ┌──────────┴────────────┐      │   incoming:   │
│  │         poll          │<─────┤  connections, │
│  └──────────┬────────────┘      │   data, etc.  │
│  ┌──────────┴────────────┐      └───────────────┘
│  │        check          │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
└──┤    close callbacks    │
   └───────────────────────┘
```

[https://github.com/nodejs/node/blob/master/doc/topics/event-loop-timers-and-nexttick.md](https://github.com/nodejs/node/blob/master/doc/topics/event-loop-timers-and-nexttick.md)

这张图简单说明了事件循环中操作的顺序 。
图中每个盒子都是一个阶段。

阶段说明：
* timers 阶段: 这个阶段执行setTimeout(callback) and setInterval(callback)预定的callback;
* I/O callbacks 阶段: 执行除了 close事件的callbacks、被timers(定时器，setTimeout、setInterval等)设定的callbacks、setImmediate()设定的callbacks之外的callbacks;
* idle, prepare 阶段: 仅node内部使用;
* poll 阶段: 获取新的I/O事件, 适当的条件下node将阻塞在这里;
* check 阶段: 执行setImmediate() 设定的callbacks;
* close callbacks 阶段: 比如socket.on(‘close’, callback)的callback会在这个阶段执行.

每一个阶段都有一个装有callbacks的fifo queue(队列)，当event loop运行到一个指定阶段时，
node将执行该阶段的fifo queue(队列)，当队列callback执行完或者执行callbacks数量超过该阶段的上限时，
event loop会转入下一下阶段.

> 注意上面六个阶段都不包括 process.nextTick()

阶段说明：
* timers:一个定时器指定了一个时间阀值，过了这个值执行提供的回调函数，而不是一个人们希望它执行的确切时间。当指定的时间已过，定时器的回调函数会尽早执行，然后，操作系统的其他定时任务或者正在执行其他回调将会延迟它们的执行。

> 注：技术上，poll轮询阶段控制定时器何时执行。

举个例子，你指定一个定时器100ms后执行，然后你的代码异步读取一个文件，耗费95ms:

```js
var fs = require('fs');

function someAsyncOperation (callback) {
  // Assume this takes 95ms to complete
  fs.readFile('/path/to/file', callback);
}

var timeoutScheduled = Date.now();

setTimeout(function () {

  var delay = Date.now() - timeoutScheduled;

  console.log(delay + "ms have passed since I was scheduled");
}, 100);


// do someAsyncOperation which takes 95 ms to complete
someAsyncOperation(function () {

  var startCallback = Date.now();

  // do something that will take 10ms...
  while (Date.now() - startCallback < 10) {
    ; // do nothing
  }

});
```

这里例子中，定时器打印时，已经是105ms了。
因为读取文件耗费95ms，当文件读物完后，执行callback，callback中又耗费10ms啥也没干，干耗着不放CPU资源。当定时器到达100ms时，将会被扔到定时器队列中。然而这时由于文件读取事件还未完成，因此定时器任务只能等待。等105ms后，文件读取任务完成，取出定时器队列中的回调执行。

poll阶段：
在node.js里，任何异步方法（除timer,close,setImmediate之外）完成时，都会将其callback加到poll queue里,并立即执行。
poll 阶段有两个主要的功能：

* 处理poll队列（poll quenue）的事件(callback);
* 执行timers的callback,当到达timers指定的时间时;

如果event loop进入了 poll阶段，且代码未设定timer，将会发生下面情况：

* 如果poll queue不为空，event loop将同步的执行queue里的callback,直至queue为空，或执行的callback到达系统上限;
* 如果poll queue为空，将会发生下面情况：
    * 如果代码已经被setImmediate()设定了callback, event loop将结束poll阶段进入check阶段，并执行check阶段的queue (check阶段的queue是 setImmediate设定的)
    * 如果代码没有设定setImmediate(callback)，event loop将阻塞在该阶段等待callbacks加入poll queue;

如果event loop进入了 poll阶段，且代码设定了timer：

* 如果poll queue进入空状态时（即poll 阶段为空闲状态），event loop将检查timers,如果有1个或多个timers时间时间已经到达，event loop将按循环顺序进入 timers 阶段，并执行timer queue.
