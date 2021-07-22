---
title: 深入理解ES7的async/await
date: 2016-12-12 21:04:28
tags: [js,async/await,es7]
categories:
- 技术博客
- 原创
---


在最开始学习ES6的Promise时，曾写过一篇博文[《promise和co搭配生成器函数方式解决js代码异步流程的比较》](/2016/07/24/promise和co搭配生成器函数方式解决js代码异步流程的比较/)，文章中对比了使用Promise和co模块搭配生成器函数解决js异步的异同。

在文章末尾，提到了ES7的async和await，只是当时只是简单的提了一下，并未做深入探讨。

在前两个月发布的Nodejs V7中，已添加了对async和await的支持，今天就来对这个东东做一下深入的探究。以更加优雅的方法写异步代码。

<!--more-->
## async/await是什么

async/await可以说是co模块和生成器函数的语法糖。用更加清晰的语义解决js异步代码。

熟悉co模块的同学应该都知道，co模块是TJ大神写的一个使用生成器函数来解决异步流程的模块，可以看做是生成器函数的执行器。而async/await则是对co模块的升级，内置生成器函数的执行器，不再依赖co模块。同时，async返回的是Promise。

从上面来看，不管是co模块还是async/await，都是将Promise作为最基础的单元，对Promise不很了解的同学可以先深入了解一下Promise。

## 对比Promise,co,async/await
下面我们使用一个简单的例子，来对比一下三种方式的异同，以及取舍。

我们采用mongodb的nodejs驱动，查询mongodb数据库作为例子，原因是mongodb的js驱动已经默认实现了返回Promise，而不用我们单独去包装Promise了。

### 使用Promise链

```js
MongoClient.connect(url + db_name).then(db => {
    return db.collection('blogs');
}).then(coll => {
    return coll.find().toArray();
}).then(blogs => {
    console.log(blogs.length);
}).catch(err => {
    console.log(err);
})
```

Promise的then()方法可以返回另一个Promise，也可以返回一个同步的值，如果返回的是一个同步值，将会被包装成一个Promise。
上面的例子中，db.collection()将返回一个同步的值，即集合对象，但是被包装成Promise，将会透传到下一个then()方法。
上面一个例子，是使用的Promise链。
先连接数据库MongoClient.connect()返回一个Promise，然后在then()方法里获得数据库对象db，然后再获取到coll对象再返回。在下一个then()方法获得coll对象，然后进行查询，查询结果返回，逐层调用then()方法，形成一个Promise链。
在这个Promise链上，如果任何一个环节出现异常，都会被最后的catch()捕捉到。
可以说，这个使用Promise链写的代码，比层层调用回调函数更优雅，流程也更明确。先获得数据库对象，再获得集合对象，最后查询数据。
但是这里有个不怎么“优雅”的问题，在于，每一个then()方法获取的对象，都是上一个then()方法返回的数据。而不能跨层访问。
什么意思，就是说在第三个then(blogs => {})中我们只能获取到查询的结果blogs，而不能使用上面的db对象和coll对象。这个时候，如果要打印出blogs列表后，要关闭数据库db.close()怎么办？
这个时候，可以两种解决方法：
第一种是，使用then()嵌套。我们将Promise链打断，使之嵌套，犹如使用回调函数的嵌套一般：

```js
MongoClient.connect(url + db_name).then(db => {
    let coll = db.collection('blogs');
    coll.find().toArray().then(blogs => {
        console.log(blogs.length);
        db.close();
    }).catch(err => {
        console.log(err);
    });
}).catch(err => {
    console.log(err);
})
```

这里我们将两个Promise嵌套，这样在最后一个查询操作里面，就可以调用外面的db对象了。但是这中方式，并不推荐。原因很简单，我们从一种回调函数地狱走向了另一种Promise回调地狱。
而且，我们要对每个Promise的异常进行捕捉，因为Promise没有形成链。

还有一种方式， 是在每个then()方法里都将db传过来：

```js
MongoClient.connect(url + db_name).then(db => {
    return {db:db,coll:db.collection('blogs')};
}).then(result => {
    return {db:result.db,blogs:result.coll.find().toArray()};
}).then(result => {
    return result.blogs.then(blogs => {   //注意这里，result.coll.find().toArray()返回的是一个Promise，因此这里需要再解析一层
        return {db:result.db,blogs:blogs}
    })
}).then(result => {
    console.log(result.blogs.length);
    result.db.close();
}).catch(err => {
    console.log(err);
});
```

我们在每个then()方法的返回中，都将db及其每次的其他结果组成一个对象返回。请注意，如果每次的结果都是一个同步的值还好说，但是如果是一个Promise值，每一个Promise都需要多做一层解析。
例如上面的一个例子，第二个then()方法返回的`{db:result.db,blogs:result.coll.find().toArray()}`对象中，`blogs`是一个Promise，在下一个then()方法中，我们无法直接引用博客列表数组值，因此需要先调用then()方法解析一层，然后将两个同步值db和blogs返回。
注意，这里涉及到了Promise的嵌套，不过一个Promise只嵌套一层then()。
这种方式，也是很蛋疼的一个方式，因为如果遇到then()方法中返回的不是同步的值，而是Promise的话，我们需要多做很多工作。而且，每次都透传一个“多余”的db对象，在逻辑上也有点冗余。

但除此之外，对于Promise链的使用，如果遇到上面的问题，好像也没其他更好的方法解决了。我们只能根据场景去选择一种“最优”的方案，如果要使用Promise链的话。

鉴于Promise上面蛋疼的问题，TJ大神将ES6中的生成器函数，用co模块包装了一下，以更优雅的方式来解决上面的问题。

### co搭配生成器函数
如果使用co模块搭配生成器函数，那么上面的例子可以改写如下：

```js
const co = require('co');
co(function* (){
    let db = yield MongoClient.connect(url + db_name);
    let coll = db.collection('blogs');
    let blogs = yield coll.find().toArray();
    console.log(blogs.length);
    db.close();
}).catch(err => {
    console.log(err);
});
```

co是一个函数，将接受一个生成器函数作为参数，去执行这个生成器函数。生成器函数中使用`yield`关键字来“同步”获取每个异步操作的值。
上面代码在代码形式上，比上面使用Promise链要优雅，我们消灭了回调函数，代码看起来都是同步的。除了使用co和yield有点怪之外。

使用co模块，我们要将所有的操作包装成一个生成器函数，然后使用co()去调用这个生成器函数。看上去也还可以接受，但是ES的进化是不满足于此的，于是async/await被提到了ES7的提案。

### async/await
我们先看一下使用async/await改写上面的代码：

```js
(async function(){
    let db = await MongoClient.connect(url + db_name);
    let coll = db.collection('blogs');
    let blogs = await coll.find().toArray();
    console.log(blogs.length);
    db.close();
})().catch(err => {
    console.log(err);
});
```

我们对比代码可以看出，async/await和co两种方式代码极为相似。
co换成了async，yield换成了await。同时生成器函数变成了普通函数。
这种方式在语义上更加清晰明了，async表明这个函数是异步的，同时await表示要“等待”异步操作返回值。
async函数返回一个Promise，上面的代码其实是这样：

```js
let getBlogs = async function(){
    let db = await MongoClient.connect(url + db_name);
    let coll = db.collection('blogs');
    let blogs = await coll.find().toArray();
    db.close();
    return blogs;
};

getBlogs().then(result => {
    console.log(result.length);
}).catch(err => {
    console.log(err);
})
```

我们定义getBlogs为一个async函数，最后返回得到的博客列表最终会被包装成一个Promise返回，如上，我们直接调用getBlogs().then()方法可获取async函数返回值。

好了，上面我们简单对比了一下三种解决异步方案，下面我们来深入了解一下async/await。

## 深入async/await
### async返回值
async用于定义一个异步函数，该函数返回一个Promise。
如果async函数返回的是一个同步的值，这个值将被包装成一个理解resolve的Promise，等同于`return Promise.resolve(value)`。
await用于一个异步操作之前，表示要“等待”这个异步操作的返回值。await也可以用于一个同步的值。

```js
//返回一个Promise
let timer = async function timer(){
    return new Promise((resolve,reject) => {
        setTimeout(() => {
            resolve('500');
        },500);
    });
}

timer().then(result => {
  console.log(result);  //500
}).catch(err => {
    console.log(err.message);
});
```

```js
//返回一个同步的值
let sayHi = async function sayHi(){
  let hi = await 'hello world';
  return hi;  //等同于return Promise.resolve(hi);
}

sayHi().then(result => {
  console.log(result);
});
```
上面这个例子返回是一个同步的值，字符串'hello world'，sayHi()是一个async函数，返回值被包装成一个Promise，可以调用then()方法获取返回值。
对于一个同步的值，可以使用await，也可以不使用await。效果效果是一样的。具体用不用，看情况。
比如上面使用mongodb查询博客那个例子，`let coll = db.collection('blogs');`，这里我们就没有用await，因为这是一个同步的值。当然，也可以使用await，这样会显得代码统一。虽然效果是一样的。



### async函数的异常
```js
let sayHi = async function sayHi(){
    throw new Error('出错了');
}
sayHi().then(result => {
  console.log(result);
}).catch(err => {
    console.log(err.message);   //出错了
});
```
我们直接在async函数中抛出一个异常，由于返回的是一个Promise，因此，这个异常可以调用返回Promise的catch()方法捕捉到。

和Promise链的对比：
我们的async函数中可以包含多个异步操作，其异常和Promise链有相同之处，如果有一个Promise被reject()那么后面的将不会再进行。

```js
let count = ()=>{
    return new Promise((resolve,reject) => {
        setTimeout(()=>{
            reject('故意抛出错误');
        },500);
    });
}

let list = ()=>{
    return new Promise((resolve,reject)=>{
        setTimeout(()=>{
            resolve([1,2,3]);
        },500);
    });
}

let getList = async ()=>{
    let c = await count();
    let l = await list();
    return {count:c,list:l};
}
console.time('begin');
getList().then(result => {
    console.log(result);
}).catch(err => {
    console.timeEnd('begin');
    console.log(err);
});
//begin: 507.490ms
//故意抛出错误
```
如上面的代码，定义两个异步操作，count和list，使用setTimeout延时500毫秒，count故意直接抛出异常，从输出结果来看，count()抛出异常后，直接由catch()捕捉到了，list()并没有继续执行。

### 并行
使用async后，我们上面的例子都是串行的。比如上个list()和count()的例子，我们可以将这个例子用作分页查询数据的场景。
先查询出数据库中总共有多少条记录，然后再根据分页条件查询分页数据，最后返回分页数据以及分页信息。
我们上面的例子count()和list()有个“先后顺序”，即我们先查的总数，然后又查的列表。其实，这两个操作并无先后关联性，我们可以异步的同时进行查询，然后等到所有结果都返回时再拼装数据即可。

```js
let count = ()=>{
    return new Promise((resolve,reject) => {
        setTimeout(()=>{
            resolve(100);
        },500);
    });
}

let list = ()=>{
    return new Promise((resolve,reject)=>{
        setTimeout(()=>{
            resolve([1,2,3]);
        },500);
    });
}

let getList = async ()=>{
    let result = await Promise.all([count(),list()]);
    return result;
}
console.time('begin');
getList().then(result => {
    console.timeEnd('begin');  //begin: 505.557ms
    console.log(result);       //[ 100, [ 1, 2, 3 ] ]
}).catch(err => {
    console.timeEnd('begin');
    console.log(err);
});
```

我们将count()和list()使用Promise.all()“同时”执行，这里count()和list()可以看作是“并行”执行的，所耗时间将是两个异步操作中耗时最长的耗时。
最后得到的结果是两个操作的结果组成的数组。我们只需要按照顺序取出数组中的值即可。
