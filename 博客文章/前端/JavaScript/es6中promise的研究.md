---
title: es6中promise的研究
date: 2016-08-16 22:38:51
tags: [es6,js]
categories:
- 技术博客
- 原创
---

学习了es6有一段时间了，对promise的概念也有一定的了解，起先觉得概念上挺简单的，但是在实际操作中却发现，用起来却没那么顺利。
首先，我们先看一个问题，有一个国外的哥们写的一篇博客，关于promise的研究上开篇提出来的问题，大家有兴趣可以看一下原文：https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html

下面这四个调用方式，返回结果有什么不同，运行方式有什么不同。

```javascript
doSomething().then(function () {
  return doSomethingElse();
});

doSomething().then(function () {
  doSomethingElse();
});

doSomething().then(doSomethingElse());

doSomething().then(doSomethingElse);
```
<!-- more -->

*其中doSomething和doSomethingElse两个方法都返回promise对象*

具体的答案，大家可以从原博客中获取，这里不再赘述。在原文中，作者提到一个观点就是，如果你对这四个例子的运行结果以及运行流程都清楚的话，那么对promise的理解还算可以的了。
这四个例子对于理解promise确实挺有帮助的，但是实际情况是，比这例子要复杂的多，实际用起来，还是有不小的挑战的。

下面这是我写的一段代码，是nodejs查询mongodb的一个例子。

```javascript
var getColl = (collname, db) => {
  return db.createCollection(collname, {capped: true,size: 11800000,max: 5000}).then((coll) => {
    return coll.stats().then((stats) => {
      if (stats.count == 0) {
        console.log('初始化集合'+collname);
        return coll.insert({coll: collname,create_time: new Date()}).then((inserted)=>{
          console.log('初始化集合'+collname+'完成，已插入初始数据');
          return coll;
        });
      }else{
        return coll;
      }
    });
  });
};
```
这个方法的功能是，创建一个固定集合，如果这个固定集合里面没数据，那么先向固定集合里插入一条记录，然后返回这个Collection实例。也就是说，我获取到这个集合后，我要确保集合中一定存在数据，然后再返回集合实例。
说实话，这段代码写的很一般，各种return，很不适合阅读。稍有不慎，就迷糊了，我擦，这个到底返回了个啥？
我们一点一点来看。
首先，函数getColl()返回的是 `db.createCollection()`;这是mongodb的nodejs原生驱动的方法，返回的是一个Promise,也就是说，整个函数getColl()返回的是一个Promise。
那么，返回的这个Promise里的value到底包含的是什么数据呢？
接下来看，then((coll)=>{})这个then方法的回调中，我们获取到了固定集合的实例coll对象，然后，根据其stats，如果里面数据0条，插入一条记录，否则，直接返回coll对象。
好的，`if(stats.count==0)`这个if判断的两个分支，都是return 的coll对象，也就是说，coll.stats().then()返回的这个Promise里value的值就是coll，而这个Promise有被return 回去，由于Promise的“穿透”（这个词，可以参见国外那哥们博客中的**进阶错误#5:promise穿透**）,由于Promise链的性质，最底层的Promise会一层一层的返回到最上层，因此getColl()方法返回的就是，一个value为coll对象的Promise。
这里之所以有嵌套，是因为，我在获取到coll对象时，并不能立即返回，而是要根据coll的状态去做不同的处理，做完处理再返回coll对象，因此，只能用嵌套去做，不能用链的形式。暂时没发现其他的方式，如果各位看客有更好的解决方式，请给我一些建议，谢谢。

这只是一个很简单的例子，像上面这个例子，或者，我们需要两个promise结果才能继续下一步操作的情况，都会用到promise的嵌套，如果嵌套多了，在阅读上会给coding的人思维上很大的干扰。
虽然，promise在整个规范上，实现上可能很好理解，但是在实际使用的情况下，我们会发现，并没有想象中的那么饱满，所以，在使用Promise的时候，还是应该谨慎，一定在思路上，代码上理清楚。
