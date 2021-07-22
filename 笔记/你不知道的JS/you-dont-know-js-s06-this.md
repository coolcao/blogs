---
title: 学习笔记-你不知道的js-this
date: 2016-11-18 23:07:46
tags: [js]
categories:
- 学习笔记
- 你不知道的js
---

this关键字是Javascript中最复杂的机制之一。它是一个很特别的关键字，被自动定义在所有函数的作用域中。但是即使是非常有经验的Javascript开发者，也很难说清除它到底指向什么。

<!--more-->

## 对this的误解
* 指向自身:人们会很容易把this理解成指向函数自身
* 它的作用域:第二种常见的误解是，this指向函数的作用域。其实，this在任何情况下都不指向函数的词法作用域。在js内部，作用域确实和对象类似，可见的标识符都是它的属性，但是作用域“对象”无法通过Javascript代码访问，它存在于Javascript引擎内部。

```javascript
function foo() {
    var a = 2;
    this.bar();
}
function bar(){
    console.log(this.a);
}
foo();  //ReferenceError: a is not defined
```
首先，这段代码试图通过调用this.bar()来引用bar()函数。这样调用成功*纯属意外*。此外，编写这段代码的开发者还试图使用this关联foo()和bar()的词法作用域，从而让bar()可以访问foo()作用域里的变量a。这是不可能实现的，使用this不可能在词法作用域中查到什么。

## this到底是什么
我们说过，this是在运行时绑定的，而不是在编写时绑定的，它的上下文取决于函数调用时的各种条件。this的绑定和函数声明的位置没有任何关系，值取决于函数的调用方式。
当一个函数被调用时，会创建一个活动记录（上下文）。这个记录会包含函数在哪里被调用（调用栈），函数的调用方式，传入的参数等信息。this就是这个记录的一个属性，会在函数执行的过程中用到。

## 调用位置
在理解this的绑定过程之前，首先要理解调用位置，调用位置就是函数在代码中被调用的位置。
你可以把函数调用栈想像成一个函数调用链，可是使用浏览器的调试工具查看。

## 绑定规则
### 默认绑定
```js
function foo() {
    console.log( this.a );
}

var a = 2;

foo(); // 2”
```
当调用foo()时,this.a被解析成了全局变量a,因为在本例中，函数调用时应用了this的默认绑定，this指向全局对象。
在代码中,foo()是直接使用不带任何修饰的函数引用进行调用的，因此只能使用默认绑定，无法应用到其他规则。
如果使用严格模式，则不能将全局对象用于默认绑定，因此this会绑定到undefined:
```js
function foo() {
    "use strict";

    console.log( this.a );
}

var a = 2;

foo(); // TypeError: this is undefined”
```
虽然this的绑定规则完全取决于调用位置，但是只有foo()运行在非strict mode下时，默认绑定才能绑定到全局对象。在严格模式下调用foo()则不影响默认绑定。
```js
function foo() {
    console.log( this.a );
}

var a = 2;

(function(){
    "use strict";

    foo(); // 2
})();

```
> 通常来说，不应该在代码中混合使用严格模式和非严格模式。

### 隐式绑定
另一条需要考虑的规则是调用位置是否有上下文对象，或者说是否被某个对象拥有或者包含。
```js
function foo() {
    console.log( this.a );
}

var obj = {
    a: 2,
    foo: foo
};

obj.foo(); // 2”
```
调用位置会使用obj上下文来引用函数，因此可以说函数被调用时obj对象“拥有”或者“包含”它。

无论你如何称呼这个模式，当 foo() 被调用时，它的落脚点确实指向 obj 对象。**当函数引 用有上下文对象时，隐式绑定规则会把函数调用中的 this 绑定到这个上下文对象。** 因为调 用 foo() 时 this 被绑定到 obj，因此 this.a 和 obj.a 是一样的。

当函数引用上下文对象时，隐式绑定规则会把函数调用中的this绑定到这个上下文对象。
对象属性引用链中只有上一层或者说最后一层在调用位置中起作用。
```js
function foo(){
    console.log(this.a);
}
var obj2 = {
    a:42,
    foo:foo
}
var obj1 = {
    a:2,
    obj2:obj2
}
obj1.obj2.foo();    //42
```

#### 隐式丢失
一个最常见的this绑定问题就是被隐式绑定的函数会丢失绑定对象，也就是说它会应用默认绑定，从而把this绑定到全局对象或者undefined上，取决于是否是严格模式.
```js
function foo() {
    console.log( this.a );
}

var obj = {
    a: 2,
    foo: foo
};

var bar = obj.foo; // 函数别名！
￼
var a = "oops, global"; // a是全局对象的属性”
bar(); // "oops, global"

```
虽然bar是obj.foo的一个引用，但实际上，它引用的是foo函数本身，因此此时bar()其实是一个不带任何修饰的函数调用，应用默认绑定。

### 显式绑定
使用call和apply显式绑定
```js
function foo() {
    console.log(this.a);
}
var obj = {
    a:2
}
foo.call(obj);  //2
```
通过foo.call()我们可以在调用foo时强制把它的this绑定到obj上。
如果你传入了一个原始值来当作this的绑定对象，这个原始值会被转换成它的对象形式。这通常被称为“装箱”。
> 从this绑定的角度来说，call()和apply()是一样的，他们的区别提现在其他的参数上。

#### 硬绑定
```js
function foo() {
    console.log(this.a);
}
var obj = {
    a:2
}
var bar = function () {
    foo.call(obj);
}
bar();  //2
setTimeout(bar,100);//2
//硬绑定的bar不可能再修改它的this
bar.call(window);   //2
```
我们创建了函数bar()，并在它的内部手动调用foo.call(obj),强制把foo的this绑定到obj，无论之后如何调用bar()，它总会在obj上调用foo。这种绑定是一种显式的强制绑定，称之为硬绑定。

硬绑定是一种常见的模式，所以ES5中提供了内置的方法Function.prototype.bind，它的用法如下：
```js
function foo(something) {
    console.log( this.a, something );
    return this.a + something;
}

var obj = {
    a:2
};

var bar = foo.bind( obj );

var b = bar( 3 ); // 2 3
console.log( b ); // 5”

```
bind()会返回一个硬编码的新函数，它会把参数设置为this的上下文并调用原始函数。

#### API提供的上下文
第三方库的许多函数，以及javascript语言和宿主环境中许多新的内置函数，都提供了一个可选的参数，通常称为上下文，其作用和bind一样，确保你的回调函数使用指定的this.
```js
function foo(el) {
    console.log( el, this.id );
}

var obj = {
    id: "awesome"
};

// 调用foo(..)时把this绑定到obj
[1, 2, 3].forEach( foo, obj );
// 1 awesome 2 awesome 3 awesome”
```

### new绑定
在传统的面向类的语言中，构造函数是类中的一些特殊方法，使用new初始化类时会调用类中的构造函数，通常的形式是这样的：
```java
something = new MyClass();
```
javascript也有一个new操作符，使用方法看起来和那些面向类的语言一样，绝大多数开发者都认为javascript中new的机制也和那些语言一样。然而javascript中new的机制实际上和面向类的语言完全不同。
使用new来调用函数时，会自动执行下面的操作：
* 创建一个全新的对象
* 这个新对象会被执行[[prototype]]连接
* 这个新对象会绑定到函数调用的this
* 如果函数没有返回其他对象，那么new表达式中的函数会自动返回这个新对象。
```js
function foo() {
    this.a = a;
}
var bar = new foo(2);
console.log(bar.a); //2
```
使用new来调用foo()时，我们会构造一个新对象并把它绑定到foo()调用中的this上。

## 优先级
* 显式绑定比隐式绑定优先级高
* new绑定比隐式绑定优先级更高

### 判断this
我们可以根据优先级来判断函数在某个调用位置应用的是哪条规则，可以按照下面的顺序来判断：
1. 函数是否在new中调用？如果是的话，this绑定的是新创建的对象
2. 函数是否通过call,apply显式绑定或者硬绑定调用？如果是的话，this绑定的是指定的对象
3. 函数是否在某个上下文对象中调用？如果是的话，this绑定的是那个上下文对象
4. 如果都不是的话，使用默认绑定。如果在严格模式下，就绑定到undefined，否则绑定到全局对象。

## 绑定例外
### 被忽略的this
如果你把null或者undefined作为this的绑定对象传入call,apply或者bind，这些值在调用时会被忽略，实际应用的是默认绑定规则。
```js
function foo() {
    console.log(this.a);
}
var a = 2;
foo.call(null); //2
```
什么情况下会传入null呢？
一种常见的做法是使用apply()来展开一个数组，并当作参数传入一个函数。
```js
function foo(a,b) {
    console.log('a:'+a+',b:'+b);
}
foo.apply(null, [2,3]); //a:2,b:3
```
> 在ES6中可以用...操作符代替apply()来展开数组

然后，总是使用null来忽略this绑定可能产生一些副作用。如果某个函数确实使用了this，那默认绑定规则会把this绑定到全局对象（在浏览器中这个对象是window），这将导致不可预计的后果（比如修改全局对象）。这种方式可能会导致许多难以分析和追踪的bug.

### 更安全的this
一种更安全的做法是传入一个特殊的对象，把this绑定到这个对象不会对你的程序产生任何副作用。
在javascript中创建一个空对象的最简单的做法是`Object.create(null)`。这和{}很想，但是并不会创建Object.prototype这个委托，所以比{}更空。
```javascript
function foo(a,b) {
    console.log( "a:" + a + ", b:" + b );
}

// 我们的DMZ空对象
var ø = Object.create( null );

// 把数组展开成参数
foo.apply( ø, [2, 3] ); // a:2, b:3

// 使用bind(..)进行柯里化
var bar = foo.bind( ø, 2 );
bar( 3 ); // a:2, b:3”

```
### 间接引用
另一个需要注意的是，你有可能创建一个函数的“间接引用”，在这种情况下，调用这个函数会应用默认绑定规则。
```js
function foo() {
    console.log( this.a );
}

var a = 2;
var o = { a: 3, foo: foo };
var p = { a: 4 };

o.foo(); // 3
(p.foo = o.foo)(); // 2”
```
复制表达式p.foo = o.foo的返回值是秒表函数的引用，因此调用位置是foo()而不是p.foo()或者o.foo()，这里会应用默认绑定。
*注意：对于默认绑定来说，决定this绑定对象的并不是调用位置是否处于严格模式，而是函数体是否处于严格模式。如果函数体处于严格模式，this会被绑定到undefined，否则this会被绑定到全局对象。*

### 软绑定
之前我们看到过，硬绑定这种方式可以把this强制绑定到指定的对象（除了使用new时），防止函数调用应用默认绑定规则。问题在于，硬绑定会大大降低函数的灵活性，使用硬绑定之后就无法使用隐式绑定或者显式绑定来修改this的能力。
如果可以给默认绑定指定一个全局对象和undefined以外的值，就可以实现和硬绑定相同的效果，同时保留隐式绑定或者显式绑定修改this的能力。
```js
if (!Function.prototype.softBind) {
    Function.prototype.softBind = function(obj) {
        var fn = this;
        // 捕获所有 curried 参数
        var curried = [].slice.call( arguments, 1 );
        var bound = function() {
            return fn.apply(
                (!this || this === (window || global)) ?
                    obj : this
                curried.concat.apply( curried, arguments )
            );
        };
        bound.prototype = Object.create( fn.prototype );
        return bound;
    };
}
```

## this词法
我们之前介绍的四条规则已经可以包含所有的正常的函数。但是ES6中添加了一种无法使用这些规则的特殊函数类型：箭头函数
箭头函数并不是使用function关键字定义的，而是使用被称为“胖箭头”的操作符=>定义的。箭头函数不适用this的四种标准规则，而是根据外整作用域来决定this.
```js
function foo(){
    return (a)=>{
        //this继承自foo()
        console.log(this.a);
    }
}
var obj1 = {
    a:2
}
var obj2  = {
    a:3
}
var bar = foo.call(obj1);
bar.call(obj2);//2
```
foo()内部创建的箭头函数会捕获调用foo()时的this.由于foo()的this绑定到obj1,bar(引用箭头函数)的this也会绑定到obj1,箭头函数的绑定无法被修改。

## 小结
如果要判断一个运行中的函数的this的绑定，就需要找到这个函数的直接调用位置。找到之后就可以顺序应用下面的四条规则来判断this的绑定对象：
1. 由new调用？绑定到新创建的对象
2. 由call或 apply或bind调用？绑定到指定的对象
3. 由上下文对象调用？绑定到那个上下文对象
4. 默认：在严格模式下，绑定到undefined，否则绑定到全局对象。

一定要注意，有些调用可能在无意中使用默认绑定规则，如果想“更安全”地忽略this绑定，你可以使用一个DMZ对象，比如 `ø=Object.create(null)`,以保护全局对象。
ES6中的箭头函数并不会使用四条标准的绑定规则，而是根据当前的词法作用域来决定this,具体来说，箭头函数会继承外层函数调用的this绑定（无论this绑定到哪里）。这其实和ES6之前代码中的self=this的机制一样。
