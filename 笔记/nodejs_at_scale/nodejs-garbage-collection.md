---
title: 【译】【5】nodejs垃圾回收机制
date: 2016-12-22 19:13:54
tags: [nodejs]
categories:
- 技术博客
- 翻译
- Nodejs At Scale
---

在本篇文章中，你将学习到，nodejs的垃圾回收机制是如何工作的，当你编写代码时，后台都发生了什么，以及系统是如何为你清理内存的。

每个应用程序都需要内存才能正常工作，内存管理提供了程序在请求时为其动态分配内存的方法，并在程序不需要它们时，释放它们，以便重用它们。

<!--more-->

## Nodejs中的内存管理

应用级的内存管理可以是手动的，也可以是自动的。自动内存管理通常涉及到垃圾收集器。

以下的代码片段展示了在C语言中，使用手动内存管理方式如何分配内存：

```c

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main() {

   char name[20];
   char *description;

   strcpy(name, "RisingStack");

   // memory allocation
   description = malloc( 30 * sizeof(char) );

   if( description == NULL ) {
      fprintf(stderr, "Error - unable to allocate required memory\n");
   } else {
      strcpy( description, "Trace by RisingStack is an APM.");
   }

   printf("Company name = %s\n", name );
   printf("Description: %s\n", description );

   // release memory
   free(description);
}

```

在手动内存管理中，释放内存中不需要的部分是开发者的责任。以这种方式管理内存可能会给应用带来几个主要的bug:

* **内存泄漏**:当使用的内存空间从未释放时
* **野指针/悬挂指针**将出现，当一个对象被删除，但是指针被重新使用。当其他数据结构呗复写或者敏感信息被读取时，可能会引入严重的安全问题。

**幸运的是，nodejs自带了垃圾回收器，你不需要手动管理内存分配。**

## 垃圾回收机制的概念
垃圾回收是一种自动管理应用内存的方式。垃圾回收器（GC）的工作是回收不再使用对象（垃圾）占用的内存。它是在1959年首次用于LISP，由John McCarthy发明。

**垃圾回收之前的内存**
下面这张图展示了，如果你有多个对象引用彼此，以及一些对象不再引用其他对象时，内存看起来的样子。这些不再被引用的对象将是垃圾回收器收集的对象。

![垃圾回收前](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/Node_js_Garbage_Collection_Explained_-_Node_js_at_Scale____RisingStack.png)

**垃圾回收后的内存**
一旦垃圾回收期运行后，那些无法访问的对象将被删除，并释放内存空间。

![垃圾回收后](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/Node_js_Garbage_Collection_Explained_-_Node_js_at_Scale____RisingStack2.png)

### 垃圾回收的益处
* 避免了野指针/悬挂指针的bug
* 它不会尝试释放已经释放的空间
* 保护你免受一些类型的内存泄漏

当然，使用垃圾回收器并不能解决你所有的问题，它并不是内存管理的万金油。让我们来看看你应该注意的方面！

### 使用垃圾回收器应注意
* **性能影响**:为了决定哪些可以被释放，GC会消耗计算资源
* **不可预测的中断**:现代的GC试图避免“停止一切”的情况发生，但是还是会不可避免的出现。

## Nodejs垃圾回收和内存管理实践
学习最简单的方式就是去实践。因此我会使用几个代码片段向你展示内存中发生了些什么。

### 栈
栈包含局部变量和指向堆中对象的指针，或者指向定义应用程序控制流的指针。

在下面这个例子中，a和b都会存储在栈中：

```js
function add (a, b) {
	  return a + b
}

add(4, 5)
```

### 堆
堆专用于存储引用类型对象，比如字符串或者对象。

下面代码中创建的`Car`对象将存储在堆中：

```js
function Car (opts) {
	  this.name = opts.name
}

const LightningMcQueen = new Car({name: 'Lightning McQueen'})
```

这之后，内存看上去像这个样子：

![栈和堆](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/Node_js_Garbage_Collection_Explained_-_Node_js_at_Scale____RisingStack3.png)

让我们创建更多的汽车实例，让我们看看内存是怎样的！

```js
function Car (opts) {
	  this.name = opts.name
}

const LightningMcQueen = new Car({name: 'Lightning McQueen'})
const SallyCarrera = new Car({name: 'Sally Carrera'})
const Mater = new Car({name: 'Mater'})
```

![更多对象](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/Node_js_Garbage_Collection_Explained_-_Node_js_at_Scale____RisingStack4.png)

如果这个时候执行GC，那么没有空间将会被释放，因为根对象root引用了每个对象。

让我们把它做的更有意思些，给汽车添加些零件：

```js
function Engine (power) {
	  this.power = power
}

function Car (opts) {
	  this.name = opts.name
	    this.engine = new Engine(opts.power)
}

let LightningMcQueen = new Car({name: 'Lightning McQueen', power: 900})
let SallyCarrera = new Car({name: 'Sally Carrera', power: 500})
let Mater = new Car({name: 'Mater', power: 100})
```

![5](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/Node_js_Garbage_Collection_Explained_-_Node_js_at_Scale____RisingStack5.png)

如果我们不再使用Mater，但是依然给他赋值一些其他的值，比如`Mater = undefined`，将会发生什么呢？

![6](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/Node_js_Garbage_Collection_Explained_-_Node_js_at_Scale____RisingStack6.png)

结果是，root不再引用Mater，那么，当垃圾回收下次运行时，它将会被释放。

![7](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/Node_js_Garbage_Collection_Explained_-_Node_js_at_Scale____RisingStack7.png)

现在我们了解了垃圾回收器预期行为的基础知识，让我们来看一下V8中是如何实现的！

## 垃圾回收方法

在我们之前的一篇文章中，我们讨论了[nodejs中垃圾回收的工作原理](https://blog.risingstack.com/finding-a-memory-leak-in-node-js/)，因此我强烈建议重新阅读一下这篇文章。

这里是你学到的最重要的几种方法：

### 新生代空间和老生代空间
在堆中存在两个主要的“段”（segments）,新生代空间和老生代空间。新生代空间是新内存分配的地方，它有1~8MB的大小，但是垃圾回收却很快很频繁。这里存储的对象被称为新生代。


老生代空间存储的对象是由新生代空间中从垃圾回收存活下来的对象提升而来，它们被称为老生代。老生代空间中，分配内存是快速的，但是回收却是昂贵的，因此它很少执行。

### 新生代
通常，大约20%的新生代存活为老生代。老生代的垃圾回收将只发生在空间耗尽时。为此V8引擎使用了两种不同的收集算法。

### 扫描和标记清除
扫描收集是快速的，并且用于新生代。而标记清除收集则相对较慢，用于老生代。

## 更多阅读

* [Finding a memory leak in Node.js](https://blog.risingstack.com/finding-a-memory-leak-in-node-js/)
* [JavaScript Garbage Collection Improvements - Orinoco](https://blog.risingstack.com/javascript-garbage-collection-orinoco/)
* [memorymanagement.org](http://www.memorymanagement.org/)
