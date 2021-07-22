---
title: 学习笔记-你不知道的js-作用域闭包
date: 2016-11-18 00:17:05
tags: [js,闭包]
categories:
- 学习笔记
- 你不知道的js
---

Javascript中闭包无处不在，你只需要能够识别并拥抱它。　
闭包是基于词法作用域书写代码所产生的自然结果，你甚至不需要为了利用它们而有意识的创建闭包。

<!--more-->

## 实质问题
**当函数可以记住并访问所在的词法作用域时，就产生了闭包，即使函数是在当前词法作用域之外执行。**
```Javascript
function foo() {
    var a = 2;
    function bar() {
        console.log(a);
    }
    return bar;
}
var baz = foo();
baz();  //2,这就是闭包
```
函数bar()的词法作用域能够访问foo()的内部作用域。然后我们将bar()函数本身当作一个值类型进行传递。在这个例子中，我们将bar所引用的函数对象本身当作返回值。
在foo()执行后，其返回值（也就是内部的bar()函数）赋值给变量baz并调用baz()，实际上只是通过不同的标识符调用了内部的函数bar().
bar()显然可以被正确执行，但是它是在自己定义的词法作用域*以外*的地方执行.
foo()执行后，由于baz仍对bar()函数有引用，因此foo内部作用域依然存在，没有被垃圾回收。bar()依然持有对该作用域的引用，而这个引用就是闭包。
当然，无论使用何种方式对函数类型的值进行传递，当函数在别处被调用时都可以观察到闭包。

## 熟悉的闭包
```Javascript
function wait(message){
    setTimeout(function timer(){
        console.log(message);
    },1000)
}
wait('hello,closure');
```
将一个内部函数timer传递给setTimeout()。timer具有涵盖wait()作用域的闭包，因此还保有对变量message的引用。wait()执行1000毫秒后，**他的内部作用域并不会消失**，timer函数依然保有wait()作用域的闭包。
> 这里有个疑问，如果wait()函数执行后，它的内部作用域不会消失，那么是不是意味着垃圾回收机制永远不会回收？或者什么时候回收？

在定时器、事件监听器、 Ajax 请求、跨窗口通信、Web Workers 或者任何其他的异步(或者同步)任务中，只要使 用了回调函数，实际上就是在使用闭包!

## 循环和闭包
要说明闭包，for循环是常用的例子：
```Javascript
for(var i=1;i<=5;i++){
    setTimeout(function timer(){
        console.log(i);
    },i*1000);
}
```
写这样的代码，可能预期是输出数字1~5，每秒一个，但实际却是，这段代码在运行时，以每秒一次的频率输出5次6.
延迟函数的回调会在循环结束时才执行，当定时器运行时，即使每个迭代中执行的是setTimeout(..,0)，所有的回调函数依然是在循环结束后才执行。因此会每次输出一个6.
这里的缺陷是，我们试图假设循环中每个迭代在运行时都会给自己捕获一个i的副本。但是根据作用域的工作原理，实际情况是尽管循环中的五个函数是在各个迭代中分别定义的，但是他们都**被封闭在一个共享的全局作用域中**，因此实际上只有一个i.

我们需要更多的闭包作用域来隔离，特别是在循环的过程中每个迭代都需要一个闭包作用域。

```Javascript
for (var i = 1; i <= 5 ; i++) {
    (function(){
        var j = i;
        setTimeout(function timer() {
            console.log(j);
        },j*1000);
    })();
}
```
这样就可以了。我们为每个迭代都创建一个闭包作用域，然后将i的副本赋值给每个闭包中的j，这里timer()执行的时候访问的就是每个闭包作用域中的变量j.
我们可以将代码改良一下：
```Javascript
for (var i = 1; i <= 5 ; i++) {
    (function(j){
        setTimeout(function timer() {
            console.log(j);
        },j*1000);
    })(i);
}
```

## let与块作用域
上面的例子中，每次迭代时，我们都会创建一个新的作用域，或者说，每次迭代我们都**需要一个块作用域**。
在第三章中，我们已经知道了es6的let，也做个实验说过。
```Javascript
for (let i = 1; i <= 5; i++) {
    console.log(i);
}
```
for循环头部的let声明还会有一个特殊的行为，这个行为指出**变量在循环的过程中不止被声明一次，每次迭代都会声明，随后的每个迭代都会使用上一个迭代结束时的值来初始化这个变量**。

## 模块
还有其他的代码模式使用闭包的强大威力，但从表面看，他们似乎与回调无关。下面研究下其中最强大的一个：**模块**
```javascript
function CoolModule(){
    var something = 'cool'
    var another = [1,2,3]
    function doSomething(){
        console.log(something);
    }
    function doAnother(){
        console.log(another.goin('!'));
    }
    return {
        doSomething:doSomething,
        doAnother:doAnother
    }
}
var foo = CoolModule();
foo.doSomething();
foo.doAnoter();
```
这个模式在Javascript中被称为模块，最常见的实现模块的方法通常被称为模块暴露。

首先，CoolModule()只是一个函数，必须要通过调用它来创建一个实例。如果不执行外部函数，内部作用域和闭包都无法被创建。
其次，CoolModule()返回一个用对象字面量语法来表示的对象。这个返回的对象中含有对内部函数而不是内部数据变量的引用。我们保持内部数据变量是隐藏且私有的状态。可以将这个对象类型的返回值看作本质是模块的公共API.

> 从模块中返回一个实际的对象并不是必须的，也可以返回一个内部函数。jQuery就是一个例子。jQuery和$标识符就是jQuery模块的公共API，但他们本身都是函数。

doSomething和doAnother函数具有涵盖模块实例内部作用域的闭包（通过调用CoolModule实现）。当通过返回一个含有属性引用的对象的方式来将函数传递到词法作用域外部时，我们已经创造了可以观察和实践闭包的条件。
如果要更简单的描述，模块模式需要具备两个必要条件：
* 必须有外部的封闭函数，该函数必须至少被调用一次（每次调用都会创建一个新的模块实例）
* 封闭函数必须返回至少一个内部函数，这样内部函数才能在私有作用域中形成闭包，并且可以访问或者修改私有的状态。

模块模式的另一个简单但强大的用法是命名将要作为公共API返回的对象：
```js

var foo = (function CoolModule(id) {
    function change() {
        // 修改公共API
        publicAPI.identify = identify2;
    }

    function identify1() {
        console.log(id);
    }

    function identify2() {
        console.log(id.toUpperCase());”

        “
    }


    var publicAPI = {
        change: change,
        identify: identify1
    };

    return publicAPI;
})("foo module");

foo.identify(); // foo module
foo.change();
foo.identify(); // FOO MODULE”

```

通过在模块实例的内部保留对公共API对象的内部调用，可以从内部对模块实例进行修改，包括添加或删除方法和属性，以及修改他们的值。

### 现代的模块机制
大多数模块依赖加载器 / 管理器本质上都是将这种模块定义封装进一个友好的 API。
### 未来的模块机制
ES6中为模块增加了一级语法支持，但通过模块系统进行加载时，ES6会将文件当作独立的模块来处理。每个模块都可以导入其他模块或特定的API成员，同样也可以导出自己的API成员。

  
>基于函数的模块并不是一个能被稳定识别的模式(编译器无法识别)，它们 的 API 语义只有在运行时才会被考虑进来。因此可以在运行时修改一个模块 的 API(参考前面关于公共 API 的讨论)。
>
>相比之下，ES6 模块 API 更加稳定(API 不会在运行时改变)。由于编辑器知 道这一点，因此可以在(的确也这样做了)编译期检查对导入模块的 API 成 员的引用是否真实存在。如果 API 引用并不存在，编译器会在运行时抛出一 个或多个“早期”错误，而不会像往常一样在运行期采用动态的解决方案。


## 小结
当函数可以记住并访问所在的词法作用域，即使函数是在当前词法作用域之外执行，这时 就产生了闭包。

模块有两个主要特征:(1)为创建内部作用域而调用了一个包装函数;(2)包装函数的返回 值必须至少包括一个对内部函数的引用，这样就会创建涵盖整个包装函数内部作用域的闭 包。

