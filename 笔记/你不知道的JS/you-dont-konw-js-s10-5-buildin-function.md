---
title: 学习笔记-你不知道的js-原生函数
date: 2016-12-25 21:46:52
tags: [js]
categories:
- 学习笔记
- 你不知道的js
---

## 原生类型
常用的原生函数有:
-   String()
-   Number()
-   Boolean()
-   Array()
-   Object()
-   Function()
-   RegExp()
-   Date()
-   Error()
-   Symbol()——ES6 中新加入的!

原生函数可以当作构造函数来使用，但其构造出来的是**对象**，可能和我们想象的有所出入。

```js
var str = new String('abc');
typeof str;	// 是 Object, 不是 String
str instanceof String;	// true
Object.prototype.toString.call( a ); // "[object String]"
```


**通过构造函数构造出来的是封装了基本类型值的封装对象。**

## 封装对象
### 内部属性`[[class]]`
所有的typeof返回值为"Object"的对象都包含一个内部属性`[[class]]`,这个属性无法直接访问，一般通过 Object.prototype.toString()来查看。

```js
Object.prototype.toString.call( [1,2,3] );// "[object Array]" 
Object.prototype.toString.call( /regex-literal/i );// "[object RegExp]"
```
上面例子中，数组内部的`[[class]]`是Array，正则表达式是`[[RegExp]]`。

### 封装对象包装
由于基本类型值没有.length和.toString()这样的方法，需要封装对象才能访问到，此时JavaScript会自动为基本类型值包装一个封装对象。

```js
var str = 'abc';
a.length; 	// 3
a.toUpperCase();	// "ABC"
```

一般情况下，我们不需要直接使用封装对象，最好的办法就是让引擎自己决定什么时候使用封装对象，我们只需要直接使用基本类型原始值即可。

### 封装对象释疑
使用封装对象时需要注意，封装对象是一个对象，而不是原始值，在使用时一定要注意。

```js
var b = new Boolean(false);
if (b) {
	console.log('true');
} else {
	console.log('false');
}
// 最终打印 true
```

b是一个值为false的封装对象，在使用if进行判断时，由于b是对象，不为空，所以返回的是true，最终打印'true'。我们如果要使用其包装的原始值，要使用valueOf()方法获取其原始值。

> 一般不推荐直接使用封装对象，如上可能会引起不必要的bug，但偶尔也可能很有用，酌情考虑。

## 拆封
如果想要得到封装对象中的基本类型值，可以使用 valueOf() 函数。

## 原生函数作为构造函数
如非十分必要，一般不使用原生函数，因为他们会产生意想不到的结果。

直接使用字面量创建即可。

### Array()
```js
var a = new Array(1,2,3);
a;	// [1,2,3]
var b = [1,2,3];
b;	// [1,2,3]
```

**构造函数只有一个参数时，该参数会被当作数组的预设长度**

```js
var a = new Array(3);
a.length; 	// 3
```

> 这实在是一个，容易混淆并且出错的地方。
> 而且上面创建这个长度只有3的空数组，在不同浏览器和nodejs环境中，表现都不一致。
> 在这里，再强调一遍，**如非必要，不要使用原生构造函数创建对象。**

### Object(..)、Function(..) 和 RegExp(..)
同样，除非万不得已，否则尽量不要使用 Object(..)/Function(..)/RegExp(..)。

### Date(..) 和 Error(..)
相较于其他原生构造函数，Date(..) 和 Error(..) 的用处要大很多，因为没有对应的常量形式来作为它们的替代。

### Symbol(..)
ES6 中新加入了一个基本数据类型 ——符号(Symbol)。符号是具有唯一性的特殊值(并 非绝对)，用它来命名对象属性不容易导致重名。

> 符号并非对象，而是一种简单标量基本类型。

