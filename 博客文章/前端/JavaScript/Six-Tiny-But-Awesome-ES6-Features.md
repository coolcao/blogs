---
title: 【译】ES6中6个小的却令人惊叹的特性
date: 2016-11-16 22:04:20
tags: [js,es6]
categories:
- 技术博客
- 翻译
---

Javascript 团体的每个人都喜欢新的API,语法更新以及特性，它们提供了更好的，更智能，更有效的方式以完成重要的任务。ES6提供了大量新的好的东西，在过去的一年内，浏览器提供商做了大量的辛勤工作将新的语言特性更新到他们的浏览器中。尽管有重大的更新，很多小的语言更新另我眼前一亮，下面6条就是我最爱的Javascript新特性：

<!--more-->

## Object[key] 语法
一个有经验的Javascript开发者，在字面量定义对象时，也不能够将变量作为对象的键值，你必须定义完对象后再添加键/值对：
```javascript
// *Very* reduced example
let myKey = 'key3';
let obj = {
    key1: 'One',
    key2: 'Two'
};
obj[myKey] = 'Three';
```

往好了说，这种模式是不方便的，往坏了说，它是混乱和丑陋的。ES6提供了一种新的方式用以消除这种混乱：
```javascript
let myKey = 'variableKey';
let obj = {
    key1: 'One',
    key2: 'Two',
    [myKey]: 'Three' /* yay! */
};
```

这种将变量作为对象的键使用`[]`包起来的方式使得开发者能够在一条语句中就能完成。

## 箭头函数（Arrow Functions）

你不必关注ES6的每一个变化来了解箭头函数，他们曾是Javascript开发者争论和混乱的源头。我会发布多篇博文来解释箭头函数每个特性。我将会指出箭头函数是如何提供方法简化代码的。
```javascript
// Adds a 10% tax to total
let calculateTotal = total => total * 1.1;
calculateTotal(10) // 11

// Cancel an event -- another tiny task
let brickEvent = e => e.preventDefault();
document.querySelector('div').addEventListener('click', brickEvent);
```

没有`function`或`return`关键字，有时甚至不需要`()`，箭头函数在代码上比普通函数更简化。

## find/findIndex
Javascript提供了`Array.prototype.indexOf`用以获取数组中给定项的索引，但是`indexOf`没有提供方法来计算想要的项的条件。比如你需要搜索一个更准确的值。`find`和`findIndex`两个方法用来搜索数组中第一个匹配计算结果的项：
```javascript
let ages = [12, 19, 6, 4];

let firstAdult = ages.find(age => age >= 18); // 19
let firstAdultIndex = ages.findIndex(age => age >= 18); // 1
```
find和findIndex通过允许计算值搜索，防止不必要的副作用以及循环通过可能的值。

## 展开操作符 ...
展开操作符将数组或可遍历对象切分为独立的参数，下面是一些例子：
```javascript
// Pass to function that expects separate multiple arguments
// Much like Function.prototype.apply() does
let numbers = [9, 4, 7, 1];
Math.min(...numbers); // 1

// Convert NodeList to Array
let divsArray = [...document.querySelectorAll('div')];

// Convert Arguments to Array
let argsArray = [...arguments];
```

令人敬畏的好处是，能够转换可遍历对象（如NodeList,arguments等）为真正的数组，我们用Array.from或其他技术做了很长时间的事。

## 模板字符串
多行字符串，一直以来，我们一直[使用连接符或`\`符](https://davidwalsh.name/multiline-javascript-strings)，这两种方法都不好维护。有很多开发者，甚至是框架的发起者滥用`<script>`标签来包含多行模板，其他人使用Dom创建了元素，然后使用outerHTML来获取元素的html作为字符串。
ES6提供了[模板字符串](https://davidwalsh.name/template-literals)，你可以简单的使用反逗号（`）符创建多行字符串。
```javascript
// Multiline String
let myString = `Hello

I'm a new line`; // No error!

// Basic interpolation
let obj = { x: 1, y: 2 };
console.log(`Your total is: ${obj.x + obj.y}`); // Your total is: 3
```

当然，模板字符串的功能可不仅仅是允许你创建多行字符串，像简单到高级的插值，但只有创建多行字符串的能力让我兴奋。

## 默认参数
很多服务器端的语言，像 python和php都提供函数的默认参数值，现在javascript也有了这种能力：
```javascript
// Basic usage
function greet(name = 'Anon') {
  console.log(`Hello ${name}!`);
}
greet(); // Hello Anon!

// You can have a function too!
function greet(name = 'Anon', callback = function(){}) {
  console.log(`Hello ${name}!`);

  // No more "callback && callback()" (no conditional)
  callback();
}

// Only set a default for one parameter
function greet(name, callback = function(){}) {}
```

其他编程语言中，如果没有为参数提供默认值，可能会抛出警告，但是在javascript里，它会继续运行，并且将这些参数的值设置为`undefined`。

我这里所列出的6个特性，只是es6提供给开发者的很小的一部分，但确实我们可能想都不想就经常用的。这些微小的补充，往往不会得到注意，但却成了我们编码的核心。

## 讨论
### [cmwd](http://cmwd.github.io):对于默认参数，我经常用以下小技巧：
```javascript
const isRequired = () => { throw new Error('param is required'); };

const hello = (name = isRequired()) => { console.log(`hello ${name}`) }
```

#### [Dimitar](http://fragged.org):+1,不过为了更好的可读性，使用 getter 比 ()更好：
```javascript
const is = {
  get required(){
    throw new Error('Required argument')
  }
}

import { is } from 'utils'

const foo(name = is.required)=>Array.from(name)
```

英文原文链接：[Six Tiny But Awesome ES6 Features](https://davidwalsh.name/es6-features)
