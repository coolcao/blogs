---
title: 学习笔记-你不知道的js-值和类型
date: 2016-12-03 21:33:49
tags: [js]
categories:
- 学习笔记
- 你不知道的js
---

## 内置类型

Javascript有七种内置类型：
* null
* undefined
* boolean
* number
* string
* object
* symbol(ES6新增)

> 除Object外，其他统称为“基本类型”

> 注意：typeof null === 'object' 这是js的一个bug，在js中二进制的前三位都是0的话会被判断为object类型，而null的二进制全是0，因此typeof操作符会返回object。这个bug可能永远不会修复。这里牵涉到太多太多的web系统，如果修复了，那么可能会引来更多的bug。

<!--more-->

## 值和类型
**JavaScript中的变量是没有类型的，只有值才有。变量可以随时持有任何类型的值。**

### undefined和undeclared
变量在未持有值的时候，此时typeof返回'undefined'
```js
var a;
typeof a;   // undefined
var b = 42;
var c;
b = c;
typeof b;   // undefined
typeof c;	// undefined
```

undefined和undeclared是两码事。
已在作用域中声明，但是未赋值的变量，是undefined的。
相反，还没有在作用域中声明的变量，是undeclared的。

```js
var a;
typeof a;	// undefined
typeof b;	// undeclared
```
> 请注意，虽然b未定义，但是可以使用 typeof 进行校验，并未报错，这是因为typeof有一个特殊的安全防范机制，这个机制很有用，后面会提到。

### typeof Undeclared
这个安全防范机制很重要，可以检测全局空间上有无变量定义。

例如，在程序使用全局变量DEBUG来控制是否开启调试模式，在输出调试信息之前，我们要判断DEBUG这个变量。顶层的全局变量声明，只有在debug.js中才有，而该文件只有在开发测试和调试时才会被加载到浏览器，生产环境不加载。
如果我们直接if判断，则会报错：
```js
// 这样会抛出错误
if (DEBUG) {
	console.log('debug');
}

// 这样是安全的
if (typeof DEBUG !== 'undefined') {
	console.log('debug');
}
```

> 虽然在浏览器中，全局变量可以通过window对象获取，但如果你的代码要执行在多种不同的浏览器或环境中，比如nodejs，那么最好不要使用window。而使用typeof便好很多。

## 数组
数组通过数字进行索引，但js的数组底层用的是对象，因此也可以使用包含字符串键值和属性，但这些并不计算在数组长度length内。
但是如果字符串键值能够被强制转换成十进制数字的话，它就会被当作数字索引来处理。
```js
let a = [1,2,3];
a.name = 'coolcao';
a.age = 23;
console.log(a.length);  //3
a[3] = '4';
console.log(a.length);  //4
```

**在数组中加入字符串键值属性并不是一个好主意，建议使用对象存储键值对，用数组来存放数字索引值**

### 类数组
类数组是一个对象，通过数字来索引值，而且有length属性指定长度的对象。比如：
```js
//类数组
var a = {'0':0,'1':1,'2':2,'3':3,length:3}
//不是类数组
var b = {'0':0,'1':1,'2':2,'3':3}
```
Javascript中常用的一个类数组是函数的 `arguments` 对象。
如果需要将类数组转换成真正的数组，在ES6中新添加了`Array.from()`函数。在ES6之前，可以使用 `Array.prototype.slice.call()`来转换。

## 字符串
字符串经常被当成字符数组。但是在Javascript中的字符串和字符数组并不是一回事，但有许多相似的方法。
他们有length属性，有indexOf()方法，concat()方法，而且字符串还可以通过call和apply调用数组的部分方法。
但无论如何，他们不是一回事。
javascript中字符串是不可变的，而数组是可变的。
如果需要经常以字符数组的方式来处理字符串的话，不如直接使用数组。这样就不用在字符串和数组之间来回折腾。可以在需要时用join('')将字符数组转换为字符串。

## 数字
javascript只有一种数值类型:number，包括整数和带小数的十进制数。javascript中整数就是没有小数的十进制数。比如 42.0即等同于整数42。
javascript使用的是双精度（即64位二进制）格式。

`42.toFixed(2)` 这种写法是错误的，引擎会将.误以为是小数点，因此得使用`(42).toFixed(2)`或者`42..toFixed(2)`，但为提高可读性，不要使用第二种，使用第一种。

二进制，八进制，十六进制建议都使用小写：0b,0o,0x，因为八进制中如果使用大写O和0容易混淆。

二进制浮点数最大的问题（不仅javascript，所有遵循IEEE754规范的语言都是如此）会出现如下情况：
```
0.1 + 0.2 === 0.3   //false
```
原因简单来说，是二进制浮点数中0.1和0.2并不是是非精确，他们相加后的结果并非刚好等于0.3而是0.30000000000000004。具体可以查看我之前写过的一篇博客[《js中0.1+0.2为什么不等于0.3》](http://coolcao.com/2016/10/12/js%E4%B8%AD0-1-0-2%E4%B8%BA%E4%BB%80%E4%B9%88%E4%B8%8D%E7%AD%89%E4%BA%8E0-3/)
那该如何判断呢？最常见的方法是设置一个误差范围，通常被称为“机器精度”，对javascript来说，这个值是 2^-52。
从ES6开始，该值定义在Number.EPSILON中，也可以为ES6之前的版本写polyfill:

```js
if(!Number.EPSILON){
    Number.EPSILON = Math.pow(2,-52);
}
```
可以使用Number.EPSILON来比较两个数字是否相等（在指定的误差范围内）：

```js
function numbersCloseEnoughToEqual(a,b) {
    return Math.abs(a - b) < Number.EPSILON;
}
var a = 0.1 + 0.2;
var b = 0.3;
numbersCloseEnoughToEqual(a,b); //true
numbersCloseEnoughToEqual(0.000000001,0.000000002);     //false
```

要检测一个数是整数，在ES6中可以使用Number.isInteger()方法，ES6之前的：

```js
if(!Number.isInteger){
    Number.isInteger = function (num) {
        return typeof num == 'number' && num % 1 == 0;
    }
}
```

### 32位有符号整数
虽然整数最大能够达到53位，但是有些数字操作（比如位操作）只适用于32位数字。

可以使用 `a | 0`来将数字转换为有符号32位数字。因为 `|`是位操作，只适用于32位整数（它只关心32位以内的值，其他数位将被忽略）。

## 特殊数值
### NaN
NaN是指不是值的数字，typeof NaN 返回 number。
NaN是一个特殊的值，它和自身不相等，是唯一一个非自反的值。即 NaN != NaN 。
我们可以使用isNaN()函数来判断，但是它有一个弊端，它会`检查参数是否不是NaN，也不是数字`：

```js
isNaN(NaN); //true
isNaN('abc');   //true
```

这个Bug一直存在，已经19年了。估计以后也不会修复。
从ES6开始，我们可以使用Number.isNan()函数进行判断:
```js
Number.isNaN(NaN);  //true
Number.isNaN('abc');//false
```
ES6之前我们可以：
```js
if(!Number.isNaN){
    Number.isNaN = function (num) {
        return (typeof num === 'number' && isNaN(num));
    }
}
```
我们可以根据NaN是js唯一一个和自身不相等的来判断：
```js
if(!Number.isNaN){
    Number.isNaN = function (num) {
        return num !== num;
    }
}
```

### 无穷数
* 正无穷：Infinity
* 负无穷：-Infinity

```js
var a = 1/0;    //Infinity
var b = -1/0;   //-Infinity
```

Infinity/Infinity是一个未定义的操作，结果是 NaN。
有穷正数除以Infinity结果是0，有穷负数除以Infinity结果是-0。

### 零值
javascript有一个常规的0和一个-0。
-0可以做常量，也可以是某些数学运算符的返回值。
```js
var a = 0 / -3;   //-0
var b = 0 * -3; //-0
```
加法和减法不会得到-0.
要判断-0，就要用到-Infinity:
```js
function isNegZero(n) {
    n = Number(n);
    return (n === 0) && (1/n === -Infinity)
}
isNegZero(-0);  //true
isNegZero(0/-3);//true
isNegZero(0);   //false
```

为什么会需要负0:
有些应用程序中的数据需要以级数形式来表示，比如动画帧的移动速度，数字的符号位用来代表其他信息，比如移动方向。此时如果有一个值为0的变量是去了它的符号位，它的方向信息就会丢失。所以保留0值的符号位可以防止类似情况。
