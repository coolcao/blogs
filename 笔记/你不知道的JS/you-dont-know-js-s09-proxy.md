---
title: 学习笔记-你不知道的js-行为委托
date: 2016-11-26 02:17:21
tags: [js,你不知道的js]
categories:
- 学习笔记
- 你不知道的js
---

`[[Prototype]]`机制就是指对象中的一个内部链接引用另一个对象。
如果在第一个对象上没有找到需要的属性或者方法引用，引擎就会继续在`[[Prototype]]`关联的对象上进行查找。同理，如果在后者中也没有找到需要的引用就会继续查找它的`[[Prototype]]`，依次类推。这一系列对象的链接被称为“原型链”。

换句话说，Javascript中这个机制的本质是**对象之间的关联关系**。

<!--more-->

## 面向委托的设计
我们需要试着把思路从类和继承的设计模式转换到委托行为的设计模式。

### 类理论
假设我们需要在软件中建模一些类似的任务（XYZ,ABC等）。

如果使用类，那设计方法可能是这样的：定义一个通用父类，可以将其命名为Task，在Task类中定义所有的任务都有的行为。接着定义子类XYZ和ABC，它们都继承自Task并且添加一些特殊的行为来处理对应的任务。
类设计模式鼓励你在继承时使用方法重写，比如在XYZ任务中重写Task中定义的一些通用方法，甚至在添加新行为时通过super调用这个方法的原始版本。你会发现许多行为可以先“抽象”到父类然后再用子类进行特殊化（重写）。

下面是对应的伪代码：

```java
class Task{
  id;
  Task(Id){id = Id}
  outputTask(){output(id)}
}
class XYZ inherits Task{
  label;
  XYZ(Id,Label){super(Id);lebal=Label;}
  outputTask(){super();output(label);}
}
class ABC inherits Task{
  //...
}
```

现在你可以实例化子类XYZ的一些副本然后使用这些实例来执行任务“XYZ”。这些实例会复制Task定义的 通用行为以及XYZ定义的特殊行为。

### 委托理论
但是现在我们使用使用*委托行为*而不是类来思考同样的问题。
首先你会先定义一个名为Task的对象，它会包含所有任务都可以使用的具体行为。接着对于每个任务（XYZ,ABC）你都会定义一个对象来存储对应的数据和细微。你会吧特定的任务对象都关联到Task功能对象上。让它们在需要的时候可以进行委托。
基本上你可以想像成，执行任务XYZ需要两个兄弟对象（XYZ和Task）写作完成。但是我们并不需要把这些行为放在一起，通过类的复制，我们可以把它们分别放在各自独立的对象中，需要时可以允许XYZ对象委托给Task。
下面是推荐的代码形式：

```js
Task = {
  setId:function(Id){this.id = Id},
  outputId:function(){console.log(this.id)}
}
//让XYZ委托Task
XYZ = Object.create(Task);
XYZ.prepareTask = function(Id,Label){
  this.setId(Id);
  this.label = Label;
}
XYZ .outputTaskDetails = function(){
  this.outputId();
  console.log(this.label);
}
```

这段代码中，Task和XYZ并不是类，它们是对象，XYZ通过Object.create()创建，它的`[[Prototype]]`委托给了Task对象。

对象关联风格的代码还有一些不同之处：
* 在上面的代码中，id和label数据成员都是直接存储在XYZ上，而不是Task。通常来说，在`[[Prototype]]`委托中最好把状态委托保存在委托者（XYZ，ABC）而不是委托目标（Task）上。
* 在类设计模式中，我们故意让父类（Task）和子类（XYZ）中都有outputTask方法，这样就可以利用重写的优势。在委托行为中恰好相反：我们会尽量避免在`[[Prototype]]`链的不同级别中使用相同的命名，否则就需要使用笨拙并且脆弱的语法来消除引用歧义。这个设计模式要求尽量少使用容易被重写的通用方法名，提倡使用更有描述性的方法名，尤其是要血清相应对象行为的类型。这样做实际上可以创建出更容易理解和维护的代码。
* this.setId(Id);XYZ 中的方法首先会寻找XYZ自身是否有setId()方法，但是XYZ中并没有这个方法，因此会通过`[[Prototype]]`委托关联到Task继续寻找，这时就可以找到setId()方法。此外，由于调用位置触发了this的隐式绑定规则，因此虽然setId方法在Task中，运行时this依然会绑定到XYZ，这正式我们想要的。换句话说，我们和XYZ进行交互时，可以使用Task中的通用方法，因为XYZ委托了Task。

**委托行为意味着某些对象在找不到属性或者方法引用时会把这个请求委托给另一个对象**

### 比较思维模型
#### （原型）面向对象风格：

```js
function Foo(who){
  this.me = who;
}
Foo.prototype.identify = function(){
  return 'I am ' + this.me;
}
function Bar(who){
  Foo.call(this,who);
}
Bar.prototype = Object.create(Foo.prototype);
Bar.prototype.speak = function(){
  console.log("hello, " + this.identify() + '.');
}

var b1 = new Bar('b1');
var b2 = new Bar('b2');
b1.speak();
b2.speak();
```

子类Bar继承了父类Foo，然后生成了b1和b2两个实例。b1委托了Bar.prototype，后者委托了Foo.prototype。这时很常见的原型式的面向对象风格代码。
类风格代码思维模型强调的是类以及实体间的关系：

![类风格思维模型图](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/class_style_oo.png)

从图中可以看出这时一张复杂的关系网。里面既有函数还有对象，而函数本身也是对象，还有各种关联关系。
我们简化一下：

![类风格思维模型图](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/class_style_oo2.png)

#### 对象关联风格

```js
Foo = {
  init:function(who){
    this.me = who;
  },
  identify:function(){
    return 'I am ' + this.me
  }
}
Bar = Object.create(Foo);
Bar.speak = function(){
  console.log('Hello,' + this.identify() + '.');
}
var b1 = Object.create(Bar);
b1.init('b1');
var b2 = Object.create(Bar);
b2.init('b2');
b1.speak();
b2.speak();
```

这段代码我们同样利用`[[Prototype]]`把b1委托给Bar,并把Bar委托给Foo，和上一段代码一模一样。我们仍然实现了三个对象之间的关联。
但非常重要的一点，这段代码简洁了许多，我们只是把对象关联起来，并不需要那些即复杂又令人困惑的模仿类的行为（构造函数，原型以及new）。
对象关联风格的思维模型：

![对象关联风格](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/object_oo.png)

通过比较可以看出，对象关联风格的代码显然更加简洁，因为这种代码只关注一件事：**对象之间的关联关系**。

在委托设计模式中，除了建议使用不相同并且更具描述性的方法名之外，还要通过对象关联避免丑陋的显示伪多态调用，代之以简单的相对委托调用。

使用类构造函数的话，你需要在同一个步骤中实现构造和初始化，然后，在许多情况下把这两步分开更灵活。

## 小结
在软件架构中你可以选择是否使用类和继承设计模式。大多数开发者理所当然的认为类是唯一（合适）的代码组织方式，但是这里我们看到了另一种更少见但是更强大的设计模式：**行为委托**。
行为委托认为对象之间是兄弟关系，互相委托，而不是父类和子类的关系。javascript的`[[Prototype]]`机制本质上就是行为委托机制。也就是说，我们可以选择在javascript中努力实现类机制，也可以拥抱更自然的`[[Prototype]]`委托机制。
当你只用对象来设计代码时，不仅可以让语法更加简洁，而且可以让代码结构更加清晰。
对象关联是一种编码风格，它倡导的是直接创建和关联对象，不把它们抽象成类。对象关联可以用基于`[[Prototype]]`的行为委托非常自然的实现。
