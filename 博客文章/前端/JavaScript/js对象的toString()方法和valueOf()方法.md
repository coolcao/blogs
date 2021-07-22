---
title: js对象的toString()方法和valueOf()方法
date: 2016-08-19 22:43:00
tags: [js]
categories:
- 技术博客
- 原创
---
在研究js的==和===的区别时，曾经说过，在js中非原始值对象，要参加运算需要ToPrimitive(x)转换成原始值类型。
在ToPrimitive(x)时，我们曾得出结论：

## ToPrimitive

>* ToPrimitive(input,hint)转换为原始类型的方法，根据hint目标类型进行转换。
>* hint只有两个值：String和Number
>* 如果没有传hint，Date类型的input的hint默认为String,其他类型的input的hint默认为Number
>* Number 类型先判断 valueOf()方法的返回值，如果不是，再判断 toString()方法的返回值
>* String 类型先判断 toString()方法的返回值，如果不是，再判断 valueOf()方法的返回值

也就是说，js中的对象离不开toString()方法和valueOf()方法。那到底何时用toString()方法，何时用valueOf()方法呢？怎么用？

```javascript
//先定义一个用户User类
class User {
    constructor(name,age) {
        this.name = name;
        this.age = age;
    }
}
```
我们先定义一个用户User类，如果我们不重写toString()方法和valueOf()方法 ，直接输入下面的值：
```javascript
var coolcao = new User('coolcao',23);
console.log(coolcao);
console.log(coolcao.toString());
console.log(coolcao.valueOf());
console.log(coolcao + 10);
//输出
//User { name: 'coolcao', age: 23 }
//[object Object]
//User { name: 'coolcao', age: 23 }
//[object Object]10
```
直接输入对象coolcao,我们得到的就是对象的表示形式。注意，此时User { name: 'coolcao', age: 23 }是在nodejs环境下的输出，在浏览器中输出的形式可能不同，但结论都是一样的，输出的就是一个User的实例化对象的表示形式。
coolcao.toString()输出的是[object Object],而coolcao.valueOf()输出的是对象本身，还是个对象。
而当使用coolcao+10进行运算的时候，将对象coolcao调用的toString()方法得到的字符串进行计算的。
那是不是意味着，我们在使用 + 操作符等对对象进行计算时，会调用toString()方法呢？
我们重写一下User的valueOf()方法。
```javascript
class User {
    constructor(name,age) {
        this.name = name;
        this.age = age;
    }
    valueOf(){
        return this.age;
    }
}
```
再来看一下上面四个console的输出：
```
{ [Number: 23] name: 'coolcao', age: 23 }
[object Object]
23
33
```
这时，我们发现，直接输出对象的形式也变了，最后使用coolcao+10的时候，结果也变了。
对比两次结果，我们可以发现，直接输出对象，只是显示变了，但是输出的还是对象，本质是没变的。
唯一变得是，当直接使用对象进行计算的时候，值变了，coolcao对象取的是valueOf()返回的值，进行计算的。
我们再来重写一下toString()方法：
```javascript
class User {
    constructor(name,age) {
        this.name = name;
        this.age = age;
    }
    valueOf(){
        return this.age;
    }
    toString(){
        return `[name:${this.name}|age:${this.age}]`;
    }
}
```
上面打印的结果：
```
{ [Number: 23] name: 'coolcao', age: 23 }
[name:coolcao|age:23]
23
33
```
我们看出，最后计算还是没有变，还是取的valueOf()的值。

那是不是我们可以得出结论，对于自定义的对象，在使用对象进行计算时，将使用其valueOf()的返回值呢？

不严谨的讲，可以这么说。

其实，是这样的，对于非原始类型，进行计算时要转换成原始类型。转换步骤就是本篇开头说的步骤方法。
User是自定义的类，因此使用ToPrimitive()方法转换时，hint默认是Number。将默认调用DefaultNumber()进行转换。
即先判断valueOf()方法的返回值。默认情况下(我们不重写其valueOf方法)valueOf()方法返回对象本身，还是一个object类型，因此再调用DefaultString()进行转换，判断的是toString()，因此第一个我们没重写之前调用的是toString()返回的结果进行计算的。后面我们重写了valueOf()方法，返回原始类型数据，就取valueOf的返回结果进行计算。

注意：并不是说，我们重写了valueOf方法，就一定调用valueOf()的返回值进行计算。而是valueOf返回的值是原始值时才会按照此值进行计算。
例如，如果我们这么写：
```javascript
var coolcao = new User('coolcao',{key:'age',value:22});
```
由于js语言的灵活性，我们在实例化User对象时，可以将age参数传一个对象。如果这样，valueOf()返回this.age将是一个object类型，还不是原始值。那么，最后的输出结果将是：
```
User { name: 'coolcao', age: { key: 'age', value: 22 } }
[name:coolcao|age:[object Object]]
{ key: 'age', value: 22 }
[name:coolcao|age:[object Object]]10
```

再看下面一个例子，User的定义还是如上，重写了toString()和valueOf()方法后：
```javascript
var user = new User('coolcao',23);
console.log(user + 1);
console.log([user,1].join());
```

先猜测一下这两个输出什么呢？

```
24
[name:coolcao|age:23]1
```

会不会有疑问，为什么第一个直接使用 + 操作符计算的时候，取的valueOf()的值，而第二个对数组进行join()的时候，取得是toString()的值呢？
我们说过,对对象进行运算操作时,默认先判断valueOf,如果valueOf()方法返回的是原始类型的值,就使用原始类型的值进行计算,否则将使用toString()方法返回的值进行计算.
那么第二个输出是什么鬼,按道理讲不应该是 字符串 '11' 嘛?难道上面的结论总结错了?
其实原因在于数组的join方法.
我们看一下join方法的实现说明:
>The join method takes one argument, separator, and performs the following steps:

>1. Let O be ? ToObject(this value).
2. Let len be ? ToLength(? Get(O, "length")).
3. If separator is undefined, let separator be the single‐element String ",".
4. Let sep be ? ToString(separator).
5. If len is zero, return the empty String.
6. Let element0 be Get(O, "0").
7. If element0 is undefined or null, let R be the empty String; otherwise, let R be ? ToString(element0).
8. Let k be 1.
9. Repeat, while k < len
   a. Let S be the String value produced by concatenating R and sep.
   b. Let element be ? Get(O, ! ToString(k)).
   c. If element is undefined or null, let next be the empty String; otherwise, let next be ? ToString(element).
   d. Let R be a String value produced by concatenating S and next.
   e. Increase k by 1.
10. Return R.
**NOTE 2** The join function is intentionally generic; it does not require that its this value be an Array object. Therefore,it can be transferred to other kinds of objects for use as a method.

从上面的步骤中我们看出,数组的join方法进行计算的时候,要对每个非undefined和非null的项转换成字符串.所以上面例子中输出的结果也就不难理解了.


## Object.prototype.toString
js的所有对象，都是继承于Object的，因此，每个对象都会默认有个toString()方法。我们先看一下ECMA-262对toString方法的解释：
```
When the toString method is called, the following steps are taken:
1. If the this value is undefined, return "[object Undefined]".
2. If the this value is null, return "[object Null]".
3. Let O be ToObject(this value).
4. Let isArray be ? IsArray(O).
5. If isArray is true, let builtinTag be "Array".
6. Else, if O is an exotic String object, let builtinTag be "String".
7. Else, if O has an [[ParameterMap]] internal slot, let builtinTag be "Arguments". 8. Else, if O has a [[Call]] internal method, let builtinTag be "Function".
9. Else, if O has an [[ErrorData]] internal slot, let builtinTag be "Error".
10. Else, if O has a [[BooleanData]] internal slot, let builtinTag be "Boolean".
11. Else, if O has a [[NumberData]] internal slot, let builtinTag be "Number".
12. Else, if O has a [[DateValue]] internal slot, let builtinTag be "Date".
13. Else, if O has a [[RegExpMatcher]] internal slot, let builtinTag be "RegExp".
14. Else, let builtinTag be "Object".
15. Let tag be ? Get(O, @@toStringTag).
16. If Type(tag) is not String, let tag be builtinTag.
17. Return the String that is the result of concatenating "[object ", tag, and "]".
This function is the %ObjProto_toString% intrinsic object.
NOTE Historically, this function was occasionally used to access the String value of the [[Class]] internal slot that was used in previous editions of this speci ication as a nominal type tag for various built‐in objects. The above de inition of toString preserves compatibility for legacy code that uses toString as a test for those speci ic kinds of built‐in objects. It does not provide a reliable type testing mechanism for other kinds of built‐in or program de ined objects. In addition, programs can use @@toStringTag in ways that will invalidate the reliability of such legacy type tests.
```
从说明上，我们可以看出，toString()默认会返回字符串 `"[object ", tag"]"`。
而tag的值将会有下面几个值：Undefined,Null,Array,String,Arguments,Boolean,Number,Date,RegExp,Object,Error
这里有一个小技巧，如果要判断一个值的具体类型，就可以使用toString()方法进行判断。例如：
```javascript
Object.prototype.toString.call([]);     //'[object Array]'
Object.prototype.toString.call(null);   //'[object Null]'
Object.prototype.toString.call(1);      //'[object Number]'
Object.prototype.toString.call(NaN);    //'[object Number]'
Object.prototype.toString.call(true);   //'[object Boolean]'
Object.prototype.toString.call(function(){});   //'[object Function]'
```
只要判断toString()返回的字符串即可。

## Array.prototype.toString

>When the toString method is called, the following steps are taken:

>1. Let array be ? ToObject(this value).
2. Let func be ? Get(array, "join").
3. If IsCallable(func) is false, let func be the intrinsic function %ObjProto_toString%.
4. Return ? Call(func, array).

>**NOTE** The toString function is intentionally generic; it does not require that its this value be an Array object. Therefore it can be transferred to other kinds of objects for use as a method.

从说明中我们看以看出，数组的toString()实现步骤是，先将数组ToObject()转换成包装类对象，这里由于数组本身就是包装类，因此这里返回的是数组本身。
array = ToObject(this)这里array即是数组本身。
然后另func等于Get(array,'join').那这里就要看一下Get()这个方法的实现了。

>**Get (O, P)**

>1. Assert: Type(O) is Object.
2. Assert: IsPropertyKey(P) is true.
3. Return ? O.\[\[Get\]\](P, O).

首先断言O是不是一个对象，这里就是数组嘛，当然是，通过断言。
然后断言O有没有P属性，对于数组Array来说，是有join方法的，断言通过。继续往下走。
返回的就是调用O上的P属性返回的值。即这里Get(array,'join')返回的就是 array.join的值，这里是一个方法。即 `func = array.join`
往上看，func 是数组的join方法，当然是可执行的，因此第三步中的 IsCallable(func)是true，继续第四步
最后返回Call(func,array)，即执行array.join()方法
**也就是说，对于数组，如果不重写其toString()方法，其默认实现就是调用数组的 join()方法返回值作为toString()的返回值。**

多说无意，看例子：
```javascript
> [].toString()
''
> [1,2,3].toString()
'1,2,3'
> [1,2,3].join()
'1,2,3'
> [true,false].join()
'true,false'
> [0,1,'2'].join()
'0,1,2'
> [0,1,'2'].toString()
'0,1,2'
```

所以，只要看明白ecma规范中对于各个方法的说明，是很容易理解的。
所以，像Boolean,Function等等的toString()方法的实现说明，看ecma的说明就好了，多说无益。这里就不再往上抄了，也不做记录了。
