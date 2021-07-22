使用async/await写代码很长一段时间了，在之前，用 co+生成器函数+yield也有一段时间，用起来很爽，但却没有仔细了解其中的原理。
为啥异步的代码，使用async/await或者Generator+yield能写成“同步”的？co又是啥，为啥Generator+yield要用co进行包裹？

Generator+yield能够实现同步方式书写异步代码，离不开co，简单说来，co是generator函数的执行器，用来执行generator函数。

co的代码很简练，总共239行，包括注释空格，而其中的“核心”代码，可能只需要区区不到20行。

下面的代码是我提取的co的核心，去掉了参数校验的判断，参数类型转换等的代码，但用来说明其核心原理是足够了，而且是可以运行的。

```js
const co = function(gen) {
  return new Promise((resolve, reject) => {
    const it = gen();
    const next = function(ret) {
      if (ret.done) return Promise.resolve(ret.value);
      const value = Promise.resolve(ret.value);
      value.then(onFulfilled, onRejected);
    }
    const onFulfilled = function(res) {
      const ret = it.next(res);
      next(ret);
    }
    const onRejected = function(err) {
      reject(err);
    }
    onFulfilled();
  });
}
```

这是一个最精简的co，我们来写个例子来看看co是如何运行的。

```js
const getUser = function() {
  return new Promise((resolve, reject) => {
    setTimeout(function() {
      resolve('coolcao');
    }, 1000);
  });
}

co(function*() {
  const s = yield 'abc';
  console.log(s);
  const u = yield getUser();
  console.log(u);
}).then().catch(err => {
  console.log('==========');
  console.log(err);
});
```

co是一个函数，接收一个参数 `gen` ，该参数类型是一个generator函数，返回一个Promise。
所以co调用完后可以调`co().then().catch()`。

`const it = gen();`

因为参数gen是一个generator函数，执行后会返回一个遍历器iterator。
这里顺便说一下，这里对generator以及iterator需要了解，如果还不清楚的到[阮一峰写的es入门学习](http://es6.ruanyifeng.com/)。
生成器执行后返回的遍历器，每次调用it.next()都会返回生成器函数yield后面的值，例如，上面的例子中，调用it.next()将返回`{ value: 'abc', done: false }`，第二次调用将返回`{ value: Promise { <pending> }, done: false }`，第三次调用将返回`{ value: undefined, done: true }`。函数里只有两个yield，因此第三次调用时，value为undefined，done为true表示遍历器以遍历完成。

继续看co的代码。
接下来分别定义三个函数：next用来处理it的next()方法返回值，onFulfilled用来处理成功结果，onRejected用来处理错误。先不一一说明，继续看代码。

然后就是直接调用`onFulfilled`函数处理。那么该函数做了啥事呢？
`const ret = it.next(res)`，这里调用遍历器的next()方法，获取到 yield后面的值，以`{ value: 'abc', done: false }`这种结构返回。

然后调用上面定义的next()函数来处理该返回值。

next()函数先是判断遍历器是否已完成遍历，也就是是否已处理了所有的生成器函数的yield后面的值。

这里很显然不是。接着走代码。

`const value = Promise.resolve(ret.value);`

这里取到ret中的value，并将其转换为Promise。因为我们知道 yield 后面可以接返回Promise的异步函数，或者接一个同步的值。这里就是为了处理如果yield后面是同步的值，将其转换为Promise，后面统一处理Promise.

value是一个Promise，直接调`.then()`方式来处理该Promise.

处理的时候，将onFulfilled, onRejected作为成功和失败的参数传入。

好了，对应着例子中第一个yield代码`const s = yield 'abc';`来看。
当co执行到next()方法的 `value.then(onFulfilled, onRejected);`时，这里的value是一个包含着'abc'的Promise，即`Promise.resolve('abc')`。
调用then()传入成功处理函数onFulfilled，再来看该函数的执行。

```js
const onFulfilled = function(res) { // 这里res为 ‘abc’
  const ret = it.next(res);         // 这里 ret 应为 { value: Promise { <pending> }, done: false }
  next(ret);
}
```
执行 onFulfilled 时， 参数 res 为 'abc'，然后再调用遍历器的next()取得下一个yield后面的值。
但是在调用it.next()时传入一个参数，将 'abc' 传入，这是什么意思呢？

这里需要说一下这是generator函数的一块知识。

it.next()调用的时候，是“取得”yield后面的值，以 `{ value: 'abc', done: false }`这种形式，前面说过了。如果it.next()调用时，传入参数，则该参数将作为`上一次`yield的返回值。这里可能有点绕，看例子：

```js
const it = (function*() {
  const s = yield 'abc';
  console.log(s);
  const u = yield getUser();
  console.log(u);
})();
console.log(it.next(1));
console.log(it.next(2));
console.log(it.next(3));

//{ value: 'abc', done: false }
//s = 2
//{ value: Promise { <pending> }, done: false }
//u = 3
//{ value: undefined, done: true }
```
第一次调用it.next(1)时，传入1作为参数，1作为上一次yield的返回值，由于这里是第一次调用，对应第一个yield，因此不存在上一个yield，所以这里1被丢弃。

第二次调用it.next(2)，传入的2作为参数，2作为上一次yield的返回值，即 `yield 'abc' === 2`，因此，这里s最后被赋值成2，从输出的结果也看出s===2.

同样，第三次调用it.next(3)，传入3作为参数，3作为上一个yield的返回值，即 `yield getUser() === 3`，因此 u === 3。如果调用 it.next()时不传任何参数，那么 yield 的返回值为 undefined，切记。

再回到上面co代码部分。

`value.then(onFulfilled, onRejected)`的成功处理函数onFulfilled中，`const ret = it.next(res);`传入的res为'abc'，即第一个`yield 'abc'`返回的值就是 'abc'，即最后 s 为 'abc'。
然后，再继续调用next()函数处理取到的第二个yield后面的值。流程类似，可以看着代码走一边试试。

至此，co如何将generator和yield代码执行的过程都解释完了，可能有点乱，再来梳理一遍。

首先调用一下该generator函数，返回一个遍历器，该遍历器中的值为generator函数中每个yield后面的值。
然后将yield后面的值转换成Promise。这里不管同步异步，转换成Promise后都将变成异步。
然后调用promise.then()来处理promise包裹的值，将这个值作为上一个yield的返回值，传入遍历器it.next()方法。
在取到下一个yield后面的值后，再调用成功函数处理该值。
如此形成一个Promise链，即所有generator后面的值都将按顺序组装成一个promise链，每个异步操作(yield)都将是这个promise链中的一个promise，然后按顺序处理这个promise链，如此就达到了“同步”的效果。

最后的流程，将转变成如下形式：
```js
Promise.resolve('abc')
.then(res => {
  const s = res;
  console.log(s);
  return getUser();
}).then(res => {
  const u = res;
  console.log(u);
}).catch(err => {
  console.log('err');
})
```

花了一点时间，研究了一下co，其实es7中的async/await在本质上与此类似，await后面要求必须接 Promise，最后流程肯定也是转变成如上模式。只不过node 8在v8已经内置了一个`co`，不再需要额外的模块执行生成器函数了。
