---
    title: js中==和===的区别
    date: 2016-08-06 22:00:00
    auther: coolcao,
    tags: [js]
---

做js的时候，也没仔细的研究过==和===两个操作符之间的具体区别，但是本着实事求是的态度，今天上网查了一下他们两个的区别，最后得出的结论是：

```
==绝对相等，会比较两个值的类型和值
===比较时，会先进行类型转换，然后再比较值
```

然后我就更迷惑了，先类型转换，我擦，怎么转换，左边转换成右边类型还是右边转换成左边类型？

先看几个例子：

```javascript
console.log([10] == 10);                //true
console.log('10' == 10);                //true
console.log([] == 0);                   //true
console.log(true == 1);                 //true
console.log([] == false);               //true
console.log(![] == false);              //true
console.log('' == 0);                   //true
console.log('' == false);               //true
console.log(null == false);             //false
console.log(!null == true);             //true
console.log(null == undefined);         //true
```

看完这几个例子，我更迷糊了，啥，不是说好先进行类型转换嘛，怎么转换？除了 `null==false`是false外，其他的都是true?但是 `!null==true`却是true?这是为啥？

没辙，越查越迷糊，乖乖去看ECMA的规范吧，看看标准是怎么规定的。

### Abstract Equality Comparison
>The comparison x == y, where x and y are values, produces true or false. Such a comparison is performed as follows:

>1. If Type(x) is the same as Type(y), then
    a. Return the result of performing Strict Equality Comparison x === y.

>2. If x is null and y is undefined, return true.

>3. If x is undefined and y is null, return true.

>4. If Type(x) is Number and Type(y) is String, return the result of the comparison x == ToNumber(y).

>5. If Type(x) is String and Type(y) is Number, return the result of the comparison ToNumber(x) == y.

>6. If Type(x) is Boolean, return the result of the comparison ToNumber(x) == y.

>7. If Type(y) is Boolean, return the result of the comparison x == ToNumber(y).

>8. If Type(x) is either String, Number, or Symbol and Type(y) is Object, return the result of the comparison x == ToPrimitive(y).

>9. If Type(x) is Object and Type(y) is either String, Number, or Symbol, return the result of the comparison ToPrimitive(x)== y.

>10. Return false.


### ToPrimitive ( input [ , PreferredType ] )
>The abstract operation ToPrimitive takes an input argument and an optional argument PreferredType. The abstract operation ToPrimitive converts its input argument to a non‐Object type. If an object is capable of converting to more than one primitive type, it may use the optional hint PreferredType to favour that type.
Conversion occurs according to Table 9:

>Table 9: ToPrimitive Conversions

>|Input Type|Result|
|----|----|
|Undefined|Return input.|
|Null|Return input.|
|Boolean|Return input.|
|Number|Return input.|
|String|Return input.|
|Symbol|Return input.|
|Object|Perform the steps following this table.|

>When Type(input) is Object, the following steps are taken:

>1. If PreferredType was not passed, let hint be "default".

>2. Else if PreferredType is hint String, let hint be "string".

>3. Else PreferredType is hint Number, let hint be "number".

>4. Let exoticToPrim be ? GetMethod(input, @@toPrimitive).

>5. If exoticToPrim is not undefined, then
    * a. Let result be ? Call(exoticToPrim, input, « hint »).
    * b. If Type(result) is not Object, return result.
    * c. Throw a TypeError exception.

>6. If hint is "default", let hint be "number".

>7. Return ? OrdinaryToPrimitive(input, hint).

>When the abstract operation OrdinaryToPrimitive is called with arguments O and hint, the following steps are taken:

>1. Assert: Type(O) is Object.

>2. Assert: Type(hint) is String and its value is either "string" or "number".

>3. If hint is "string", then
    * a. Let methodNames be « "toString", "valueOf" ».

>4. Else,
    * a. Let methodNames be « "valueOf", "toString" ».

>5. For each name in methodNames in List order, do
    * a. Let method be ? Get(O, name).
    * b. If IsCallable(method) is true, then
        * i. Let result be ? Call(method, O).
        * ii. If Type(result) is not Object, return result.

>6. Throw a TypeError exception.

>**NOTE** When ToPrimitive is called with no hint, then it generally behaves as if the hint were Number. However, objects may over‐ride this behaviour by de ining a @@toPrimitive method. Of the objects de ined in this speci ication only Date objects (see 20.3.4.45) and Symbol objects (see 19.4.3.4) over‐ride the default ToPrimitive behaviour. Date objects treat no hint as if the hint were String.

### Strict Equality Comparison
>The comparison x === y, where x and y are values, produces true or false. Such a comparison is performed as follows:

>1. If Type(x) is different from Type(y), return false.

>2. If Type(x) is Number, then
    * a. If x is NaN, return false.
    * b. If y is NaN, return false.
    * c. If x is the same Number value as y, return true.
    * d. Ifxis+0andyis‐0,returntrue.
    * e. If x is ‐0 and y is +0, return true.
    * f. Return false.

>3. Return SameValueNonNumber(x, y).

>**NOTE** This algorithm differs from the SameValue Algorithm in its treatment of signed zeroes and NaNs.

### SameValueNonNumber (x, y)
>The internal comparison abstract operation SameValueNonNumber(x, y), where neither x nor y are Number values, produces
true or false. Such a comparison is performed as follows:

>1. Assert: Type(x) is not Number.

>2. Assert: Type(x) is the same as Type(y).

>3. If Type(x) is Unde ined, return true.

>4. If Type(x) is Null, return true.

>5. If Type(x) is String, then
    * a. If x and y are exactly the same sequence of code units (same length and same code units at corresponding indices), return true; otherwise, return false.

>6. If Type(x) is Boolean, then
    * a. If x and y are both true or both false, return true; otherwise, return false.

>7. If Type(x) is Symbol, then
    * a. If x and y are both the same Symbol value, return true; otherwise, return false.

>8. Return true if x and y are the same Object value. Otherwise, return false.


上面就是ecma规范对==和===两个操作符的规范说明，看看这两个操作符是怎么操作的。

### ===
* 如果Type(x)和Type(y)不同，返回false
* 如果Type(x)和Type(y)相同
    * 如果Type(x)是Undefined，返回true
    * 如果Type(x)是Null，返回true
    * 如果Type(x)是String，当且仅当x,y字符序列完全相同（长度相同，每个位置上的字符也相同）时返回true，否则返回false
    * 如果Type(x)是Boolean，如果x,y都是true或x,y都是false返回true，否则返回false
    * 如果Type(x)是Symbol，如果x,y是相同的Symbol值，返回true,否则返回false
    * 如果Type(x)是Number类型
        * 如果x是NaN，返回false
        * 如果y是NaN，返回false
        * 如果x的数字值和y相等，返回true
        * 如果x是+0，y是-0，返回true
        * 如果x是-0，y是+0，返回true
        * 其他返回false

### ==
* 如果Type(x)和Type(y)相同，返回x===y的结果
* 如果Type(x)和Type(y)不同
    * 如果x是null，y是undefined，返回true
    * 如果x是undefined，y是null，返回true
    * 如果Type(x)是Number，Type(y)是String，返回 x==ToNumber(y) 的结果
    * 如果Type(x)是String，Type(y)是Number，返回 ToNumber(x)==y 的结果
    * 如果Type(x)是Boolean，返回 ToNumber(x)==y 的结果
    * 如果Type(y)是Boolean，返回 x==ToNumber(y) 的结果
    * 如果Type(x)是String或Number或Symbol中的一种并且Type(y)是Object，返回 x==ToPrimitive(y) 的结果
    * 如果Type(x)是Object并且Type(y)是String或Number或Symbol中的一种，返回 ToPrimitive(x)==y 的结果
    * 其他返回false


## ToPrimitive() 方法
转换成原始类型方法。
js中的原始类型：
* Null: null值.
* Undefined: undefined 值.
* Number: 所有的数字类型，例如0,1,3.14等 以及NaN, 和 Infinity.
* Boolean: 两个值true和false.
* String: 所有的字符串，例如'abc'和''.
其他的都是'非原始'的，像Array,Function,Object等。

*注意：typeof null 得到的结果是object。这里是由于js在最初的设计的问题。但其实null应该是属于原始类型的。*

ToPrimitive ( input [ , PreferredType ] )方法传递两个参数input和PreferredType，其中PreferredType是可选的。input输入,PreferredType可选的期望类型。
ToPrimitive 运算符把其值参数转换为非对象类型。如果对象有能力被转换为不止一种原语类 型,可以使用可选的 期望类型 来暗示那个类型。根据下表完成转换:

|输入类型|结果|
|----|----|
|Undefined|不转换|
|Null|不转换|
|Boolean|不转换|
|Number|不转换|
|String|不转换|
|Object|返回该对象的默认值。（调用该对象的内部方法[[DefaultValue]]一样）

具体的实现步骤，在ECMA-262的说明中都有详细的说明，请参阅。

对于一个Object，ToPrimitive()方法是如何转换成原始类型的呢，最后转换成的原始类型的值又是多少呢？
我们光从上面的表格中可能看不出太多的东西来。下面我们来看看js的源码实现。

```javascript
// ECMA-262, section 9.1, page 30. Use null/undefined for no hint,
// (1) for number hint, and (2) for string hint.
function ToPrimitive(x, hint) {
  // Fast case check.
  // 如果为字符串，则直接返回
  if (IS_STRING(x)) return x;
  // Normal behavior.
  if (!IS_SPEC_OBJECT(x)) return x;
  if (IS_SYMBOL_WRAPPER(x)) throw MakeTypeError('symbol_to_primitive', []);
  if (hint == NO_HINT) hint = (IS_DATE(x)) ? STRING_HINT : NUMBER_HINT;
  return (hint == NUMBER_HINT) ? %DefaultNumber(x) : %DefaultString(x);
}
```
默认情况下，如果不传hint参数，x如果是日期类型，则hint为String,否则hint为Number类型。
然后根据hint将处理结果返回。
对于[10]数组这个例子，默认hint应该为Number类型。
处理函数为 DefaultNumber(x) 。
那么，DefaultNumber()又是如何处理的呢？
继续找代码看呗

DefaultNumber()方法：
```javascript
// ECMA-262, section 8.6.2.6, page 28.
function DefaultNumber(x) {
  if (!IS_SYMBOL_WRAPPER(x)) {
    var valueOf = x.valueOf;
    if (IS_SPEC_FUNCTION(valueOf)) {
      var v = %_CallFunction(x, valueOf);
      if (%IsPrimitive(v)) return v;
    }

    var toString = x.toString;
    if (IS_SPEC_FUNCTION(toString)) {
      var s = %_CallFunction(x, toString);
      if (%IsPrimitive(s)) return s;
    }
  }
  throw %MakeTypeError('cannot_convert_to_primitive', []);
}
```

* 首先通过valueOf 转换，即 obj.valueOf()方法的返回值
* 如果 obj.valueOf()方法的返回值是原始类型，那么直接返回
* 如果不是，再通过 obj.toString()方法转换
* 如果obj.toString()返回的是原始类型，直接返回该值
* 如果还不是原始类型，抛出不能转换异常。

DefaultString()方法：
```javascript
// ECMA-262, section 8.6.2.6, page 28.
function DefaultString(x) {
  if (!IS_SYMBOL_WRAPPER(x)) {
    // 转换为字符串原始类型时首先通过toString
    var toString = x.toString;
    if (IS_SPEC_FUNCTION(toString)) {
      var s = %_CallFunction(x, toString);
      if (%IsPrimitive(s)) return s;
    }

    // 否则通过valueOf
    var valueOf = x.valueOf;
    if (IS_SPEC_FUNCTION(valueOf)) {
      var v = %_CallFunction(x, valueOf);
      if (%IsPrimitive(v)) return v;
    }
  }
  // 否则抛出异常
  throw %MakeTypeError('cannot_convert_to_primitive', []);
}
```

* 首先使用toString()转换
* 如果obj.toString()返回的是原始类型的值，直接返回该值
* 如果不是，再使用obj.valueOf()转换
* 如果obj.valueOf()返回的是原始类型的值，返回该值
* 如果还不是，抛出不能转换异常

好乱的有木有，简单总结下：
* ToPrimitive(input,hint)转换为原始类型的方法，根据hint目标类型进行转换。
* hint只有两个值：String和Number
* 如果没有传hint，Date类型的input的hint默认为String,其他类型的input的hint默认为Number
* Number 类型先判断 valueOf()方法的返回值，如果不是，再判断 toString()方法的返回值
* String 类型先判断 toString()方法的返回值，如果不是，再判断 valueOf()方法的返回值

来两个简单的例子看一下ToPrimitive()的结果可能会更有效。

1.求 ToPrimitive([10])
对于数组，默认的hint应该为Number ,所以先判断 valueOf()
```
> [10].valueOf()
[10]
```
数组的valueOf()方法返回的是数组本身，不是一个原始类型，我们继续判断 toString()
```
> [10].toString();
'10'
```
toString()返回的是一个字符串 '10' 是一个原始类型，因此，`ToPrimitive([10]) = '10'`

2.ToPrimitive(function sayHi(){})
```
var sayHi = function (name) {
    console.log('hi ' + name);
}
console.log(sayHi.valueOf());
console.log(typeof sayHi.valueOf());
console.log(sayHi.toString());
console.log(typeof sayHi.toString());
```

结果：
```
[Function]
function
function (name) {
    console.log('hi ' + name);
}
string
```
先判断 valueOf() ，返回的是一个function类型，只能继续判断 toString()返回的是string类型的，是原始类型，因此，
ToPrimitive(sayHi)是源码字符串。

有点远了，我们回过头来看 == 的那几个例子。
首先 [10]==10 这个
类型不同，Type([10])是Object，而Type(10)是Number,我们应该判断`ToPrimitive([10])==10`的结果
由上面的分析，ToPrimitive([10])的结果为 字符串'10'，因此结果变为要判断 '10'==10
Type('10')为String,(10)为Number，因此结果变为 ToNumber('10')==10
ToNumber('10')的值是 数字 10 ，因此，最后要判断的是 10==10
很明显，类型相同，值也相同，最终返回的就是 true

再来看 []==0 这个
同样地道理，ToPrimitive([])==''
而 ToNumber('')==0
因此 []==0 的结果是 true

[]==false也很简单了
false是Boolean类型的，我们要比较 []==ToNumber(false)即 []==0，就是上面这个啊，结果是 true

既然[]==false返回的是true，那么，![]==false应该返回的false吧，我靠，怎么还是true?
首先，左边 ![] 进行取反，要转换成Boolean类型，然后取反。Boolean([])的结果是true,取反是false
也就是，最终比较的是 false==false   结果当然是 true 了。

从上面我们可以看出，[]==![]是成立的，神奇吧，对，js就是这么神奇。

null==false的结果是false，==运算规则的最后一条，前面所有的都不满足，最后返回false，没什么好纠结的。
null==false的结果是false，但这并不代表 null==true 。因为 null==true 最后走的还是最后一条，返回false。

好了，写到这里应该差不多了，现在已经是凌晨3点了，脑子都转不动了，额，思路也比较乱，文章写的可能没那么有条理，如有发现错误之处，敬请指教吧。

其实还有很多隐藏的内容没说明白，比如 js 内部 ToNumber(),ToBoolean()是如何转换的等，ECMA-262规范说明中已经有很详细的说明，可以具体参考一下，这是最最最官方的说明了，没有比这个更准确的了。

主要参考内容：

* [ECMA-262规范说明](http://www.ecma-international.org/ecma-262/7.0/)
* [javascript类型与类型转换](http://sentsin.com/web/1108.html)
* [Object-to-Primitive Conversions in JavaScript](http://www.adequatelygood.com/Object-to-Primitive-Conversions-in-JavaScript.html)
* [MDN](https://developer.mozilla.org/en-US/docs/Mozilla/Projects/SpiderMonkey/JSAPI_Reference/JS::ToPrimitive)
* [知乎上的讨论贴：JavaScript 中应该用 "==" 还是 "==="](https://www.zhihu.com/question/20348948)
