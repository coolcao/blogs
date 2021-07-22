---
title: 学习笔记-你不知道的js-函数作用域和块作用域
date: 2016-11-13 22:17:14
tags: [js]
categories:
- 学习笔记
- 你不知道的js
---

## 函数中的作用域
每声明一个函数都会为其自身创建一个作用域。

<!--more-->

函数作用域的含义是指，属于这个函数的全部变量都可以在整个函数的范围内使用及复用（事实上在嵌套的作用域中也可以使用）。

## 隐藏内部实现
从所写的代码中挑选出一个任意的片段，然后用函数声明对他进行包装，实际上就是把这些代码“隐藏”起来了。
最小授权或最小暴露原则：应该最小限度的暴露必要内容，而将其他内容都“隐藏”起来，比如某个模块或对象的API设计。

### 避免冲突
“隐藏”作用域中的变量和函数所带来的另一个好处，是可以避免同名标识符之间的冲突，两个标识符可能具有相同的名字但用途却不一样，无意间可能造成命名冲突。
冲突会导致变量的值被意外覆盖。
#### 全局命名空间
当程序中加载了第三方库时，这些库通常会在全局作用域中声明一个名字足够独特的变量，通常是一个对象。这个对象被用作库的**命名空间**，所有需要暴露给外界的功能都会称为这个对象的属性。
> 比如TypeScript中就有**命名空间**这一概念，将所有功能相关的属性和方法都放到同一命名空间下，以达到更好的内聚效果。纯js中可以使用闭包实现（其实TypeScript的命名空间转译为js时就是使用的闭包）。
#### 模块管理
另一个避免冲突的办法和现代的*模块*机制接近，从众多的模块管理器中挑选一个来使用。使用这些工具，任何库都无需将标识符加入到全局作用域中，而是通过依赖管理器的机制将库的标识符显式的导入到另一个特定的作用域中。

### 函数作用域
在任意代码片段外部添包装加包装函数，可以将内部的变量和函数定义“隐藏”起来，外部作用域无法访问包装函数内部的任何内容。
但是它不并理想，首先必须声明一个函数名，这意味着这个名称本身“污染”了所在的作用域。其次必须显示的通过函数名调用这个函数才能运行其中的代码。
如果函数不需要函数名就能够自动运行，这将会更加理想。
两种方式：
1. 自执行函数
	```javascript
	var a = 2;
	(function(){
		var a = 3;
		console.log(a);
	})();
	console.log(a);
	```
1. 匿名函数
	```javascript
	setTimeout(function(){
		var a = 2;
		console.log(a);
	}, 1000);
	```
	> 匿名函数的缺点是：
	> 1. 匿名函数在函数栈中不会显示有意义的函数名，使得调试困难。
	> 2. 如果没有函数名，当函数需要引用自身时只能使用已经的arguments.callee引用，比如递归中。另一个函数需要引用自身的例子是，在事件触发后监听器需要解绑自身。

## 块作用域
块作用域是一个用来对之前的最小授权原则进行拓展的工具，将代码从在函数中隐藏信息拓展为在块中隐藏信息。
但可惜，表面上看Javascript并没有块作用域的相关功能。

### with
with是块作用域的一个例子，用with从对象中创建出来的作用域仅在with声明中而非外部作用域有效。

### try/catch
~~try/catch的catch分句会创建一个块作用域，其中声明的变量仅在catch内部有效~~。
> 这个地方说法不准确，这里只针对catch的err参数有效，如果在catch块中用var声明变量，在catch块外部是可以访问到这个变量的。

```Javascript
try{
    undefined();    //执行一个非法操作制造一个异常
}catch(err){
    console.log(err);   //能够正常执行
    var a = 'a';
}
console.log(err);   //ReferenceError:err not found
console.log(a);     //a
```

### let
let 关键字可以将变量绑定到任意作用域（通常是{..}内部）。
```Javascript
let foo = true;
if(foo){
    {
        let bar = 1;
        console.log(bar);   //1
    }
    console.log(bar);   //ReferenceError: bar is not defined
}
console.log(bar);       //ReferenceError: bar is not defined
```
只要声明是有效的，在声明中的任意位置都可以使用{..}为let创建一个用于绑定的块。在这个例子中，我们在if声明内部显示的创建了一个块，在块内使用let声明一个变量bar，可以看到，只有在块内才能访问的变量bar，而出了块，都访问不到bar.

#### 垃圾回收
另一个块作用域非常有用的原因和闭包及回收内存垃圾的机制有关。
```Javascript
function process(data){
    //做一些有趣的事
}
var someReallyBigData = {..};
process(someReallyBigData);
var btn = document.getElementById('mybtn')
btn.addEventListener('click',function click(evt){
    console.log('button clicked');
})
```

click点击函数并不需要someReallyBigData变量，当process()执行完后，在内存中占用大量空间的数据结构就可以被垃圾回收了。但是，**由于click函数形了一个覆盖整个作用域的闭包**，javascript引擎极有可能依然保存着这个结构（这取决于具体实现）。

块作用域就可以消除这种顾虑，可以让引擎清楚的知道，没有必要极选保存someReallyBigData了。
```javascript
function process(data){
    //做一些有趣的事
}
//这个块中定义的内容完事后可以销毁
{
var someReallyBigData = {..};
process(someReallyBigData);
}
var btn = document.getElementById('mybtn')
btn.addEventListener('click',function click(evt){
    console.log('button clicked');
})
```
为变量显示的声明块作用域，并对变量进行本地绑定是非常有用的工具。

#### let循环
一个let可以发挥优势的典型例子就是之前讨论的for循环。
```javascript
for(let i=0;i<10;i++){
    console.log(i);
}
console.log(i); //ReferenceError
```
for循环头部的let不仅将i绑定到了for循环的块中，**实际上它将其重新绑定了循环的每一个迭代中**，确保使用上一个循环迭代结束时的值重新进行赋值。
```javascript
//代码1
for (var i = 0; i < 5; i++) {
    setTimeout(function(){
        console.log(i);
    },1000);
}

//代码2
for (let i = 0; i < 5; i++) {
    setTimeout(function(){
        console.log(i);
    },1000);
}
```
看上面例子，基本有点经验的js程序员都会知道，代码1的写法是有问题的，最后输出的是5个5，并不是期望的0,1,2,3,4，但是代码2就可以，这正是上面所说的，let将变量i绑定到了循环的每一个迭代中，每次迭代使用i时，都会重新赋值。这里还涉及到闭包的问题，不细说，后面学习到闭包的时候再说。

### 块作用域的替代方案
随着es6新加入的let，javascript也随之有了创建完整，不受约束的块作用域的能力。但是如果是在es6之前的环境中，如何使用块作用域呢？
考虑下面代码：
```javascript
{
    let a = 2;
    console.log(a); //2
}
console.log(a); //ReferenceError
```
这段代码在es6中可以正常运行，但是在es6之前的环境中如从能实现这个效果呢？答案是`catch`：
```javascript
try {
    throw 2;
} catch (a) {
    console.log(a);
}
console.log(a); //ReferenceError
```
catch 具有这样的功能，但是写这样的代码实在蛋疼。所以最好的方式是，可以先用es6写代码，然后使用工具将其转成可以在之前环境中运行的代码。
