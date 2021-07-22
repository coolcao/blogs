---
title: 学习笔记-你不知道的js-提升
date: 2016-11-14 22:17:14
tags: [js]
categories:
- 学习笔记
- 你不知道的js
---

## 提升
引擎会在解释javascript代码之前首先对其进行编译。编译阶段的一部分工作就是找到所有的声明，并用合适的作用域将它们关联起来。

包括`变量`和`函数`在内的所有声明都会在*任何*代码被执行前首先被处理。

<!--more-->

`var a = 2;`，Javascript实际上会将其堪称两个声明：`var a;`和`a = 2;`。第一个定义声明是在编译阶段进行的。第二个赋值声明会被留在原地等待执行阶段。

如下代码：
```javascript
console.log(a);
var a = 2;
```

将会以如下的形式进行处理：
```javascript
var a;
a = 2;
console.log(a);
```
其中第一部分是编译，而第二部分是执行。

**先有声明，后有赋值**

> * 只有声明本身会被提升，而赋值或其他运行逻辑会留在原地。
> * 每个作用域都会进行提升操作。

```javascript
foo();  //undefined
function foo(){     //这里注意是函数声明。区分函数声明和函数表达式
    console.log(a);
    var a = 2;
}
//以上代码会以被理解成如下：
function foo(){
    var a;
    console.log(a);
    a = 2;
}
foo();
```

**函数声明会被提升，而函数表达式不会**

```javascript
foo();  //不是ReferenceError，而是 TypeError
var foo = function (){
    //...
}
//以上代码将被理解为：
var foo;
foo();
foo = function(){
    //...
}
```
foo标识符被提升，foo()由于对undefined值进行函数调用而导致非法操作，因此抛出TypeError异常。

同时也要记住，即使是具名的函数表达式，名称标识符在赋值之前也无法在所在的作用域中使用：
```javascript
foo();
bar();
var foo = function bar(){
    //...
}
//这段代码经过提升后，被理解为如下形式：
var foo;
foo();
bar();
foo = function (){
    var bar = ...self...
    //...
}
```

## 函数优先
**函数声明和变量声明都会被提升，但是函数会被首先提升，然后才是变量**。

```javascript
foo();  //1
var foo;
function foo(){
    console.log(1);
}
foo = function(){
    console.log(2);
}
//会输出1而不是2，这个代码片段会被引擎理解为如下：
function foo(){
    console.log(1);
}
foo();
foo = function(){
    console.log(2);
}
```
尽管`var foo`出现在function foo()的声明之前，但它是重复声明，因此被忽略了，因为函数声明被提升到普通变量之前。
尽管重复的var 声明会被忽略，但出现在后面的函数声明还是可以覆盖前面的。

```javascript
foo();  //3
function foo(){
    console.log(1);
}
var foo = function(){
    console.log(2);
}
function foo(){
    console.log(3);
}
```

## let
let是es6新添加的关键字，用以声明变量。let声明的变量，用于块级作用域，并且不存在变量提升。
```javascript
console.log(a); //ReferenceError: a is not defined
let a = 2;
```

## 普通块内函数声明提升
一个普通块内的函数声明通常会被提升到所在的作用域顶部。这个过程不会像下面的代码暗示的那样可以被条件判断所控制：
```javascript
foo();  //b
var a = true;
if(a){
    function foo(){console.log('a');}
}else{
    function foo(){console.log('b');}
}
```
**但是注意这个行为并不可靠，在javascript未来的版本中有可能发生改变。因此禁止在块内声明函数。**
**在node.js 6以上的版本中，上面代码会报错：ReferenceError: foo is not defined.**

## 小结
* javascript引擎将`var a = 2;`当作两个单独的声明：`var a;`和`a = 2;`，第一个是编译阶段的任务，第二个是执行阶段的任务。
* 无论作用域中的声明出现在什么地方，都将在代码本身被执行前首先进行处理。（变量提升）
* 声明本身会被提升，而包括函数表达式的赋值在内的赋值操作不会被提升
* 避免重复声明，特别是当普通的var声明和函数声明混在一起的时候，否则会引起很多危险的问题。
* es6的let不存在变量提升。
