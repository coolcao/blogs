---
title: es6初探
date: 2016-08-02 20:32:51
tags: [js,es6]
categories:
- 技术博客
- 原创
---

在es6中新添加了一个Promise对象，其实这并不是“新”的概念，很早之前就有了promise规范，在es6之前有很多第三方库对其做了实现，es6对其进行原生实现。Promise规范的提出是为了解决异步编程中回调函数的“滥用”。

## promise的三种状态：
* pending 等待状态
* resolved 完成状态
* rejected 拒绝状态
promise的三种状态，只能是pending->resolved或者pending->rejected，不能有其他类型的状态转换，并且状态一旦发生转换，就不再发生变化。

<!-- more -->

## then方法
promise **必须** 提供一个then方法，then方法的返回值是一个 **promise**，then方法接收两个参数:

```js
promise.then(onResolved,onRejected);
```

* 这两个参数是可选的。
* onResolved是成功处理函数。如果不是函数，将被忽略
* onRejected是被拒绝函数，如果不是函数，将被忽略。
onResolved函数第一个参数为promise的最终值，onRejected函数的第一个参数是promise的拒绝原因。这里强调第一个是因为，js语言的“不严谨性”，你可以传多个参数，但只处理第一个参数。正常情况下，正常人写的promise的这两个函数，也只会有一个参数。
来看一个例子：

```javascript
var doSomething = function () {
    return new Promise(function (resolve, reject) {
        setTimeout(function () {
            resolve('第一个promise,2秒后出现');
        }, 2000);
    });
};
doSomething().then(function (resolved) {
    console.log(resolved);
},function(err){
    console.log(err);
});
```

这是一个简单的promise例子，doSomething()函数返回一个promise，其then方法接收两个函数，一个处理成功逻辑，一个处理失败逻辑。这里只是将成功终值和错误打印出来。

### 问题1.then方法的返回也是一个promise，那么对这个promise再调用then方法会怎样？
将上面代码修改一下如下

```javascript
doSomething().then(function (resolved) {
    console.log(resolved);
},function(err){
    console.log(err);
}).then(function (value) {
    console.log(value);
});
```

输出结果：

```
第一个promise,2秒后出现
undefined
```

这里第二个then方法调用的时候，打印出 undefined
再看一个例子：

```javascript
doSomething().then(function (resolved) {
    console.log(resolved);
    return '999';
},function(err){
    console.log(err);
}).then(function (value) {
    console.log(value);
});
```

这里的输出：

```
第一个promise,2秒后出现
999
```

有点意思，我们再看一个例子：

```javascript
doSomething().then(function (resolved) {
    console.log(resolved);
    return new Promise(function(resolved,reject){
        setTimeout(function(){
            resolved('then方法返回的promise,又得3秒');
        },3000);
    });
},function(err){
    console.log(err);
}).then(function (value) {
    console.log(value);
});
```

输出结果：

```javascript
第一个promise,2秒后出现
then方法返回的promise,又得3秒
```

第二个例子和第三个例子的返回，初看上去是一样的，但其运行流程却是不一样的。第二个返回一个常量字符串 999 ,等第一个promise执行完毕，即2秒打印出 “第一个promise,2秒后出现”后立即会输出 999 。第三个例子里，then方法返回了一个promise，于是，第一个promise2秒后输出“第一个promise,2秒后出现”，返回另一个promise，然后再3秒输出“then方法返回的promise,又得3秒”。

 对于一个promise的then方法，我们可以有三种返回：

* return 一个同步的值（或undefined）,第一二两个例子
* return 一个promise
* throw 一个异常
throw一个异常将由catch()方法捕捉。

## Promise.race方法和Promise.all方法
all()方法和race()方法都接收一個promise對象的數組作爲參數。all()方式是，當這個數組裏的所有promise對象全部變爲resolve狀態的時候，它才會去調用.then()方法。而race()方法是當數組中的promsie對象，只要有一個變爲resolved狀態的話，就會繼續調用.then()方法。
在邏輯上，all類似於 “邏輯與” 操作，而race類似於“邏輯或”操作，但又不盡相同，看下面的例子。
### 例子1
```javascript
var p1 = new Promise(function(resolve,reject){
    setTimeout(function(){
        resolve('p1');
    },3000);
});
var p2 = new Promise(function(resolve,reject){
    setTimeout(function(){
        reject(new Error('what is a fuck error'));
    },1000);
});

Promise.all([p1,p2]).then(function(value){
    console.log(value);
}).catch(function(err){
    console.log('onCatched:'+err);
});
```

上面這個例子中，毫無疑問的，輸出結果是`onCatched:Error: what is a fuck error`。但有一個有意思的現象是，由於p1的延時是3秒，p2的延時是1秒，因此，輸出最終的現象是`1秒後輸出onCatched:Error: what is a fuck error，然後又2秒後，程序退出`其實原因很簡單，由於數組中的promise是同時執行的，因此p1,p2同時執行，當1秒後，p2執行完畢，並被拒絕，這時的reject被catch到，直接輸出`onCatched:Error: what is a fuck error`，而此時的p1還未執行完畢，等到又2秒後才執行完畢，但此時已無濟於事，因爲p2的reject已被捕捉到，即使p1執行完成後，也不會在觸發then()方法。

### 例子2
```javascript
var p1 = new Promise(function(resolve,reject){
    setTimeout(function(){
        resolve('p1');
    },1000);
});
var p2 = new Promise(function(resolve,reject){
    setTimeout(function(){
        reject(new Error('what is a fuck error'));
    },3000);
});

Promise.race([p1,p2]).then(function(value){
    console.log(value);
}).catch(function(err){
    console.log('onCatched:'+err);
});
```
這個例子中，最終的輸出結果是 打印 p1。是不是感覺很簡單啊？繼續
### 例子3
```javascript
var p1 = new Promise(function(resolve,reject){
    setTimeout(function(){
        resolve('p1');
    },3000);
});
var p2 = new Promise(function(resolve,reject){
    setTimeout(function(){
        reject(new Error('what is a fuck error'));
    },1000);
});

Promise.race([p1,p2]).then(function(value){
    console.log(value);
}).catch(function(err){
    console.log('onCatched:'+err);
});
```

這個例子中，和例子2相比，調換p1和p2的setTimeout的時間，這時的輸出結果卻和例子2大相徑庭，這個例子中最終輸出`onCatched:Error: what is a fuck error`。
對比例子2和3可以發現，race方法和邏輯或還是不同的，數組中的promise，哪個的狀態先改變，則執行相應的處理方法。例如2中，p1先resolved，則執行then()方法，例子3中，p2先rejected，那麼就執行catch()方法捕捉異常。
其實從方法名上，我們可以看出，race的意思是競速，那麼從語義上來說，誰先改變狀態，就按誰的算。


## 參考資料
* [we have a problem with promises](https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html)
* 中文翻譯版：[你真的會用promise嗎？](http://fex.baidu.com/blog/2015/07/we-have-a-problem-with-promises/)
* [JavaScript Promises ... In Wicked Detail](http://www.mattgreer.org/articles/promises-in-wicked-detail/)
* 中文翻譯版：[Javascript Promise探微](http://www.html-js.com/article/Promise-translation-JavaScript-Promise-devil-details)
* MDN（火狐開發者中心）:[MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise)
* promise官網：[promise官網](https://www.promisejs.org/)
* [Javascript Promise迷你書](http://liubin.org/promises-book/)
* [一步一步實現Promise，可做參考學習的文章](https://github.com/xieranmaya/blog/issues/3)
