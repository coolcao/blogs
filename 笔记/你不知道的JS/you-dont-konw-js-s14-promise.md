---
title: 学习笔记-你不知道的js-Promise
date: 2021-03-18 21:33:21
tags: [js]
categories:
- 学习笔记
- 你不知道的js
---

在回调那一章节里，我们探讨过使用回调函数来表达异步和管理并发的两个缺陷：缺乏顺序性和可信任性。尤其是缺乏可信任性，信任很脆弱，也很容易失去。

回忆一下，我们用回调函数来封装程序中的continuation，然后把回调交给第三方（甚至可能是外部代码），接着期待其能够回调，实现正确的功能。这种控制权的反转，会导致信任问题。

但如果我们能够把控制反转再反转过来，会怎样呢？如果我们不把自己程序的continuation传给第三方，而是希望第三方给我们提供了解其任务何时结束的能力，然后由我们自己的代码来决定下一步做什么，那将会怎样？

这种范式就称为Promise。

## 什么是Promise
### 未来值
promise是一种封装和组合未来值的易于复用的机制。

### 完成事件
假定要调用一个函数foo(..)来完成某个任务，我们不知道，也不关心它的任何细节，这个函数可能立即完成，也可能需要一段时间才能完成，我们只需要直到这个函数什么时候结束，这样就可以进行下一个任务了。
换句话说，我们想要通过某种方式在foo()函数完成的时候得到通知，以便可以进行下一步操作。

在典型的JavaScript风格中，如果要监听某个通知，你可能会想到事件。可以把对通知的需求重新组织为对foo()发出的一个完成事件的监听。

考虑如下伪代码：
```js
foo(x) {  
	// 开始做点可能耗时的工作
}
foo( 42 );
on (foo "completion") { 
	// 可以进行下一步了!
}

on (foo "error") {  
	// 啊，foo(..)中出错了
}
```

我们调用foo()并建立了两个监听事件：一个用于completion，一个用于error，从本质上来讲，foo()并不需要了解调用代码订阅了这些事件，这样就很好的实现了**关注点分离**。

在JavaScript中我们使用事件来解决：
```js
function foo(x) {
  // 开始做点可能耗时的工作
  // 构造一个listener事件通知处理对象来返回
  return listener;
}
var evt = foo(42);
evt.on("completion", function () {
  // 可以进行下一步了!
});
evt.on("failure", function (err) {
  // 啊，foo(..)中出错了
});
```

foo()创建并返回了一个事件订阅对象，调用代码得到这个对象，并在其上注册两个事件处理函数。这样就把控制权返回给调用者，这也是我们最开始想要的结果。

对控制反转的恢复实现了更好的关注点分离， 从本质上说，evt 对象就是分离的关注点之间一个中立的第三方协商机制。

## 具有then方法的鸭子类型
在Promise领域，一个重要的细节就是，如何确定某个值就是Promise？

既然Promise是通过new Promise(..)创建的，那么，是不是可以通过 `p instanceof Promise`来检查，遗憾的是，这并不足以作为检查方法，因为Promise值可能是从其他浏览器窗口接受到的，这个浏览器窗口的Promise和当前窗口的不同，并且，有一些第三方的库或框架可能会自己实现Promise，而不是使用ES6 的Promise实现，这样就不能如此判断了。

因此，识别 Promise(或者行为类似于 Promise 的东西)就是定义某种称为 thenable 的东 西，将其定义为任何具有 then(..) 方法的对象和函数。我们认为，任何这样的值就是 Promise 一致的 thenable。
 

根据一个值的形态(具有哪些属性)对这个值的类型做出一些假定。这种类型检查(type check)一般用术语鸭子类型(duck typing)来表示——“如果它看起来像只鸭子，叫起来 像只鸭子，那它一定就是只鸭子”。

这就导致了，如果你的函数恰好返回一个带有then()回调的对象，那么它就会被认定为thenable，在ES6之前，社区已经有一些非常著名的非Promise库恰好名为then()的方法，这些库中的一些选择了重命名自己的方法以避免冲突，而有些库则因为无法通过改变避免冲突，很不幸的被降级进入了“与Promise编码不兼容”的状态。

>  我并不喜欢最后还得用 thenable 鸭子类型检测作为 Promise 的识别方案。还 有其他选择，比如 branding，甚至 anti-branding。可我们所用的似乎是针 对最差情况的妥协。但情况也并不完全是一片黯淡。后面我们就会看到， thenable 鸭子类型检测还是有用的。只是要清楚，如果 thenable 鸭子类型误 把不是 Promise 的东西识别为了 Promise，可能就是有害的。

## Promise 的信任问题
Promise是如何解决回调函数的信任问题的呢？Promise的then()不也是回调函数么？

回顾一下回调编码的信任问题，当把一个回调传入一个第三方工具函数时，可能会出现如下问题：
- 调用回调过早
- 调用回调过晚，或不被回调
- 调用回调次数过多或过少
- 未能传递环境所需要的参数
- 吞掉所出现的异常或错误

### 调用过早
这个问题主要就是担心，代码是否会引入类似Zalgo这样的副作用，在这类问题中，一个任务同步完成，一个任务异步完成，可能会导致竞态条件。

在Promise中就不用担心这个问题，因为在Promise中都是异步任务，即使是及时完成的Promise，也无法被同步观察到。
 
也就是说，对一个 Promise 调用 then(..) 的时候，即使这个 Promise 已经决议，提供给 then(..) 的回调也总会被异步调用。

### 调用过晚
 Promise 创建对象调用 resolve(..) 或 reject(..) 时，这个 Promise 的 then(..) 注册的观察回调就会被自动调度。可以确信，这些被调度的回调在下一个异步事 件点上一定会被触发。
 
 ### 回调未调用
 没有任何东西(甚至 JavaScript 错误)能阻止 Promise 向你通知它的决议(如果它 决议了的话)。如果你对一个 Promise 注册了一个完成回调和一个拒绝回调，那么 Promise 在决议时总是会调用其中的一个。
 
 ### 调用次数过多或过少
 根据定义，回调被调用的正确次数应该是1， “过少”的情况就是调用0次，和前面的未调用情况是一致的。
 “过多”的情况也很容易解释，Promise的定义方式使得它只能被决议一次，如果出于某种原因，Promise的代码试图调用resolve或reject多次，或者试图两者都调用，那么这个Promise将会只接受第一次决议，并默默的忽略任何后续调用。
  
由于 Promise 只能被决议一次，所以任何通过 then(..) 注册的(每个)回调就只会被调 用一次。

### 未能传递参数/环境值
Promise至多只能有一个决议值（完成或拒绝）。

如果你没有用任何值显式决议，那么这个值就是 undefined，这是 JavaScript 常见的处理方 式。但不管这个值是什么，无论当前或未来，它都会被传给所有注册的(且适当的完成或 拒绝)回调。

如果要传递多个值，你就必须要把它们封装在单个值中传递，比如通过一个数组或对象。

### 吞掉错误或异常

基本上，这部分是上个要点的再次说明。如果拒绝一个 Promise 并给出一个理由(也就是一个出错消息)，这个值就会被传给拒绝回调。

不过在这里还有更多的细节需要研究。如果在 Promise 的创建过程中或在查看其决议 结果过程中的任何时间点上出现了一个 JavaScript 异常错误，比如一个 TypeError 或 ReferenceError，那这个异常就会被捕捉，并且会使这个 Promise 被拒绝。

```js
var p = new Promise(function (resolve, reject) {
  foo.bar(); // foo未定义，所以会出错!
  resolve(42); // 永远不会到达这里 :(
});
p.then(
  function fulfilled() {
    // 永远不会到达这里 :(
  },
  function rejected(err) {
    // err将会是一个TypeError异常对象来自foo.bar()这一行
  }
);
```

foo.bar() 中发生的 JavaScript 异常导致了 Promise 拒绝，你可以捕捉并对其作出响应。

### 是可信任的Promise吗
可能你已经注意到了，Promise并未完全摆脱回调，它们只是改变了传递回调的位置，我们并不是把回调传递给foo()而是从foo()得到某个东西（Promise），然后把回调传递给这个东西。

但是，为什么这样比单纯使用回调更值得信任呢？如何确定返回的这个东西实际上就是一个可信任的Promise？

如果向Promise.resolve()传递一个非Promise,非thanable的值，就会得到用这个值填充的Promise。

```js
var p1 = new Promise( function(resolve,reject){
  resolve( 42 );
} );
var p2 = Promise.resolve( 42 );
```
p1和p2的行为是完全一样的。

而如果向Promise.resolve()传递一个真正的Promise，就会返回同一个Promise。

```js
var p1 = Promise.resolve(1);
var p2 = Promise.resolve(p1);

console.log(p1 === p2); // true
```

更重要的是，如果向 Promise.resolve(..) 传递了一个非 Promise 的 thenable 值，前者就会试图展开这个值，而且展开过程会持续到提取出一个具体的非类 Promise 的最终值。

```js
var p = {
  then: function (cb, errcb) {
    cb(42);
    errcb("evil laugh");
  },
};
p.then(
  function fulfilled(val) {
    console.log(val); // 42
  },
  function rejected(err) {
    // 啊，不应该运行!
    console.log(err); // 邪恶的笑
  }
);

Promise.resolve(p).then(v => {
  console.log('---------');
  console.log(v);
  console.log('----------');
}).catch(err => console.log(err));
//42
//evil laugh
//---------
//42
//----------
```

Promise.resolve(..) 可以接受任何 thenable，将其解封为它的非 thenable 值。从 Promise. resolve(..) 得到的是一个真正的 Promise，是一个可以信任的值。如果你传入的已经是真 正的 Promise，那么你得到的就是它本身，所以通过 Promise.resolve(..) 过滤来获得可信 任性完全没有坏处。

## 链式流
Promise并不只是一个单步执行的 this-then-that 操作的机制，我们可以把多个Promise链接到一起，来表示一系列的异步步骤。

这种方式可以实现的关键在于以下两个Promise固有的特征：
1. 每次对Promise调用then()，它都会创建并返回一个新的Promise，我们可以将其链接起来。
2. 不管从then()调用的完成回调(第一个参数)返回值是什么，它都会被自动设置为被链接Promise的完成。

比如下面的两个Promise：
```js
var p1 = Promise.resolve(1);
var p2 = p1.then(v => {
  console.log(v);
  return 2;
});

p2.then(v => {
  console.log(v);
});
```

p1，p2都是标准的Promise，所以可以对p1,p2做then()回调。

因此，上面的代码可以将中间的命名过程省略掉，改写成下面链式代码：

```js
Promise.resolve(1)
  .then((v) => {
    console.log(v);
    return 2;
  })
  .then((v) => {
    console.log(v);
  });
```

这样的链式操作，优势在于，不管我们有多少个异步操作，都可以很好的根据调用顺序，组织成一个Promise链，这样每个Promise的决议都将成为下一个Promise的信号值。


如果调用p.then()只传一个处理函数，那么另外的一个处理函数会有一个默认的处理函数。对于默认的成功处理函数，会把返回值透传到下一个promise，而对于默认的错误处理函数，会直接抛出这个错误，将错误透传到下一个promise。

```js
Promise.resolve(1)
  .then()
  .then((v) => {
    console.log(v);	// 1
  });
```

```js
Promise.reject('error')
  .then((v) => {
    console.log(v); // 不会走到这里
  }).then(v => {
    console.log(v); // 同样不会走到这里
  },err => {
    console.log(err);
  });
```

## 错误处理
对于多数开发者而言，处理错误最自然的方式是使用try..catch，但遗憾的是，在JavaScript中，它只支持同步的方式，无法支持异步代码。

在回调的模式中，一些模式化的处理方式出现，比如大家经常用的 error-first 回调风格：
```js
function foo(cb) {
  setTimeout(function () {
    try {
      var x = baz.bar();
      cb(null, x); // 成功!
    } catch (err) {
      cb(err);
    }
  }, 100);
}
foo(function (err, val) {
  if (err) {
    console.error(err); // 烦 :(
  } else {
    console.log(val);
  }
});
```
如上，你无法直接在foo()使用try..catch，因为foo()是异步操作，你只能传一个回调过去，而这个回调，约定有两个参数，err和val，并约定err作为第一个参数，表示发生异步的错误值。这就是 error-first 回调风格。

传给 foo(..) 的回调函数保留第一个参数 err，用于在出错时接收到信号。如果其存在的 话，就认为出错;否则就认为是成功。

对于Promise的错误处理，并没有使用 error-first 风格的回调，而是使用了分离回调风格。一个回调用于处理完成的情况，一个回调用于处理错误的情况。

但这种风格也是有个弊端，如下：

```js
var p = Promise.resolve(42);
p.then(
  function fulfilled(msg) {
    // 数字没有string函数，所以会抛出错误
    console.log(msg.toLowerCase());
  },
  function rejected(err) {
    // 永远不会到达这里
  }
);
```

在完成函数fulfiled()中，我们对数字进行toLowerCase()肯定会抛出错误，这是错误会被传到下一个promise,这里rejected()就不会被执行到，所以抛出的错误也无法被捕捉到。

所幸的是，promise除了then()还有一个catch()回调，这个catch可以捕捉整个promise链中的任何一个环境抛出的异常，上面的代码就可以直接在后面加一个catch来捕捉抛出的异常。

```js
var p = Promise.resolve(42);
p.then(
  function fulfilled(msg) {
    // 数字没有string函数，所以会抛出错误
    console.log(msg.toLowerCase());
  },
  function rejected(err) {
    // 永远不会到达这里
  }
).catch(err => {
  console.log(err);
});
// 输出以下结果：
TypeError: msg.toLowerCase is not a function
    at fulfilled
```


## Promise模式
### Promise.all([..])
上面所讨论的Promise链，都是有顺序的，一个Promise完成，再执行下一个Promise。
但如果想要同时执行多个任务，怎么办？比如有两个异步任务，其并不关联，谁先完成不重要，但重要的是要等到这两个任务都完成了再执行下一个任务。

Promise提供了Promise.all()方法就是做这个事的。

```js
// request(..)是一个Promise-aware Ajax工具
// 就像我们在本章前面定义的一样
var p1 = request("http://some.url.1/");
var p2 = request("http://some.url.2/");
Promise.all([p1, p2])
  .then(function (msgs) {
    // 这里，p1和p2完成并把它们的消息传入
    return request("http://some.url.3/?v=" + msgs.join(","));
  })
  .then(function (msg) {
    console.log(msg);
  });
```

### Promise.race([..])
尽管Promise.all([ .. ])协调多个并发Promise的运行，并假定所有Promise都需要完成，但有时候你会想只响应“第一个跨过终点线的 Promise”，而抛弃其他 Promise。 这种模式传统上称为门闩，但在 Promise 中称为竞态。

与Promise.all([ .. ])类似，一旦有任何一个Promise决议为完成，Promise.race([ .. ]) 就会完成;一旦有任何一个 Promise 决议为拒绝，它就会拒绝。

```js
Promise.race([]).then(v => console.log(v)).catch(e => console.log(e));
// 什么也不会发生，不会决议，也不会拒绝
```
>  一项竞赛需要至少一个“参赛者”。所以，如果你传入了一个空数组，主 race([..]) Promise 永远不会决议，而不是立即决议。这很容易搬起石头砸 自己的脚! ES6 应该指定它完成或拒绝，抑或只是抛出某种同步错误。遗憾 的是，因为 Promise 库在时间上早于 ES6 Promise，它们不得已遗留了这个 问题，所以，要注意，永远不要递送空数组。

### all和race的变体
原生ES6Promise中提供了内建的Promise.all([ .. ])和Promise.race([ .. ])，但这些语义还有其他几个常用的变体模式。

- none([..])
	这个模式类似于all()，不过完成和拒绝的情况互换了，所有的Promise都要被拒绝，即拒绝转化为完成值，反之亦然。
- any([..])
	它只需要一个完成即可，其他会忽略。
- first([..])
	这个模式类似于any()，只要第一个Promise完成，它就会忽略后续的任何拒绝和完成。
- last([..])
	这个模式类似于first()，但却只有最后一个完成胜出。
	

## Promise API概述
### new Promise()构造器
```js
var p = new Promise(function (resolve, reject) {
  // resolve(..)用于决议/完成这个promise
  // reject(..)用于拒绝这个promise
});
```

Promise()构造器必须和new一起使用，并且必须提供一个回调函数，这个回调是同步的，立即调用的。这个函数接受两个函数回调，用以支持promise的决议，通常我们把这两个函数称为 resolve和reject。

### Promise.resolve()和Promise.reject()
分别创建一个已完成的Promise和已拒绝的Promise。

### then()和catch()
 每个 Promise 实例(不是 Promise API 命名空间)都有 then(..) 和 catch(..) 方法，通过 这两个方法可以为这个 Promise 注册完成和拒绝处理函数。Promise 决议之后，立即会调用 这两个处理函数之一，但不会两个都调用，而且总是异步调用。
 
 then(..) 接受一个或两个参数:第一个用于完成回调，第二个用于拒绝回调。如果两者中 的任何一个被省略或者作为非函数值传入的话，就会替换为相应的默认回调。默认完成回 调只是把消息传递下去，而默认拒绝回调则只是重新抛出(传播)其接收到的出错原因。
 
then(..) 和 catch(..) 也会创建并返回一个新的 promise，这个 promise 可以用于实现 Promise 链式流程控制。如果完成或拒绝回调中抛出异常，返回的 promise 是被拒绝的。如 果任意一个回调返回非 Promise、非 thenable 的立即值，这个值会被用作返回 promise 的完 成值。如果完成处理函数返回一个 promise 或 thenable，那么这个值会被展开，并作为返回 promise 的决议值。

### Promise.all() 和 Promise.race()
ES6PromiseAPI静态辅助函数Promise.all()和Promise.race()都会创建一个 Promise 作为它们的返回值。这个 promise 的决议完全由传入的 promise 数组控制。

对Promise.all()来说，只有传入的所有promise都完成，返回promise才能完成。 如果有任何 promise 被拒绝，返回的主 promise 就立即会被拒绝(抛弃任何其他 promise 的 结果)。如果完成的话，你会得到一个数组，其中包含传入的所有 promise 的完成值。对于 拒绝的情况，你只会得到第一个拒绝 promise 的拒绝理由值。这种模式传统上被称为门: 所有人都到齐了才开门。

对Promise.race()来说，只有第一个决议的promise(完成或拒绝)取胜，并且其 决议结果成为返回 promise 的决议。这种模式传统上称为门闩:第一个到达者打开门闩通 过。

## Promise局限性
### 顺序错误处理
 本章前面已经详细介绍了适合 Promise 的错误处理。Promise 的设计局限性(具体来说，就 是它们链接的方式)造成了一个让人很容易中招的陷阱，即 Promise 链中的错误很容易被 无意中默默忽略掉。

关于 Promise 错误，还有其他需要考虑的地方。由于一个 Promise 链仅仅是连接到一起的 成员 Promise，没有把整个链标识为一个个体的实体，这意味着没有外部方法可以用于观 察可能发生的错误。

如果构建了一个没有错误处理函数的 Promise 链，链中任何地方的任何错误都会在链中一 直传播下去，直到被查看(通过在某个步骤注册拒绝处理函数)。

### 单一值
根据定义，Promise只有一个完成值或拒绝理由。
这在简单的例子中，可能不是问题，但在更复杂的场景中，你可能发现这种局限了。

一般的建议是，构造一个封装值（数组或对象）来保持这样的多个信息。这种方式可以解决问题，但需要每一步都要封装和解封对象，实在不优雅。

### 单决议
Promise最本质的一个特征是，Promise只能被决议一次。

### 无法取消的Promise
 一旦创建了一个 Promise 并为其注册了完成和 / 或拒绝处理函数，如果出现某种情况使得这个任务悬而未决的话，你也没有办法从外部停止它的进程。
 
 ### Promise性能
把基本的基于回调的异步任务链与 Promise 链中需要移动的部分数量进行比较。很显然， Promise 进行的动作要多一些，这自然意味着它也会稍慢一些。

更多的工作，更多的保护。这些意味着 Promise 与不可信任的裸回调相比会更慢一些。这 是显而易见的，也很容易理解。

