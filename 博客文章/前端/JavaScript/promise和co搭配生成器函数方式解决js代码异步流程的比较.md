---
title: promise和co搭配生成器函数方式解决js代码异步流程的比较
date: 2016-07-24 18:50:59
tags: [js,es6]
categories:
- 技术博客
- 原创
---

在es6中引入的原生Promise为js的异步回调问题带来了一个新的解决方式，而TJ大神写的co模块搭配Generator函数的同步写法，更是将js的异步回调带了更优雅的写法。
今天我想对比一下这两种方式，来看看这两种方式的区别以及优劣。

<!-- more -->

我们先抽象几个操作：
以做饭为例，我们先去买菜，回来洗菜，刷碗，烧菜，最后才是吃。定义如下方法：

```javascript
var buy = function (){};  //买菜，需要3s
var clean = function(){};   //洗菜，需要1s
var wash = function(){};    //刷碗，需要1s
var cook = function(){};    //煮菜，需要3s
var eat = function () {};   //吃饭，2s，最后的一个步骤。
```


在实际中，我们可能这样，先去买菜，然后洗菜，然后开始烧菜，烧菜的同时，刷碗，等碗刷完了，菜煮好了，我们才开始吃饭。也就是，煮菜和刷碗是并行的。

## Promise方式

```javascript
var begin = new Date();
buySomething().then((buyed)=>{
    console.log(buyed);
    console.log(new Date()-begin);
    return clean();
}).then((cleaned)=>{
    console.log(cleaned);
    console.log(new Date()-begin);
    return Promise.all([cook(),wash()]);
}).then((value)=>{
    console.log(value);
    console.log(new Date()-begin);
    return eat();
}).then((eated)=>{
    console.log(eated);
    console.log(new Date()-begin);
}).catch((err)=>{
    console.log(err);
});
```

输出结果：

```
菜买到啦
3021
菜洗乾淨了
4023
[ '飯菜煮好了，可以吃飯了', '盤子洗乾淨啦' ]
7031
飯吃完了，好舒服啊
9034
```

Promise里有个all()方法，可以传递一个promise数组，功能是当所有promise都成功时，才返回成功。上面的例子，我们就将 cook()和wash()放到all()方法，模拟两个操作同时进行。从时间上来看，先去买菜，耗时3s，洗菜耗时1s，输出4023，刷碗和煮菜同时进行，以耗时长的煮菜3s，输出7031，最后吃饭2s，输出9034。

Promise的优势就是，可以随意定制Promise链，去掌控你的流程，想要同步的时候，就使用Promise链，想要异步，就使用Promise.all()，接口也很简单，逻辑也很简单。

## co+Generator搭配使用

```javascript
let begin = new Date();
co(function* (){
    let buyed = yield buySomething();
    console.log(buyed);
    console.log(new Date() - begin);
    let cleaned = yield clean();
    console.log(cleaned);
    console.log(new Date() - begin);
    let cook_and_wash = yield [cook(),wash()];
    console.log(cook_and_wash);
    console.log(new Date() - begin);
    let eated = yield eat();
    console.log(eated);
    console.log(new Date() - begin);
});
```

输出：

```
菜买到啦
3023
菜洗乾淨了
4026
[ '飯菜煮好了，可以吃飯了', '盤子洗乾淨啦' ]
7033
飯吃完了，好舒服啊
9035
```

从代码上，我们可以看出，使用co+Generator函数的写法，更趋向于同步代码的写法，具体代码是怎么执行的，大家可以研究一下es6的Generator函数。而且yield也可以接收一个数组，用来异步执行两个方法。代码上更精炼，更符合逻辑。
当前来说，co模块+Generator函数是一个比较好的改善“回调地狱”的优美的解决方案。

而且，这种方式比Promise优的一点是，Promise在实际操作中可能需要嵌套，例如我上一篇博客中<<es6中promise的研究>>中的例子一样，如果使用co+generator方式，则可以减少嵌套，在代码逻辑上更清晰，也不会让人思维混乱。

如可以改写为如下：

```javascript
var getColl = (collname, db) => {
  return new Promise(function(resolve,reject){
    co(function* (){
      var coll = yield db.createCollection(collname,{capped: true,size: 11800000,max: 5000});
      var stats = yield coll.stats();
      if(stats.count == 0){
        var inserted = yield coll.insert({coll: collname,create_time: new Date()});
      }
      resolve(coll);
    }).catch(function(err){
      reject(err);
    });
  });
};
```

这段代码在逻辑上，看起来就比之前用纯Promise实现的清爽些，逻辑上不混乱。

## es7的async和await
es7提供了async函数和await,这里和co+Generator函数方式是一样的，都是基于Promise实现的。这里async函数可以看作是Generator函数的语法糖，await可以看作是yield的语法糖。
co模块可以看作是Generator函数的执行器，而es7中，async函数自带执行器，这样就不必引用第三方的co模块了。
更好的语义，async和await，比起星号和yield，语义更清楚了。async表示函数里有异步操作，await表示紧跟在后面的表达式需要等待结果。
更广的适用性。 co模块约定，yield命令后面只能是Thunk函数或Promise对象，而async函数的await命令后面，可以是Promise对象和原始类型的值（数值、字符串和布尔值，但这时等同于同步操作）。
返回值是Promise。async函数的返回值是Promise对象，这比Generator函数的返回值是Iterator对象方便多了。你可以用then方法指定下一步的操作。
可以说，es7的async和await才是解决回调地狱的终极大招，虽然现在还不能以原生代码编写，但是可以使用es7编写代码，然后使用babel转译成es5代码。
