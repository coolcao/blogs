---
title: 学习笔记-你不知道的js-强制类型转换
date: 2016-12-05 06:37:56
tags: [js]
categories:
- 学习笔记
- 你不知道的js
---

将值从一种类型转换为另一种类型通常称为 *类型转换（type casting）* ，这是显式的情况。隐式的情况称为 *强制类型转换（coercion）*。
Javascript的强制类型转换总是返回标量 **基本类型值**，如字符串，数字和布尔值，不会返回对象和函数。
也可以这样区分：类型转换发生在静态类型语言的编译阶段，而强制类型转换则是发生在动态类型语言的运行时（runtime）。
<!--more-->
## 抽象值操作
ES规范定义了一些抽象操作（即仅供内部使用的操作）和转换规则，包括ToString,ToNumber,和ToBoolean,ToPrimitive。

**之前已经写过两篇关于ecma中类型转换的博客，可以直接参考：
[ecmasrcipt类型转换](http://coolcao.com/2016/08/12/ecma%E7%B1%BB%E5%9E%8B%E8%BD%AC%E6%8D%A2/)
[js中==和===的区别](http://coolcao.com/2016/08/06/js%E4%B8%AD-%E5%92%8C-%E7%9A%84%E5%8C%BA%E5%88%AB/)
**

### ToString
基本类型值的字符串转化规则：null转换为'null',undefined转换为'undefined'，true转换为'true'，数字的字符串化则遵循通用规则。
对于普通对象而言，除非自行定义，否则toString()（Object.prototpye.toString()）返回内部属性[[Class]]的值。例如[object Object]。
数组默认的toString()方法经过了重新定义，将所有单元字符串化后再用','连接起来。

#### JSON字符串化
所有安全的JSON值都可以用JSON.stringify()字符串化。安全JSON值是指能够呈现为有效JSON格式的值。
undefined,function,symbol和包含循环引用（对象之间相互引用，形成一个无限循环）的对象都不符合JSON结构标准。

```js
JSON.stringify(undefined);      //undefined
JSON.stringify(function(){});      //undefined
JSON.stringify([1,undefined,function(){},4]);   //'[1,null,null,4]'
JSON.stringify({a:2,b:function(){}});           //'{"a":2}'
```

对于循环引用的对象执行JSON.stringify()会出错。

如果对象中定义了toJSON()方法，JSON字符串化时会首先调用该方法，然后用它的返回值来进行序列化。
**如果要对含有非法JSON值的对象做字符串化，或者对象中的某些值无法被序列化时，就需要定义toJSON()方法来返回一个安全的JSON值。**


```js
var o = {};
var a = {
    b:42,
    c:o,
    d:function(){}
}
o.e = a;
//循环引用在这里会产生错误
//JSON.stringify(a)
//自定义JSON序列化
a.toJSON = function(){
    return {b:this.b}
}
JSON.stringify(a);  //'{"b":42}'
```

我们可以向JSON.stringify()传递一个可选参数replacer，它可以是数组或者函数，用来指定对象序列化过程中哪些属性应该被处理，哪些应该被排除。
如果replacer是一个数组，那么它必须是一个字符串数组，其中包含序列化要处理的对象的属性名称，除此之外其他的属性则被忽略。
如果replacer是一个函数，它会对对象本身调用一次，然后对对象中的每个属性各调用一次，每次传递两个参数，键和值，如果要忽略某个键就返回undefined，否则返回指定的值。

```js
var a  = {
    b:42,
    c:"42",
    d:[1,2,3]
}
JSON.stringify(a,['b','c']);    //'{"b":42,"c":"42"}'
JSON.stringify(a,function(k,v){
    if(k!=='c'){
        return v;
    }
});
//'{"b":42,"d":[1,2,3]}'
```

JSON.stringify()还有一个可选参数space，用来指定输出的缩进格式。space为正整数时，指定每一级缩进的字符数，它还可以是字符串，此时最前面的十个字符被用于每一级的缩进。

### ToNumber
有时我们需要将非数字值当作数字来使用，比如数学运算，为此，ES5规范在9.3节定义了抽象操作 `ToNumber`。
true转换为1，false转换为0,undefined转换为NaN,null转换为0.
对象，包括数组，会首先被转换为对应的基本类型值，如果返回的是非数字的基本类型值，则再遵循以上的规则将其强制转换为数字。

从ES5开始，使用Object.create(null)创建的对象`[[Prototype]]`属性为null，并且没有valueOf()和toString()方法，因此无法进行强制类型转换。

为了将值转换为相应的基本类型值，抽象操作 ToPrimitive(参见 ES5 规范 9.1 节)会首先 (通过内部操作 DefaultValue，参见 ES5 规范 8.12.8 节)检查该值是否有 valueOf() 方法。 如果有并且返回基本类型值，就使用该值进行强制类型转换。如果没有就使用 toString()的返回值(如果存在)来进行强制类型转换。

### ToBoolean
#### 假值
Javascript中的值可以分为两类：
* 可以被强制转换为false的值
* 其他（被强制类型转换为ture的值）

假值列表：
* undefined
* null
* false
* +0,-0,和NaN
* ''

假值的布尔强制类型转换结果为false.我们可以理解为，假值以外的值都是真值。
#### 假值对象
浏览器在某些特定的情况下，在常规的JavaScript语法基础上自己创建了一些外来值，这些就是“假值对象”。
假值对象看起来和普通对象并无二致，都有属性等，但将它们强制类型转换为布尔值时结果为false。
最常见的例子是document.all，它是一个类数组对象，包含了页面上的所有元素，有DOM提供给Javascript程序使用，以前是一个真正的对象，不过现在它是一个假值对象。
#### 真值
假值列表以外的值都是真值。
真值列表可以无限长，无法一一列举。我们只能用假值列表作为参考。

## 显式强制类型转换
显式强制类型转换是那些显而易见的类型转换。很多类型转换都属于此类。
我们在编码时，**尽可能地将类型转换表达清楚，以免留坑。类型转换表达越清晰，代码可读性越高，更容易理解。**

### 字符串和数字之间的转换
字符串和数字之间的转换是通过String()和Number()这两个内建函数来实现的。注意他们前面没有new关键字，并不创建封装对象。

```js
var a = 42;
var b = String(a);
var c = "3.14";
var d = Number(c);
b;	// "42"
d;	// 3.14
```
String(..)遵循前面讲过的ToString()规则，Number(..)遵循前面的ToNumber(..)规则。

除了String(..)和Number(..)，还有其他方法可以实现字符串和数字之间的显式转换：
```js
var a = 42;
var b = a.toString();
var c = "3.14";
var d = +c;
b;	// "42"
d;	// 3.14
```
a.toString()这是显式的转换，但其中也包含隐式转换。因为42是基本类型值，不包含toString()方法，因此JS引擎会自动为42创建一个封装对象，然后调用该对象的toString()方法。

+c 是 + 运算符的一元形式， +运算符显式的将c转换为数字，而非数字加法运算，也不是字符串拼接运算。
一元运算符 + 还有如下功能：
1. 日期显式转换为数字
一元运算符+可以将日期对象(Date)转换为数字，返回结果为Unix时间戳，以微秒为单位。
```
const d = new Date();
console.log(+d);			//1615971327473
console.log(d.getTime());	//1615971327473
console.log(Date.now());	//1615971327473
```
不建议使用+将Date对象转换为时间戳，可读性很差，建议使用Date.now()来获取当前时间戳，使用new Date(..).getTime()来获取指定日期的时间戳。



#### 奇特的~运算符
我们说过，位运算符只适用于32位整数，运算符会强制操作数使用32位格式。这个通过抽象操作ToInt32来实现的。
ToInt32首先执行ToNumber强制转换成Number，然后再执行ToInt32。
对于~,它首先将值强制类型转换为32位数字，然后执行位操作非。
对~还有另外一种诠释，源自早期的计算机科学与离散数学：~返回2的补码。
~x大致等同于-(x+1)。
在~(x+1)中唯一能够得到0的x值是-1.
-1是一个哨位值，哨位值是那些在各个类型中被赋予了特殊含义的值。
在c语言中，我们用-1来代表函数执行失败，用大于等于0来代表函数执行成功。
Javascript中indexOf()方法也遵循这一惯例，该方法在字符串中搜索指定的子字符串，如果找到就返回子字符串所在的位置，否则返回-1。
我们在检查字符串中是否包含子字符串时，会用到indexOf()方法：

```js
var a = 'hello world';
if(a.indexOf('lo') >=0 ){
    //
}
if(a.indexOf('lo') != -1){

}
```
这种写法不是很好，称为*抽象泄漏*，意思是在代码中暴露了底层的实现细节。这里是指用-1作为失败时的返回值，这些细节应该被屏蔽掉。

**~和indexOf()一起使用可以将结果强制类型转换为真假值**
如果indexOf()返回-1，~将其转换为假值0，其他一律转换为真值。

```js
var a = 'hello world';
~a.indexOf('lo');   //-4 ,真值
if(~a.indexOf('lo')){   //true
    //找到匹配
}
```

> 注：这种写法看上去很具有技巧，但对于JS基础并不强的开发者来说，可读性很差，有些人甚至不明白什么意思，所以，我本人还是更倾向于使用上面提到的 `a.indexOf('lo')!=-1`这种更明显的写法，可读性强。

~~x 能将值截除为一个32位整数。x|0也可以。

### 显式解析数字字符串
* 解析: parseInt(),parseFloat():从左到右解析，遇到非数字字符就停止。
* 转换：Number()：不允许出现非数字字符，否则就会失败并返回NaN。

#### 解析非字符串
```js
parseInt(1/0,19);
```
这个最后的结果是：18.
parseInt()第二个参数表示要解析进制。这里传19表示按照19进制进行解析。
之所以是18，原因很简单：**parseInt()如果传递的不是字符串，那么将被强制转换为字符串，然后再进行解析**。
这里，1/0的结果是Infinity。结果是Number类型，非字符串，将被转换成字符串'Infinity'，然后将其按照19进制进行解析。
在19进制中，第一个字符'I'被解析成18，后面'n'不能再继续解析，停止，最后得出结果18.


再来看一些看起来奇怪，但实际上解释得通的例子：

```js
parseInt(0.000000); //0
parseInt(0.0000008); //8，8来自'8e-7'
parseInt(false,16); //250 ,fa来自false
parseInt('0x10');    //16
parseInt('103',2);  //2
```

### 显式转换为布尔值
通常使用`Boolean()`和`!!`来显式转换为布尔值。以增强代码的可读性。

## 隐式强制类型转换
隐式强制类型转换指的是那些隐蔽的强制类型转换，副作用也不是很明显。
#### 字符串与数字之间的隐式强制类型转换
`+`运算符既能用于加法运算，也能用于拼接字符串。

```js
var a = "42";
var b = "0";
var c = 42;
var d = 0;

console.log(a+b);	// "420"
console.log(c+d);	//42
```
这里为什么会得到 "420" 和 42 两个不同的结果呢?通常的理解是，因为某一个或者两个操作数都是字符串，所以 + 执行的是字符串拼接操作。这样解释只对了一半，实际情况要复 杂得多。

ES规范的定义：

> AdditiveExpression:AdditiveExpression+MultiplicativeExpression
> * Let lref be the result of evaluating AdditiveExpression.
> * Let lval be ? GetValue(lref).
> * Let rref be the result of evaluating MultiplicativeExpression.
> * Let rval be ? GetValue(rref).
> * Let lprim be ? ToPrimitive(lval).
> * Let rprim be ? ToPrimitive(rval).
> * If Type(lprim) is String or Type(rprim) is String, then
>     * Let lstr be ? ToString(lprim).
>     * Let rstr be ? ToString(rprim).
>     * Return the String that is the result of concatenating lstr and rstr.
> * Let lnum be ? ToNumber(lprim).
> * Let rnum be ? ToNumber(rprim).
> * Return the result of applying the addition operation to lnum and rnum.

大致的流程是：
* 将`+`两边的表达式先计算值，然后将左右两边的计算值转换成原始值。
* 如果其中一个为字符串类型，那么将左右两边转换为字符串然后进行字符串拼接。
* 否则转换成数字类型，进行数字加法运算。

因此这里的`+`运算符就涉及到一个隐式强制类型转换。

```js
[]+{}   //[object Object]
{}+[]   //0 这里{}被解析成代码块
```

根据上面的规则，可以看出，其实`a+''`就是一个隐式转换，将a转换为字符串。它和前面介绍的显示转换String(a)有什么区别呢？

```js
const a = {
    val: 1,
    toString() {
        return 'a';
    },
    valueOf() {
        return this.val;
    }
};

console.log(a+'');			// 1
console.log(String(a));		// a
```
从上面的例子可以看出，还是有差别的。

我们再仔细看上面的ES规范中的流程，第一条有一个**将两边的计算值转换为原始值**，在转换为原始值时，会调用抽象方法 ToPrimitive ，然后再调用抽象方法 ToString()将其转换为字符串进行拼接。而String(a)是直接调用ToString()方法。

### 布尔值到数字的隐式强制类型转换
在将某些复杂的布尔逻辑转换为数字加法的时候，隐式强制类型转换能派上大用场。
```js
function onlyOne(a,b,c) {
    return !!((a && !b && !c) ||
        (!a && b && !c) || (!a && !b && c));
}
var a = true;
var b = false;
onlyOne( a, b, b ); // true
onlyOne( b, a, b ); // true
onlyOne( a, b, a ); // false
```

假设有一个函数onlyOne()，用来判断传入的参数是否只有一个为true，上面代码只有3个参数，代码就已经很晦涩难懂了，参数之间进行组合。如果参数再多，比如10个，100个呢？

这个时候，我们就可以使用隐式强制类型转换将布尔值转换为数字，用数字的形式进行计算，得出结果：

```js
function onlyOne() {
    var sum = 0;
    for (var i=0; i < arguments.length; i++) { // 跳过假值，和处理0一样，但是避免了NaN if (arguments[i]) {
        sum += arguments[i];
    }
    return sum == 1;
}
var a = true;
var b = false;
onlyOne( b, a );    //true
onlyOne( b, a, b, b, b ); // true
onlyOne( b, b );                // false
onlyOne( b, a, b, b, b, a );    // false
```

我们使用隐式强制类型转换，将布尔值true转换为1，false转换为0，然后进行累加，如果最后sum等于1，那么说明只有一个是true。

虽然这种场景在真实环境中很难遇到，但也说明了，如果在一些特定的场景下，如果活用隐式类型转换，还是能简化操作的。

### || 和 &&
&& 和 || 运算符的返回值并不一定是布尔类型，而是两个操作数其中一个的值。

```js
var a = 42;
var b = "abc";
var c = null;

a || b;     // 42
a && b;     // "abc"
c || b;     // 42
c && b;     // null
```



### 符号的强制类型转换
ES6添加了Symbol类型，它的强制类型转换有坑：
* 可以显式强制类型转换为字符串，隐式会报错
```js
let s1 = Symbol('good');
String(s1); //'Symbol(good)'
s1 + '';    //TypeError
```
* 不可以强制类型转换为数字，显式隐式都不可
* 可以强制类型转换为布尔值，显式隐式都转换为true

##  宽松相等(`==`)和严格相等(`===`)

==存在隐式强制类型转换，具体的可以查看整理的博客 [`js中==和===的区别`](http://coolcao.com/2016/08/06/js中-和-的区别/)
