---
title: 【译】【4】理解Nodejs的事件循环
date: 2016-12-18 17:22:25
tags: [js]
categories:
- 技术博客
- 翻译
- Nodejs At Scale
---

本篇文章将帮助你了解Nodejs事件循环的工作原理，以及如何利用它来构建快速应用程序。我们还将讨论你可能遇到的最常见的问题，以及它们的解决方案。

<!--more-->

## 问题

大部分网站的后端不需要进行复杂的计算。我们的程序花费大部分的时间等待磁盘的读写或者等待网络传输我们的消息并发送回答。

IO操作比数据处理要慢几个数量级。举例来说，SSD的读取速度可以达到200-730MB/s，至少一个高端的SSD能达到。只读取一千字节的数据将需要1.4微秒，但在此期间，时钟频率为2GHz的CPU可能执行了28000个指令处理周期。

对于网络通信来说，情况可能更糟，只需尝试着ping一下google.com：

```shell
$ ping google.com
64 bytes from 172.217.16.174: icmp_seq=0 ttl=52 time=33.017 ms
64 bytes from 172.217.16.174: icmp_seq=1 ttl=52 time=83.376 ms
64 bytes from 172.217.16.174: icmp_seq=2 ttl=52 time=26.552 ms
64 bytes from 172.217.16.174: icmp_seq=3 ttl=52 time=40.153 ms
64 bytes from 172.217.16.174: icmp_seq=4 ttl=52 time=37.291 ms
64 bytes from 172.217.16.174: icmp_seq=5 ttl=52 time=58.692 ms
64 bytes from 172.217.16.174: icmp_seq=6 ttl=52 time=45.245 ms
64 bytes from 172.217.16.174: icmp_seq=7 ttl=52 time=27.846 ms
```

平均延迟大约44毫秒。只是等待数据包在线路上进行往返时，前面提到的处理器可以执行88百万个周期。

## 解决方案
大多数操作系统提供了某种异步IO接口，它允许你开始处理一些数据，而不需要直接返回通信结果，同时，通信仍在继续。

这有几种实现方式。现如今，它主要通过利用多线程的方式来实现，但是以额外的软件复杂性为代价。例如，用Java或Python读取文件是一个阻塞操作。你的程序在等待网络/磁盘通信完成时，无法执行任何操作。你所能做的(至少在Java中)是启动不同的线程，然后在操作完成后通知你的主线程。

这种方式乏味且复杂，但是它解决了上面提到的问题。但是对于node呢？在nodejs方面，我们肯定面临着一些问题，因为nodejs或者说v8是单线程的。我们的代码只能在一个线程中运行。

> 这并不完全是真的。Java和Python都有异步接口，但是使用它们肯定比nodejs更难。感谢[Shahar](https://disqus.com/by/keidi19/)和[Dirk Harrington](https://twitter.com/dirkjharrington)指出这一点。

你可能听过说，在浏览器中，设置`setTimeout(someFunction,0)`有时可以魔法的解决一些问题。但是为什么设置超时为0？推迟执行0毫秒能解决什么呢？是不是和简单的立即调用someFunction函数一样呢？并不是这样的。

首先，我们来看一下调用栈，或简单地，“堆栈”。我们把事情简单化，因为我们只需要了解堆栈的基本只是。如果你熟悉它的工作原理，请随时跳到下一章节。

## 堆栈
无论什么时候，调用函数返回地址，参数和局部变量都将被推送到栈中。如果你在当前运行的函数调用另一个函数，它的内容连同返回地址，将被以之前同样的方式推送到栈的顶部。

为简单起见，从现在起，我将说一个函数被推入到栈的顶部，即使它不完全正确。

让我们来看一下：

```js
 function main () {
   const hypotenuse = getLengthOfHypotenuse(3, 4)
   console.log(hypotenuse)
 }

 function getLengthOfHypotenuse(a, b) {
   const squareA = square(a)
   const squareB = square(b)
   const sumOfSquares = squareA + squareB
   return Math.sqrt(sumOfSquares)
 }

 function square(number) {
   return number * number
 }

 main()
```

main函数被首先调用：

![main](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/the-main-function.png)

然后，main函数里面调用了getLengthOfHypotenuse()，以3和4作为参数。

![The-getLengthOfHypotenuse-function](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/The-getLengthOfHypotenuse-function.png)

再然后是以a为参数调用square()函数：

![The-square-a--function-1](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/The-square-a--function-1.png)

当square()函数返回时，它从栈中弹出，并且将其返回值赋值给squareA变量。squareA被添加到getLengthOfHypotenuse的堆栈帧中。

![squareA](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/variable_a.png)

下一个square()函数将以同样的方式调用：

![The-square-b-function-1-1](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/The-square-b-function-1-1.png)

![squareB](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/variable_b.png)

下一行的表达式`squareA + squareB`将被计算：

![sumOfSqaures](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/sumOfSqaures.png)

然后，Math.sqrt()将被调用，以sumOfSquares作为参数。

![Math.sqrt](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/Math.sqrt.png)

现在，getLengthOfHypotenuse只剩下返回它计算的最终的值：

![The-return-functio](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/The-return-function.png)

返回值在main函数中被赋值给hypotenuse。

![hypotenuse](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/hypotenuse.png)

hypotenuse的值被打印到终端：

![console-log](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/console-log.png)

最后，main函数没有返回任何值，从栈中弹出，栈为空。

![finally](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/finally.png)

> 注意：当函数执行完成时，局部变量从栈中弹出。它只发生在当你使用简单值时，例如数字，字符串和布尔值。对象，数组等的值存储在堆中，你的变量只是指向它们的指针。如果传递这个变量，将只传递这个指针，使这些值在不同栈帧中是可变的。当函数从栈弹出时，只有指向对象的指针会弹出，它们实际的值将保留在堆中。垃圾回收器就是这样一个角色，一旦对象无用了，就会释放这些对象占的空间。

## nodejs的事件循环
当我们调用像setTimeout,http.get,process.nextTick或fs.readFile这样的函数时会发生什么呢？在v8的代码中找不到这些东西，但它们可以在Chrome的WebApi中或者如果是nodejs的话，可以在C++的API中找到。要理解这一点，我们必须了解一点执行的顺序。

让我来看一个更常见的Nodejs应用，一个服务器监听在`localhost:3000`。得到请求后，服务器将调用`wttr.in/<city>`来获取添加，向控制台打印一些消息，并在收到响应后转发给调用者。

```js
'use strict'
const express = require('express')
const superagent = require('superagent')
const app = express()

app.get('/', sendWeatherOfRandomCity)

function sendWeatherOfRandomCity (request, response) {
  getWeatherOfRandomCity(request, response)
  sayHi()
}

const CITIES = [
  'london',
  'newyork',
  'paris',
  'budapest',
  'warsaw',
  'rome',
  'madrid',
  'moscow',
  'beijing',
  'capetown',
]

function getWeatherOfRandomCity (request, response) {
  const city = CITIES[Math.floor(Math.random() * CITIES.length)]
  superagent.get(`wttr.in/${city}`)
    .end((err, res) => {
      if (err) {
        console.log('O snap')
        return response.status(500).send('There was an error getting the weather, try looking out the window')
      }
      const responseText = res.text
      response.send(responseText)
      console.log('Got the weather')
    })

  console.log('Fetching the weather, please be patient')
}

function sayHi () {
  console.log('Hi')
}

app.listen(3000)
```

当请求localhost:3000时，除了获取的天气，还会打印什么？

如果你有一些node的经验，你应该不会惊讶，即使在代码上`console.log('Fetching the weather, please be patient')`是在`console.log('Got the weather')`之后，上面的结果将会是如下形式：

```
Fetching the weather, please be patient
Hi
Got the weather
```

这发生了什么？即使v8是单线程的，但是node底层的C++API不是。这意味着每当我们调用一个非阻塞操作时，node会调用一些代码，这些代码与我们的javascript代码同时运行。一旦这个隐藏线程接收到它等待的返回值或抛出错误，提供的callback将会使用必要的参数被调用。

> 注意：我们提到的“一些代码”实际上是[libuv](https://github.com/libuv/libuv)的一部分。libuv是一个开源的库，用以处理线程池，执行信令及其他魔法，以使异步任务工作。它最初是为nodejs开发的，但现在[很多项目](https://github.com/libuv/libuv/wiki/Projects-that-use-libuv)都使用它。

要揭开表象，窥探本质，我们需要引入两个新的概念：事件循环和任务队列。

### 任务队列
Javascript是单线程，事件驱动的语言。这意味着我们可以给事件附加监听器，当事件触发时，监听器就会执行我们提供的回调函数。

每当你调用setTimeout,http.get或fs.readFile,nodejs将这些操作发送到不同的线程，允许v8继续执行我们的代码。当计数器运行完成或IO/http操作完成时，nodejs将调用回调函数。

这些回调函数可以入队其他任务，这些函数也可以入队其他等等。(原文是：These callbacks can enqueue other tasks and those functions can enqueue others and so on.有点迷糊。。。)这样，你可以在处理服务器中的请求时读取文件，然后根据读取的内容进行http调用，而不阻止其他请求。

然而，我们只有一个主线程和一个调用栈，所以如果在读取文件时有另一个请求被服务，它的回调需要等待栈变为空。回调函数等待轮到它们执行的地方被称为任务队列（事件队列或消息队列）。每当主线程完成其前一任务时，回调函数会被调用在无限循环中，因此名称为“事件循环”。

在我们前一个例子中，看起来应该是这样的：

* express注册了一个request事件的处理器，当请求到达'/'时会被调用。
* 跳过函数并开始监听端口3000
* 此时的栈为空，等待request事件的触发
* 当传入请求时，等待的事件被触发，express调用提供的处理程序sendWeatherOfRandomCity
* sendWeatherOfRandomCity被推送到栈
* getWeatherOfRandomCity被调用并且推送到栈
* Math.floor和Math.random将被调用，推送到栈，然后弹出，一个从cities获取的随机值将被赋值给变量city
* 发送到 http://wttr.in/${city} 的一个http将被发送到后台线程，然后继续执行
* 'Fetching the weather, please be patient'被记录到控制台，getWeatherOfRandomCity函数返回
* sayHi被调用，"Hi"被打印到控制台
* sendWeatherOfRandomCity返回，从栈获取弹出值，栈此时为空
* 等待http://wttr.in/${city}发送响应
* 一旦接收到响应，end事件将被触发
* 我们传递给.end()的匿名处理器将被调用，被推送到栈，所有变量都在其闭包中，这意味着它可以查看和修改express,superagent,app,CITIES,request,response,city和所有我们定义的函数。
* response.send()使用200或500作为状态码，但是它又被发送到后台进程，因此响应流不会阻止我们的执行，匿名处理器被从栈中弹出。

现在我们就了解了之前提到过的setTimeout是如何工作的。尽管我们将计数器设置为0，它被延迟执行，一直到当前栈和任务队列为空时，这样允许浏览器重绘UI或者node服务其他请求。

### 微任务和宏任务
如果这还不够，我们实际上有多于一个任务队列。一个用于微任务，另一个用于宏任务。

微任务的例子：
* process.nextTick
* promise
* Object.observe

宏任务的例子：
* setTimeout
* setInterval
* setImmediate
* I/O

让我们看下面的代码：

```js
console.log('script start')

const interval = setInterval(() => {
  console.log('setInterval')
}, 0)

setTimeout(() => {
  console.log('setTimeout 1')
  Promise.resolve().then(() => {
    console.log('promise 3')
  }).then(() => {
    console.log('promise 4')
  }).then(() => {
    setTimeout(() => {
      console.log('setTimeout 2')
      Promise.resolve().then(() => {
        console.log('promise 5')
      }).then(() => {
        console.log('promise 6')
      }).then(() => {
        clearInterval(interval)
      })
    }, 0)
  })
}, 0)

Promise.resolve().then(() => {
  console.log('promise 1')
}).then(() => {
  console.log('promise 2')
})
```

结果：

```
script start
promise1
promise2
setInterval
setTimeout1
promise3
promise4
setInterval
setTimeout2
setInterval
promise5
promise6
```

根据WHATVG规范，在事件循环的一个周期中，应该从宏任务队列中处理正好一个（宏）任务。 在所述宏任务完成之后，所有可用的微任务将在相同的周期内被处理。 当这些微任务正在被处理时，它们可以入队更多的微任务，这些微任务将一个接一个地运行，直到微任务队列耗尽。

使用下面的图片可能说的更清楚些：

![the-Node-js-event-loop](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/the-Node-js-event-loop.png)

在我们的例子中：

周期1:
1. `setInterval`被安排为一个任务
2. `setTimeout1`被安排为一个任务
3. 在`Promise.resolve1`中，两个`then`都被安排为微任务
4. 栈是空的，微任务被执行

任务队列：setInterval,setTimeout1

周期2:
5. 微任务队列为空，`setInterval`处理器被执行，另外一个`setInterval`被安排为任务，就在`setTimeout1`之后

任务队列：setTimeout1,setInterval

周期3:
6. 微任务队列为空，`setTimeout1`的处理函数被执行，`promise3`和`promise4`被安排为微任务
7. `promise3`和`promise4`的处理函数被执行，`setTimeout2`被安排为任务

任务队列：setInterval,setTimeout2

周期4:
8. 微任务队列为空，`setInterval`处理函数被执行，另外一个`setInterval`被安排为任务，在`setTimeout`之后

任务队列：setTimeout2,setInterval

9. `setTimeout2`的处理函数被执行，`promise5`和`promise6`被安排为微任务

现在，promise5和promise6应当被执行，清除了我们的interval，但是，由于一些奇怪的原因，setInterval又执行了。然后，如果你在chrome下执行这段代码，你将会得到期望的结果。

在Node中，我们也可以使用`process.nextTick`来解决这个问题，和一些令人难以置信的回调地狱。

```js
console.log('script start')

const interval = setInterval(() => {
  console.log('setInterval')
}, 0)

setTimeout(() => {
  console.log('setTimeout 1')
  process.nextTick(() => {
    console.log('nextTick 3')
    process.nextTick(() => {
      console.log('nextTick 4')
      setTimeout(() => {
        console.log('setTimeout 2')
        process.nextTick(() => {
          console.log('nextTick 5')
          process.nextTick(() => {
            console.log('nextTick 6')
            clearInterval(interval)
          })
        })
      }, 0)
    })
  })
})

process.nextTick(() => {
  console.log('nextTick 1')
  process.nextTick(() => {
    console.log('nextTick 2')
  })
})
```

这和我们所爱的promise使用完全相同的逻辑，只是有点丑陋。至少它按照我们预期的方式完成工作。

### 驯服异步这头怪兽
正如我们所见，当我们使用nodejs编写应用时，我们要管理并且注意两个任务队列以及事件循环，如果我们希望利用它所有的功能，并且我们希望我们的长任务不会阻塞主线程。

事件循环最开始可能是一个不易掌握的概念，但是一旦你掌握了它，你将无法想象没有它的生活。连续传递风格会导致一个看起来很丑陋的回调地狱问题，但是我们有Promise，并且很快我们就可以使用async-await了，在等待这期间，你可以使用co或koa去模拟async-await。

### 最后一点建议
了解了nodejs和v8如何处理长时间运行的任务，你可以现在就使用它。你可能之前已经听说过，应该将长时间运行的循环发送到任务队列。你可以自己手动处理或直接使用 async.js。
