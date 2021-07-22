---
title: 学习笔记-你不知道的js-Object
date: 2016-11-21 20:02:48
tags: [js]
categories:
- 学习笔记
- 你不知道的js
---

## 类型
在Javascript有七种类型：
* undefined
* null
* string
* number
* object
* symbol

<!--more-->

> typeof null 返回的是object,这只是一个语言本身的一个bug,在Javascript中二进制前三位都是0的话会被判断为object类型，null的二进制表示是全0，自然前三位也是0，所以执行typeof时会返回“object”

## 内置对象
* String
* Number
* Boolean
* Object
* Function
* Array
* Date
* RegExp
* Error

在Javascript中，内置对象实际上是一些内置函数。这些内置函数可以当作构造函数来使用，从而可以构造一个对应子类型的新对象。

```js
var strPrimitive = "I am a string";
typeof strPrimitive; // "string"
strPrimitive instanceof String; // false

var strObject = new String( "I am a string" );
typeof strObject; // "object"
strObject instanceof String; // true

// 检查sub-type对象
Object.prototype.toString.call( strObject ); // [object String]

```

原始值'I am a string'并不是一个对象，它只是一个字面量，并且是一个不可变的值。如果要在这个字面量上执行一些操作，比如获取长度，访问其中某个字符等，需要将其转换为String对象。
同样，对于数值字面量，引擎会把数值字面量转换成Number对象，对于布尔字面量来说也是如此。
null和undefined没有对应的构造形式，它们只有文字形式。相反，Date只有构造，没有文字形式。

## 内容
存储在对象容器内的是这些属性的名称，它们就像指针一样，指向这些值真正的存储位置。
如果要访问属性值，可以使用.操作符或者[]操作符。
区别在于，.操作符要求属性名称满足标识符的命名规范，而[]操作符可以接受任意UTF-8/Unicode字符串作为属性名。由于[]操作符使用字符串来访问属性，所以可以在程序中构造这个字符串。
在对象中，属性名永远是字符串。

```js
var obj = {
	a: 'this is a text'
};
// 下面两种都可以访问到对象属性
console.log(obj.a);
console.log(obj['a']);
```

### 可计算属性名
ES6增加了可计算属性名，可以在文字形式中使用[]包裹一个表达式来当作属性名：
```js
var prefix = 'foo';
var myObject = {
    [prefix + 'bar'] : 'hello',
    [prefix + 'baz'] : 'world'
}
myObject['foobar'];//hello
myObject['foobaz'];//world
```
### 属性与方法
函数永远不会属性一个对象。
```js
function foo(){
    console.log('foo');
}
var someFoo = foo();
var myObject = {
    someFoo:foo
}
```

someFoo和myObject.someFoo只是对于同一个函数的不同引用，并不能说明这个函数是特别的或者“属于”某个对象。
即使你在对象的文字形式中声明一个半熟表达式，这个函数也不会属于这个对象，它们只是对于相同函数对象的多个引用。
```js
var myObject = {
    foo:function(){
        console.log('foo');
    }
}
var someFoo = myObject.foo;

```

### 数组
数组也支持[]访问形式，不过就像我们之前提到过的，数组有一套更加结构化的值存储机制。数组期望的是数值下标，也就是说值存储的位置是整数。
数组也是对象，所以虽然每个下标都是整数，你仍然可以给数组添加属性。
```js
var myArray = ['foo',42,'bar']
myArray.baz = 'baz'
myArray.length;//3
myArray.baz;//'baz'
```
虽然添加了命名属性，数组的length值并未发生变化。
*数组和普通的对象都根据其对应的行为和用途进行了优化，所以最好只用对象来存储键/值对，只用数组来存储数值下标/值对*
**注意：如果你试图向数组添加一个属性，但是属性名“看起来”像一个数字，那它会变成一个数值下标**
```js
var myArray = ['foo',42,'bar']
myArray['3'] = 'baz'
myArray.length;//4
myArray[3];//'baz'
```

### 复制对象
区分浅复制还是深复制。
对于JSON安全（可以被序列化成为一个JSON字符串并且可以根据这个字符串解析出一个结构和值完全一样的对象）的对象来说，有一种巧妙的复制方法：
```js
var newObj = JSON.parse(JSON.stringify(someObj));
```
但是这种方式需要保证对象是JSON安全的。

### 属性描述符
从ES5开始，所有的属性都具备了属性描述符。
```js
var myObj = {
    a:2
}
Object.getOwnPropertyDescriptor(myObj,'a');
//{
//    value:2,
//    writable:true,
//    enumerable:true,
//    configurable:true
//}
```
属性描述符包含三个特性：writable(可写),enumerable(可枚举)和configurable(可配置)
#### Writable
Writable决定是否可以修改属性的值
如果writable设置为false，那么在非严格模式下，会静默修改失败（不会报错），在严格模式下，会报错。
#### Configurable
把configurable修改成false是单向的，无法撤销。
> 要注意一个小小的例外：即便属性是configurable:false,我们还是可以把writable的状态由true改为false，但是无法由false改为true.

```js
var obj = {
    a:0
}
Object.defineProperty(obj,'a',{
    value:1,
    writable:true,
    configurable:false,
    enumerable:true
});
console.log(obj.a); //1
Object.defineProperty(obj,'a',{
    value:2,
    writable:true,
    configurable:false,
    enumerable:true
});
console.log(obj.a); //2
Object.defineProperty(obj,'a',{
    value:3,
    writable:false,
    configurable:false,
    enumerable:true
});
console.log(obj.a); //3
Object.defineProperty(obj,'a',{     //TypeError: Cannot redefine property: a
    value:4,
    writable:false,
    configurable:false,
    enumerable:true
});
console.log(obj.a);
```
上面一个例子，obj.a初始为0，将configurable设置为false后，我们还可以继续修改其value值，也可以继续将writable:true改为false，当configurable和writable都为false时，再继续改value，会报错：TypeError: Cannot redefine property: a

除此之外，configurable:false还会禁止删除这个属性。

#### Enumerable
这个描述符控制的是属性是否会出现在对象的属性枚举中，如果把enumerable设置成false,这个属性就不会出现在枚举中，虽然仍然可以正常访问它。

### 不可变性
所有的方法创建的不可变性都是浅不可变性，也就是说，它们只会影响目标对象和它的直接属性。如果目标对象引用了其他对象（数组，对象，函数等），其他对象的内容不受影响，仍然是可变的。

#### 对象常量
结合writable:false和configurable:false就可以创建一个真正的常量属性（不可修改，重新定义，或删除）。
> 这里二者缺一不可，如果只有writable:false，还可以使用defineProperty来改变value的值。

#### 禁止拓展
如果你想禁止一个对象添加新属性并且保留已有属性，可以使用Object.preventExtensions()

#### 密封
Object.seal()会创建一个密封的对象，这个方法实际上会在一个现有对象上调用Object.preventExtensions()并把所有现有属性标记为configurable:false.
密封之后不仅不能添加新属性，也不能重新配置或者删除任何现有属性（虽然可以修改属性的值）

#### 冻结
Object.freeze()会创建一个冻结对象，这个方法实际上会在一个现有对象上调用Object.seal()并把所有属性标记为writable:false，这样就无法修改它们的值。

### [[Get]]
通过obj.a方式访问对象obj的属性a时，实际上是实现了[[Get]]操作，对象默认的内置[[Get]]操作首先在对象中查找是否有名称相同的属性，如果有就返回属性的值，如果没有，继续查找原型链上有没有相同名称的属性，有返回，最终都没找到，则返回undefined。
```js
var obj = {
    a:'a'
}

console.log(obj.b); //undefined
console.log(a);     //ReferenceError:a is not defined

//为什么同样是未定义的，第一个输出undefined,而第二个却报ReferenceError错误？
//obj.b实际上是调用了obj对象的[[Get]]操作，该操作会在对象obj上查找是否有属性b，如果没有，再继续查找obj的原型链上有没有属性b,最终没有找到返回undefined.
//而第二个是直接访问词法作用域中的变量，该变量未定义，就会报ReferenceError错误
```

### [[Put]]
[[Put]]被触发时，实际的行为取决于许多因素，包括对象中是否已经存在这个属性（这是最重要的因素）。
如果已经存在这个属性，[[Put]]算法大致会检查下面这些内容。

1. 属性是否是访问描述符？如果是并且存在setter就调用setter。
2. 属性的数据描述符中writable是否是false？如果是，在非严格模式下静默失败，在严格模式下抛出TypeError异常。
3. 如果都不是，将该值设置为属性的值。

### Getter和Setter
getter和setter是两个隐藏函数，分别在获取属性值和设置属性值时调用。
当你给一个属性定义getter、setter或者两者都有时，这个属性会被定义为“访问描述符”（和“数据描述符”相对）。对于访问描述符来说，JavaScript会忽略它们的value和writable特性，取而代之的是关心set和get（还有configurable和enumerable）特性。
```js
'use strict';
let obj = {
    set a(value) {
        this._a = value;
    },
    get a() {
        return this._a;
    }
}
Object.defineProperty(obj, 'a', {
    enumerable: true,
    configurable: true
})
obj.a = 'abc';
console.log(obj.a); //abc
console.log(obj);   //{ a: [Getter/Setter], _a: 'abc' }
```
我们可以看到，如果定义了getter和setter，还可以继续定义其enumerable和configurable属性。那如果再继续设置value和writable属性会怎样？
```js
Object.defineProperty(obj, 'a', {
    value:'aa',
    enumerable: true,
    configurable: true,
    writable:false
})
//obj.a = 'abc';      //TypeError: Cannot assign to read only property 'a' of object '#<Object>'
console.log(obj.a); //aa
console.log(obj);   //{ a: 'aa' }
```
从上面这个例子中，我们可以看出，如果定义了getter,setter后，再定义其writable和value属性，则会覆盖掉getter和setter.

### 存在性
```js
'use strict';
var obj = {
    a:'a'
}
console.log('a' in obj);    //true
console.log('toString' in obj); //true
console.log(obj.hasOwnProperty('a'));   //true
console.log(obj.hasOwnProperty('b'));   //false

```
* **in** 操作符会检查属性是否在对象及其[[Prototype]]原型链中。
* **hasOwnProperty(..)** 只会检查属性是否在对象中，不会检查[[Prototype]]链。

#### 枚举
```js
var myObject = { };
Object.defineProperty(
    myObject,
    "a",
    // 让a像普通属性一样可以枚举
    { enumerable: true, value: 2 }
);
Object.defineProperty(
    myObject,
    "b",
    // 让b不可枚举
    { enumerable: false, value: 3 }
);
myObject.b; // 3
("b" in myObject); // true
myObject.hasOwnProperty( "b" ); // true
// .......
for (var k in myObject) {
    console.log( k, myObject[k] );
}
// "a" 2
Object.keys(myObject);  //['a']
Object.getOwnPropertyNames(myObject);   //['a','b']
```
可枚举就相当于可以出现在对象属性的遍历中。
* **Object.keys()** 会返回一个数组，包含所有可枚举属性（不包含其原型链上的可枚举属性）
* **Object.getOwnPropertyDescriptor()** 会返回一个数组，包含所有属性，无论它们是否可枚举。（不包含其原型链上的属性）
* **in** 和 **hasOwnProperty()** 的区别在于是否查找原型链。

## 遍历
ES6增加了一个 `for of`循环用于遍历数组（如果对象本身定义了迭代器的话，也可以使用for of遍历对象）。
```js
var array = [1,2,3];
var it = array[Symbol.iterator]();
it.next();  //{value:1,done:false}
it.next();  //{value:2,done:false}
it.next();  //{value:3,done:false}
it.next();  //{done:true}
```
数组默认实现了迭代器，因此，可以可以直接获取其迭代器进行迭代输出。
如果我们给普通对象实现迭代器，也可以使用for of遍历这个对象。
```js
'use strict';

var obj = {
    a:'a',
    b:'b'
}

Object.defineProperty(obj,Symbol.iterator,{
    enumerable:false,
    configurable:true,
    value:function () {
        let o = this;
        let idx = 0;
        let ks = Object.keys(o);
        return {
            next:function(){
                return {
                    value:o[ks[idx ++]],
                    done:(idx > ks.length)
                }
            }
        }

    }
});

for(let v of obj){
    console.log(v);
}
```
for of循环每次调用迭代器对象的next()方法时，内部的指针都会向前移动并返回对象属性列表的下一个值。
