---
title: 学习笔记-你不知道的js-原型
date: 2016-11-24 22:01:50
tags: [js]
categories:
- 学习笔记
- 你不知道的js
---

## 原型
### 原型对象
无论什么时候，只要创建了一个新函数，就会根据一组特定的规则为该函数创建一个prototype属性，这个属性指向函数的原型对象。
![原型](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/js_prototype.png)
Person为构造函数，其有一个属性prototype，指向其原型对象。
Person Prototype为Person的原型对象，其必有一个属性constructor，指向Person
person1,person2为两个实例对象，都有一个属性`[[Prototype]]`，指向其构造函数的原型对象。

> 注意：实例对象没有constructor属性，如果直接person1.constructor，则返回的是其原型对象上的contructor指向，即Person构造函数。

## `[[Prototype]]`
Javascript中的对象有一个特殊的`[[Prototype]]`属性，其实就是对于其他对象的引用。几乎所有的对象在创建时`[[Prototype]]`属性都会被赋予一个非空的值。

<!--more-->

```js
var obj = {
  name:'coolcao'
}
var obj1 = Object.create(obj);
console.log(obj1.name); //coolcao
```

> Object.create()会创建一个对象，并把这个对象的`[[Prototype]]`关联到指定对象。

### Object.prototype
所有普通的`[[Prototype]]`链最终都会指向内置的Object.prototype。由于所有的普通对象都“源自”这个Object.prototype，所以它包含JavaScript中许多的功能。

### 属性设置与屏蔽
给对象属性赋值`obj.name = 'coolcao'`到底发生了什么？
如果obj对象中包含名为name的普通数据访问属性，这条赋值语句只会修改已有的属性值。
如果name不是直接存在与obj中，`[[Prototype]]`链就会被遍历，如果原型链上找不到foo，foo就会被直接添加到obj上。
如果原型链上找到了name属性，那么会出现三种情况：
1. 如果在`[[Prototype]]`链上层存在名为name的普通数据访问属性，并且没有被标记为只读（writable:false）,那就会直接在obj中添加一个名为name的新属性，它是屏蔽属性。
2. 如果在`[[Prototype]]`链上存在name，但是它被标记为只读（writable:true），那么无法修改已有属性或者在obj上创建屏蔽属性。如果运行在严格模式下，代码会抛出一个错误，否则这条赋值语句会被忽略。总之不会发生属性屏蔽。
3. 如果在`[[Prototype]]`链上存在一个名为name的setter，那就一定会调用这个setter。name不会被添加到obj，也不会重新定义name这个setter。

**屏蔽**:如果属性名name既出现在obj中也出现在obj的原型链上，那么就会发生屏蔽。因为obj.name总是会选择原型链中最底层的属性。

```js
//如果原型链上已存在名为name的属性，没有被标记为只读
var proto = {
  name:'coolcao'
}
var obj = Object.create(proto);

obj.name = 'lili';

console.log(proto);   //{ name: 'coolcao' }
console.log(obj);     //{ name: 'lili' } 发生了属性屏蔽


//在原型链上存在name ,且被标记为只读
Object.defineProperty(proto,'name',{
  writeable:false
});
obj.name = 'lucy';  //严格模式下报错，TypeError: Cannot assign to read only property 'name' of object '#<Object>'
console.log(proto); //非严格模式下，输出 { name: 'coolcao' }
console.log(obj);   //非严格模式下，输出 {}

//在原型链上存在名为name的setter
var proto = {
  _name:'coolcao',
  set name(n){
    this._name = n;
  },
  get name(){
    return this._name;
  }
}
var obj = Object.create(proto);
obj.name = 'lili';
console.log(proto);     //{ _name: 'coolcao', name: [Getter/Setter] }
console.log(obj);       //{ _name: 'lili' }
console.log(proto.name);//coolcao
console.log(obj.name);  //lili
```

有些情况下，会发生隐式屏蔽，一定要当心：

```js
var proto = {
  name:'coolcao',
  age:23
}
var obj = Object.create(proto);
obj.age ++;         //隐式屏蔽
console.log(proto); //{ name: 'coolcao', age: 23 }
console.log(obj);   //{ age: 24 }
```

## 构造函数
### 构造函数还是调用
```js
var SayHello = function(){
  console.log('hello');
}
var obj = new SayHello();
console.log(obj); //SayHello {}
```
SayHello本身是一个函数，但使用new关键字调用后会创建一个对象，并赋值给obj，但此时obj为空对象。

```js
var SayHello = function SayHello(){
  return function(){
    console.log('hello');
  }
}
var obj = new SayHello();
console.log(obj); //[Function]
obj();    //hello
```
同样，这里SayHello还是一个函数，只是使用return返回了一个函数，当使用new调用时，这时返回的是一个函数（js里函数也是对象嘛），我们继续调用obj()实际上是调用SayHello内部返回的函数。

## (原型)继承
```js
function Foo(name){
  this.name = name;
}
Foo.prototype.myName = function(){
  return this.name;
}
function Bar(name,label){
  Foo.call(this,name);
  this.label = label;
}
Bar.prototype = Object.create(Foo.prototype);
Bar.prototype.myLabel = function(){
  return this.label;
}
var a = new Bar('a','obj a');
a.myName(); //a
a.myLabel();  //'obj a'
```

上面代码产生的对象原型示意图：

![原型示意图](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/foo_bar_prototype.png)

其中Foo和Bar为两个构造函数，Foo Prototype为Foo.prototype对象，Bar Prototype为Bar.prototype对象。

注意：下面两种方式是常见的错误：

```js
//和你想要的不一样
Bar.property = Foo.prototype;

//基本上满足你的要求，但是可能会产生一些副作用
Bar.prototype = new Foo();
```
`Bar.property = Foo.property`并不会创建一个关联到Bar.prototype的新对象，它只是让Bar.property直接引用Foo.prototype对象。因此，再定义 Bar.property.myLabel时，相当于影响了 Foo.prototype对象，这不是我们想要的。
`Bar.property = new Foo()`的确会创建一个关联到Bar.prototype的新对象。但是它使用了Foo()的构造函数调用，如果函数Foo有一些副作用（比如写日志，修改状态，注册到其他对象，给this添加数据属性等）的话，就会影响到Bar()的“后代”。

因此，要创建一个适合的关联对象，必须使用`Object.create()`，而不是使用构造函数`Foo()`。这样做的缺点是需要创建一个新的对象，然后把旧对象抛弃掉，不能直接修改已有的默认对象。

> 这句话里的新对象即是Object.create()创建的对象，而旧对象则是指Bar函数的默认原型对象`Bar.prototype`对象，重新赋值后，相当于将默认的Bar.prototype抛弃掉了，从性能上说，会有轻微的性能损失。

ES6添加了函数`Object.setPrototypeOf()`，可以用标准并且可靠的方法来修改关联。

```js
//ES6之前要抛弃默认的Bar.prototype
Bar.prototype = Object.create(Foo.prototype);
//ES6可以直接修改现有的Bar.property
Object.setPrototypeOf(Bar.property,Foo.prototype);
```
如果忽略掉Object.create()方法带来的轻微性能损失（抛弃的对象需要进行垃圾回收），它实际上比ES6及其之后的方法更短且可读性更高。

### 检查“类”关系
```js
a instanceof Bar;   //true
a instanceof Foo;   //true
```
instanceof 操作符的左边是一个普通的对象，右边是一个函数。instanceof回答的是：**在a的整条`[[Prototype]]`链中是否存在指向Foo.prototype的对象呢**。
如果你想判断两个对象（比如a和Foo.prototype）之间是否通过`[[Prototype]]`链关联，无法用instanceof实现。
可以使用第二种方法：`isPrototypeOf()`

```js
Bar.prototype.isPrototypeOf(a); //true
Foo.prototype.isPrototypeOf(a); //true
```
同样的问题，同样的答案，但是第二种方法中并不需要间接引用函数Foo，它的.prototype属性会被自动访问。
我们也可以直接获取一个对象的`[[Prototype]]`：`Object.getPrototypeOf(a)`.按照上面的例子话，`Object.getPrototypeOf(a) == Bar.prototype;Object.getPrototypeOf(Bar.prototype) == Foo.prototype`。
*注意，获取对象的原型，直接使用ES5标准的 Object.getPrototypeOf()方法，而不要使用obj.__proto__的形式，因为这种形式只是浏览器中自行定义的，并非标准。*

## 对象关联
### 创建对象关联
Object.create()会创建一个新对象，并把它关联到我们指定的对象。
> Object.create(null)会创建一个拥有空`[[Prototype]]`链接的对象，由于没有原型链，所以instanceof操作符无法进行判断，总是返回false。这些特殊的空`[[Prototype]]`对象通常被称作“字典”，他们完全不会受到原型链的干扰，因此非常适合用来存储数据。

## 解惑 instanceof,isPrototypeOf,getPrototypeOf

到现在为止，我们了解了原型及原型链，那么，对于`instanceof`,`isPrototypeOf()`，`Object.getPrototypeOf()`之间的区别，你都熟悉了么？
我们现在就做实验，揭开它们之间的关系，理清它们剪不断，理还乱的关系。

```js
var Animal = function (name) {
    this.name = name;
    this.sayHi = function () {
        console.log('hello,I\'m ' + this.name + ',and I\'m a ' + this.type);
    }
}

var Dog = function (name) {
    Animal.call(this,name);
    this.type = 'dog';
}

Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;

var dog = new Dog('biubiu');
```

对象原型关系图：

![对象原型关系图](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/instanceof_isprototypeof_getprototype.png)

dog的原型链上依次是:Dog.prototype -> Animal.prototype

这里我们定义了一个`Animal`的“基类”，然后定义了一个`Dog`“类”，然后通过原型链实现了所谓的“继承”。

### instanceof

下面思考如下的代码输出：

```js
console.log(dog instanceof Animal);   //true
console.log(dog instanceof Dog);      //true
```

上面代码都输出了`true`，大家可能会说，这本来就是啊，有什么疑问么？可是，为什么呢？
是因为*dog是由Dog类创建的一个实例，所以`dog instanceof Dog`肯定是true，而Dog类继承了Animal类，因此`dog instanceof Animal`当然也是true。*
可能从类式面向对象语言的角度去分析，也没什么错对吧，但是，别忘了，javascript根本就不存在类，所谓的“继承”也是由原型链实现的。上面的解释，只能说是，*碰巧对了*。
为什么呢？
如果我们在`var dog = new Dog('biubiu');`这个语句后面，添加这么一句`Dog.prototype = {};`，也就是说，我们在实例化完dog对象后，立即改变Dog的原型对象，那么，`dog instanceof Dog`的结果是什么呢？
毫无疑问，结果是`false`。所以，从这里可以看出，我们不能简单的将类式面向对象语言的角度来看待javascript中的对象及其原型。
那么，`instanceof`到底怎么理解？
当我们将`Dog.prototype = {}`时，从原型链图上我们可以看出，Dog.prototype就不在dog对象的原型链上了，所以返回的是false。而Animal.prototype还在dog对象的原型链上，因此，`dog instanceof Animal`还是会返回ture。

因此，instanceof 可以理解为：**在dog的原型链中，是否存在指向Dog.prototype的对象**，左操作数是一个对象，右操作数是一个函数。

### isPrototypeOf

```js
console.log(Dog.prototype.isPrototypeOf(dog));  //true
console.log(Animal.prototype.isPrototypeOf(dog));   //true
```

很明显了，isPrototypeOf 用于判断一个对象是否在另一个对象的原型链上。注意，这里是**对象与对象之间的关系**。

当我们执行 `Dog.prototype = {}`时，自然，`Dog.prototype.isPrototypeOf(dog)`也是false了。

### Object.getPrototypeOf()

```js
console.log(Object.getPrototypeOf(dog) === Animal.prototype);  //false
console.log(Object.getPrototypeOf(dog) === Dog.prototype);  //true
```

Object.getPrototypeOf()用于获取当前对象的原型对象，而不是原型链上所有对象，请注意。
