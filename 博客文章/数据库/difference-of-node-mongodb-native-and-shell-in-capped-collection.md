---
title: node-mongodb-native原生驱动在固定集合上的坑
date: 2016-12-13 20:39:09
tags: [mongodb,nodejs]
categories:
- 技术博客
- 原创
---

我们都知道， mongodb的固定集合，一旦插入数据后，再进行修改数据，会有限制：文档的大小不能改变，只能按照原来文档的大小进行修改。
我在实际项目中，遇到这么一个问题，找了好久才找到问题根源。

<!--more-->

## 整型值无法更新

有一个固定集合的文档，其中有一个status字段，用于标记状态：
使用mongo shell插入了一条记录用于测试：

```json
{status:0}
```

在nodejs代码中，我读到这条记录，要将此记录status改为1:

```js
collection.update({_id:doc._id},{$set:{status:1}});
```

按理来说，这应该是能修改成功的，因为其大小并没有改变，只是由0改为1而已。
但是，事实却修改失败了：

```
{ MongoError: Cannot change the size of a document in a capped collection: 53 != 49
    at Function.MongoError.create (/Users/coolcao/mycode/coolcao/test/node_modules/mongodb-core/lib/error.js:31:11)
    at toError (/Users/coolcao/mycode/coolcao/test/node_modules/mongodb/lib/utils.js:114:22)
    at /Users/coolcao/mycode/coolcao/test/node_modules/mongodb/lib/collection.js:1047:67
    at /Users/coolcao/mycode/coolcao/test/node_modules/mongodb-core/lib/connection/pool.js:455:18
    at _combinedTickCallback (internal/process/next_tick.js:67:7)
    at process._tickCallback (internal/process/next_tick.js:98:9)
  name: 'MongoError',
  message: 'Cannot change the size of a document in a capped collection: 53 != 49',
  driver: true,
  index: 0,
  code: 10003,
  errmsg: 'Cannot change the size of a document in a capped collection: 53 != 49' }
```

这个错误很让人诧异，字段值从0改为1，文档的大小并没有改变啊，为什么会失败呢？

从mongo的报错信息来看，文档是由53变成了49，少了4个字节。。。

查看日志，最终执行的更新命令没有错，修改的就是整型的1，也没有转换为字符串，也没有添加额外的其他字符之类的。

分析一下错误，文档由53变成了49，少了4个字节，细想一下，在js中并没有细分整型和浮点型，数字类型就只有一个Number类型，其是按照浮点型进行存储的。
不管是整数还是小数，都是按照64位进行存储，即8个字节。这里少了4个字节，猜想会不会是mongodb的js驱动做了手脚？

仔细上网搜了下，在stackoverflow上也有人提到过相关的问题，[如何将mongo中js驱动的数字类型强制转换为Double类型](http://stackoverflow.com/questions/14382346/forcing-javascript-numbers-to-double-in-mongodb-document)，原来nodejs原生驱动，对数字类型做了优化。
将数字类型细分成Int,Long,Double类型，从源码中我们也可以看出端倪：

![](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/mongo_data_type.png)

如图框中的部分，已然定义了Int32,Long,Double三种数字类型。

现在问题基本了然了。

nodejs的驱动在做插入时，将整数转换为Int32类型，使用32位存储，即4个字节。而mongo shell在做插入时，很忠实的使用了js的64位Number类型，8个字节，进行插入的，因此，这里正好相差4个字节。
更新时，由8个字节更新为4个字节，在mongo 3.0之前的版本，还允许这么做，3.0之前只规定固定集合大小不能由小改到大，并未规定不能由大改到小。
从3.0版本开始限制更严格了，**固定集合不能修改文档大小。不管改大还是改小。**
所以，也就出现了上面的一个问题。

这里算不算是nodejs驱动优化的一个bug，也不能算是，因为它考虑的是优化空间与速度。

造成这个问题的根本原因在于，在插入数字类型的值时，使用了两种不同的客户端，mongo shell和nodejs驱动，而这两种客户端对于数字的处理出现了差异。

所以清楚了这一点，在做关于mongodb固定集合的其他工作时，就有的放矢了，统一数据源录入客户端即可。

如果说，由于各种原因不能统一的话，那么最好是将数字类型改为字符串类型。即`{status:'0'}`。这样也是OK的。其实对于这种用来标识状态，非计算用的字段，最好还是使用字符串类型。

## 固定集合游标不移动
mongodb的固定集合游标查询，可以模拟队列，做一些消息队列服务。
但是在实施中发现一个问题，由于固定集合需要提前创建，我们在启动服务时，创建完固定集合后开始监听游标。
但是在插入数据时，游标却无法监听到新插入的数据。
重启mongo服务，新插入的数据就可以被监听到。
原来对于一个空的集合，无法创建游标，所以当你启动服务，创建一个游标时，游标并没有创建成功，自然无法监听数据。
因此，我们在创建完固定集合后，要先插入一条记录，随便一条记录即可。当插入完记录后，再创建游标，这个时候就可以创建成功。当有新数据插入时，就会自动监听到，不会有问题了。

## 小结
使用固定集合时，注意以下几点：

* tailable cursors不使用索引
* 查询时，默认顺序是按照插入順序，如果想使用倒序，使用 find().sort($natural:-1);
* 游标在以下几种情況下会死掉或不可用
  * 沒有返回
  * 返回的文档在集合末尾，并且刪除了文档
