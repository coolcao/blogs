---
title: 换一种角度看Javascript的面向对象-行为委托
date: 2016-11-27 01:41:53
tags: [js,面向对象]
categories:
- 技术博客
- 原创
---

javascript是一门面向对象语言，这一点应该毫无疑问。不是有句话这么说的么，*js中万物皆对象*，就连函数的本质都是对象，因此js里函数也是一等公民。
虽然js是面向对象的语言，但是Js却没有类的概念，其继承方式也和基于类的面向对象语言有所不同，是基于原型链的继承。
但，很久以来，我们在尝试说明js的原型链的机制时，都是用“类”的概念去做类比，构造函数，实例，继承。就连最新的ES6都添加了`class`关键字，这让人们越来越迷糊，js到底存不存在类，它的原型到底本质和类有什么区别？

<!--more-->

## 基于类和基于原型
面向对象语言可分为两类：基于类的面向对象，这类的代表有Java，还有一类是基于原型的面向对象，这类的代表有Javascript。
在基于类的面向对象语言中，对象（object）依靠类（class）类产生，由类构造而来。而继承的实现也是基于类。
而在基于原型的面向对象语言中，对象则是依靠构造器（constructor），利用原型（prototype）构造出来，所有的对象都离不开原型。


## js的继承
面向对象的语言中，免不了的就是“继承”两个字。在Java中，继承指的是类与类之间的关系，子类继承父类，这样子类就可以拥有父类的一些方法和属性。
在js中，“继承”是通过原型实现的。很多人都这么说，网上能搜到的，也这么说。
那么，仔细想想，原型以及原型链讲的是对象与对象的关系，而继承，据我所知讲的是类与类之间的关系。
很多年以来，人们在解释js的原型链时，都是按照类的思路来，因此会造成很多不大不小的误解。

什么是原型，原型链，这里不做太细致的讨论。实在不记得，可以再回去翻翻书。

一直以来，我们写js代码时，都是按照类的思想来写，也试图按照类的思想来解释js中的原型，比如：

```js
var Person = function Person(name){
  this.name = name;
}
Person.prototype.say = function say(){
  console.log('my name is ' + this.name);
}
var Student = function Student(name){
  Person.call(this,name);
  this.type = 'student';
}
Student.prototype = Object.create(Person.prototype);

var s1 = new Student('coolcao');
s1.say(); //my name is coolcao
```

我们使用类的概念，将Person,Student称作父类，子类，将s1称作实例对象。
s1是通过new调用Student实例化而来。
但这和基于类的面向对象有本质的区别，基于类的实例化是由复制而来，而基于原型的面向对象，则是通过原型产生。

我们来分析一下以上代码的原型图：

![原型图](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/prototype_class_extends.png)

Person,Student分别是两个构造函数，他们的prototype分别指向两个原型对象，而Sutdent.prototype的[[Prototype]]指向Person.prototype，s1为Student的一个实例对象，其[[Prototype]]指向Sutdent.prototype对象，这样就形成了原型链：
`s1 -> Student.prototype -> Person.prototype`
所以从上图看，原型链也是对象与对象之间的关系。

从上面的代码中，看出js中原型链实现的继承，代码还是有点繁琐。幸好ES6中实现了`class`,`extends`关键字，因此上面代码如果用ES6编写，将会是如下：

```js
class Person{
  constructor(name){
    this.name = name;
  }
  say(){
    console.log('my name is ' + this.name);
  }
}

class Student extends Person{
  constructor(name){
    super(name);
    this.type = 'student';
  }
}
var s1 = new Student('coolcao');
s1.say(); //my name is coolcao
```

ES6的代码更简洁，而且可读性更强。看上去和Java等基于类的面向对象更贴近了。但是，请记住，`class`,`extends`只是对原型的语法糖而已，js的内部实现机制，还是用原型链实现的。
基于上面的原因，越来越误导人们，使用基于类的思想去思考javascript的原型机制。
可是，在这里，我想说的是，js的原型机制，应该是用*委托*来思考更合理，而不是类。
为什么这么说呢，因为javascript的原型链，都是对象与对象之间的关系，对象与对象之间，更像委托，而不是父子。

## 行为委托
委托是说，我这里没有，但是你那里有，我就委托你帮我做这件事。
比如上面的原型图，对于对象s1而言，它有的仅仅是name和type两个属性，而没有say()方法，但最后却能调用say()成功，是因为，s1没有这个方法，但是它的原型链上，Person.prototype对象有say()方法，于是，没有这个方法没关系，我可以委托Person.prototype对象给做这这件事嘛。
所以，委托的本质，就是对象与对象之间的关键，这和原型链是一致的。
所以，如果我们用委托的思想来重新思考一下上面的代码，将构造函数去掉，只剩下原型对象呢：

```js
var Person = {
  init:function(name){
    this.name = name;
  },
  say:function(){
    console.log('my name is ' + this.name);
  }
}

var Student = Object.create(Person);
Student.settup = function(name){
  this.init(name);
  this.type = 'student';
}
Student.sayHi = function(){
  this.say();
}
var s1 = Object.create(Student);
s1.settup('coolcao');
s1.sayHi();
```

这段代码简洁了许多，我们只是把对象关联起来，并不需要那些既复杂又令人困惑的模仿类的行为（构造函数、原型以及new）。
这里，Person,Student是两个对象，而非函数。Student的原型对象指向Person,而s1对象的原型指向Student，因此三者构成了原型链：`s1 -> Student -> Person`，这里没有了构造函数，没有了类的误导，同样也可以很好的工作。
而且原型链关系更简洁，就是上面原型链图，去掉了构造函数 ，将Person.prototype和Student.prototype替换为Person和Student两个对象。

![原型图](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/prototype_proxy.png)

对象Student和s1都是通过Object.create()创建，Student委托了Person,s1委托了Student。
因此，在这里，使用委托去理解javascript的原型链更符合常规，更清晰易懂。

## 小结
我们从委托的角度，理解了一下js的原型链，发现一切都变的那么明朗。但是，为什么这么些年来，大家都用类的角度去分析，然后模拟类和继承来实现复杂的js代码呢？
其实很简单，用基于类的思想去组织管理代码，维护性更好，也更符合大型系统的开发思维。虽然如此理解js的原型有些偏颇。
尤其ES6出了`class`和`extends`两个关键字后，代码看起来更像*类式面向对象*编程语言。
但在理解上，把原型链理解为委托似乎更妥些。
仁者见仁，智者见智。

## 参考
* 《你不知道的javascript》
* [全面理解面向对象的 JavaScript](http://www.ibm.com/developerworks/cn/web/1304_zengyz_jsoo)
* [拥抱原型面向对象编程](http://www.ibm.com/developerworks/cn/web/wa-protoop)
