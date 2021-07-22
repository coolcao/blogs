---
title: js数组拷贝
date: 2016-10-30 23:14:52
tags: [js,Array]
categories:
- 技术博客
- 原创
---

利用js原生已实现的方法，我们就可以不用自己写循环实现数组的拷贝复制。

原数组：
```js
let array1 = [1,'a',true,null,undefined];
```

<!--more-->

## slice()方法
```js
let c1 = array1.slice();
```

## concat()方法
```js
let cc1 = array1.concat();
```

## from()方法
```js
let fc1 = Array.from(array1);
```

## push()方法
```js
let pc1 = [];
Array.prototype.push.apply(pc1,array1);
```

## map()方法
```js
let mc1 = array1.map(function(item){
    return item;
});
```

以上几种方法都能实现数组的`浅拷贝`，即数组的每一项只能是原始类型的数据，如果数组的项包含引用类型，如数组（即js中的二维数组），对象等，以上方法复制的项只是引用。
还有一种方法是，使用json进行转换，先将数组序列化为json字符串，然后再将字符串转换成json对象即可。
## JSON转换
```js
let jsonc = JSON.parse(JSON.stringify(array1));
```
这种方法可以变相的实现`深拷贝`,但是这种方法也有其限制：
* 首先，数组中的项如果是`undefined`，那么转换后将变为`null`
* 如果数组的项为对象，那么对象之间不可相互引用。会造成循环引用，无法JSON序列化。

## 性能分析
以上几种方法都可以实现数组的拷贝，那么，每种方法的性能如何呢，我使用`console.time()`和`console.timeEnd()`跟踪当数组大小为100-10000000时，每个方法所用的时间。注意，这里数组每一项都是随机的5种原始值的一个，不包含引用类型。

|数组大小|forof|slice|concat|from|push|map|json|
|----|----|-----|----|----|-----|----|-----|
|100|0.593ms|0.038ms|0.034ms|0.404ms|0.054ms|0.193ms|0.042ms|
|1000|1.606ms|0.052ms|0.044ms|1.311ms|0.090ms|0.897ms|0.124ms|
|10000|2.272ms|0.097ms|0.093ms|3.294ms|0.145ms|1.845ms|0.772ms|
|100000|14.665ms|0.901ms|0.730ms|22.283ms|4.002ms|15.894ms|9.101ms|
|1000000|175.663ms|16.051ms|8.400ms|235.900ms|Maximum call stack size exceeded|144.058ms|97.946ms|
|10000000|1597.242ms|83.860ms|124.781ms|2425.540ms|Maximum call stack size exceeded|1424.344ms|1043.772ms|
> 当数组大小超过1000000时，push方式就挂了，报错：`Maximum call stack size exceeded`
