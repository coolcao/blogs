---
title: 【译】nodejs v7初体验之async/await
date: 2016-12-12 16:25:36
tags: [nodejs,es7,async/await]
categories:
- 技术博客
- 翻译
---

几个月之前，async/await终于登录了V8引擎。与此同时，nodejs中V8引擎也进行多次升级，最新的Nightly版本也支持async/await了。
> 声明：async/await现在只在nodejs的非稳定版本Nightly版本中可用。**现在不要用在生产环境中**！！！

<!--more-->

## 什么是async/await
首先，让我们看一下，用Promise是如何处理异步操作的。这个简单的小例子展示了如何用Fetch API和Promise获取数据。

```js
function getTrace () {
  return fetch('https://trace.risingstack.com', {
    method: 'get'
  })
}

getTrace()
  .then()
  .catch()
```

使用async/await你可以等待Promise。这将以非阻塞的方式停​​止执行 - 因为它等待结果并返回。如果Promise被拒绝而没有resolve，则拒绝的值将会被抛出，这意味着它可以被try/catch捕获。

前面的例子使用async/await编写将看起来下面的样子：

```js
async function getTrace () {
  let pageContent
  try {
    pageContent = await fetch('https://trace.risingstack.com', {
      method: 'get'
    })
  } catch (ex) {
    console.error(ex)
  }

  return pageContent
}

getTrace()
  .then()
```

想要深入了解async/await，我建议阅读以下资源：
* [https://ponyfoo.com/articles/understanding-javascript-async-await](https://ponyfoo.com/articles/understanding-javascript-async-await)
* [https://tc39.github.io/ecmascript-asyncawait/](https://tc39.github.io/ecmascript-asyncawait/)

## 不用转译工具使用async/await
### 安装Node 7
要在nodejs中使用async/await，首先必须获取最新的nodejs版本。登录每日[构建版本（Nightly builds）](https://nodejs.org/download/nightly/)获取最新的v7版本node。

下载安装包安装，就可以使用了。

如果你使用nvm管理你的node版本，你可以这样安装：

```
NVM_NODEJS_ORG_MIRROR=https://nodejs.org/download/nightly
nvm install 7
nvm use 7
```

### 使用async/await运行
我们创建一个简单的js文件，使用`setTimeout`延迟执行一个函数，但是使用async/await包裹调用。

```js
// app.js
const timeout = function (delay) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve()
    }, delay)
  })
}

async function timer () {
  console.log('timer started')
  await Promise.resolve(timeout(100));
  console.log('timer finished')
}

timer()
```

你可以使用如下方式运行：

```
node app.js
```

但是你会发现，并不起作用。原因是async/await的支持现在仍然是隐藏的标识，要运行它，你必须得使用下面语句：

```
node --harmony-async-await app.js
```

### 使用async/await创建一个web服务器
对于Koa v2，Koa已经支持async函数作为中间件。之前的时候，它支持转译工具，但是从现在开始不再是这样了。
你可以传一个async函数作为Koa中间件：

```js
// app.js
const Koa = require('koa')
const app = new Koa()

app.use(async (ctx, next) => {
  const start = new Date()
  await next()
  const ms = new Date() - start
  console.log(`${ctx.method} ${ctx.url} - ${ms}ms`)
})

app.use(ctx => {
  ctx.body = 'Hello Koa'
})

app.listen(3000)
```

当你使用Koa写了一个服务器，你可以使用下面语句启动：

```
node --harmony-async-await app.js
```

## 什么时候开始用它
Nodejs V8版本，下一个稳定的版本，将包含支持async/await操作的V8引擎版本将会在2017年4月份发布。在此之前，你可以继续使用不稳定分支Nodejs v7版本体验async/await。

原文地址：[Experimenting With async/await in Node.js 7 Nightly](https://blog.risingstack.com/async-await-node-js-7-nightly/)
