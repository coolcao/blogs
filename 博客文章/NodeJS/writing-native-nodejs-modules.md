---
title: 使用C/C++编写nodejs原生模块
date: 2017-01-05 20:47:52
tags: [js,nodejs]
categories:
- 技术博客
- 原创
---

一直想了解一下使用C/C++编写nodejs原生模块，从网上找到的博客，大多都停留在如何搭建环境，然后一个Hello World完事。连更多的参考资料也没有。于是就自己整理了一下，备份成此博客，分享于此。

<!--more-->

至于准备环境什么的，网上一抓一大把，就不再详述 。

主要参考两个地方：
* [nodejs官方文档](https://nodejs.org/dist/latest-v7.x/docs/api/addons.html)
* [v8文档](http://bespin.cz/~ondras/html/index.html)

其中第一个是nodejs的官方文档，里面介绍了几个不错的参考例子。
第二个是v8引擎的文档，c++的，编写c++模块主要看这个文档。

好了，我们开始几个例子，逐步的了解如何使用c++编写nodejs模块。

## Hello World
不能免俗，第一个先上来写个Hello World吧，毕竟程序员认识的第一个程序就是Hello World。

```c++
#include <node.h>
void hello(const v8::FunctionCallbackInfo<v8::Value> &args) {
    v8::Isolate *isolate = args.GetIsolate();
    auto message = v8::String::NewFromUtf8(isolate, "Hello World!");
    args.GetReturnValue().Set(message);
}
void Initialize(v8::Local<v8::Object> exports) {
    NODE_SET_METHOD(exports, "hello", hello);
}

NODE_MODULE(module_name, Initialize)
```

好了，这是最简单的一个HelloWorld，我们将文件命名为addon.cc，我们使用node-gyp编译一下，然后在我们的js文件中直接使用require引入模块，然后就可以调用了。

```js
const myAddon = require('./build/Release/addon') ;
console.log(myAddon.hello());
```

如无意外，将会在终端打印`Hello World!`。

我们简单来看一下代码，第一行`#include <node.h>`是C++中引入node.h头文件的代码。头文件可理解为接口，我们在里面只定义了接口方法，并未实现，然后通过其他文件实现，C++链接器负责将这两个链接在一起。

然后定义了一个方法`hello()`，没有返回值。方法参数通过`const v8::FunctionCallbackInfo<v8::Value> &args`传递，注意，这里我们加了`v8::`前缀注解，也可以直接在文件开始使用`using v8;`这样就可以不用每次都使用这个注解了。
`v8::Isolate *isolate = args.GetIsolate();`这里，我们在函数中访问了javascript的作用域。
`auto message = v8::String::NewFromUtf8(isolate, "Hello World!");`我们创建了一个字符串类型的变量，赋值`Hello World!`并将其绑定到作用域。
我们通过args.GetReturnValue()获取了我们函数的返回值。

`Initialize()`方法用于初始化模块方法，将方法和要导出的模块的方法名进行绑定。

最后`NODE_MODULE`导出这个模块。

上面这个例子很简单，如果是js代码的话：

```js
'use strict';

let hello = function hello() {
    let message = "Hello World!";
    return message;
};

module.exports = {
    hello:hello
};
```

好了，第一个HelloWorld就结束了。网上很多介绍nodejs C++模块的博客文章，到这里就结束了。看完之后，一脸懵逼，啥啊这是？我想再写个传参数，并对参数做简单操作的方法该怎么写？

## sum(a,b)

好吧。那我们就再写一个sum(a,b)函数，传递两个数字类型参数a,b，并求两个参数的和返回。

js中代码简单到下：

```js
let sum = function(a,b){
    if(typeof a == 'number' && typeof b == 'number'){
        return a + b;
    }else{
        throw new Error('参数类型错误');
    }
}
```

那么，C++该如何编写：

```c++
void sum(const FunctionCallbackInfo<Value> &args) {
    Isolate *isolate = args.GetIsolate();

    if(!args[0]->IsNumber()){
        isolate->ThrowException(v8::Exception::TypeError(
            v8::String::NewFromUtf8(isolate, "args[0] must be a number")));
        return ;
    }
    if(!args[1]->IsNumber()){
        isolate->ThrowException(v8::Exception::TypeError(
            v8::String::NewFromUtf8(isolate, "args[1] must be a number")));
        return ;
    }
    args.GetReturnValue().Set(args[0]->NumberValue() + args[1]->NumberValue());
}
```

首先判断两个参数是否是Number类型，如果不是，直接抛出异常。如果是，则将返回值设置为两个参数的和。

这里我们并没有在参数列表中，直接使用a,b作为参数，而是直接使用 args 对象。 这和js是类似的，第一个参数是 args[0]，第二个参数是 args[1]。

调用IsNumber()来判断是否是数字类型。如果不是，抛出一个TypeError类型错误异常。
如果类型没问题，使用`args[0]->NumberValue()`获取参数的数字值，然后相加，赋值给返回值。

可能你会问，args[0] 这是个啥？它的IsNumber()方法又是怎么来的？哪里有文档可以查阅呢？

这里其实是v8引擎内部类型，基本和js的内置对象是一一对应的。可以查阅[v8类型说明文档](http://bespin.cz/~ondras/html/classv8_1_1Value.html)。

![v8 Types](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/V8__v8__Value_Class_Reference.png)

上面这个图是不是很熟悉，和js的类型系统特别像。
js的Array,Date,Function,String等等都是继承自Object，而v8引擎内部，Object和Primitive都是继承自Value类型。

这里的IsNumber()方法就是Value类型的方法。那么除了这个方法，还有什么方法呢？

![Value方法](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/V8__v8__Value_Class_Reference1.png)

上面这张图，我只是截了一小部分，全部的可以直接去查阅文档。看，这里有各种方法，判断是否是数字类型的IsNumber()，判断是否是日期类型的IsDate()，判断是否是数组的IsArray()方法等等。

v8的接口实现的也很完善了，即使并不精通C++的开发者也可以照猫画虎的实现个简单的模块。

`args[0]->NumberValue()`返回的是一个double的值，是的，这里是实打实的C++里的double类型，可以直接进行加减运算的。类似的还有BooleanValue()方法等等，都是获取不同类型的值使用的方法。

第二个例子中，我们简单实现了一个sum()方法，传递两个参数，求和。但是这里涉及到的只是整型的值，那如果有其他类型的值怎么办呢？比如数组。

## sumOfArray(array)

下面将方法升级一下，传递一个数组，然后求数组中所有值的和。js的话：

```js
let sumOfArray = function(array){
    if(!Array.isArray(array)){
        throw new Error('参数错误，必须为Array类型');
    }
    let sum = 0;
    for(let item of array){
        sum += item;
    }

    return sum;
}
```

逻辑很简单，就是将传过来的数组进行遍历一遍，然后将所有项累加即可。C++也是如此：

```c++
void sumOfArray(const FunctionCallbackInfo<Value> &args){
    Isolate *isolate = args.GetIsolate();
    if(!args[0]->IsArray()){
        isolate->ThrowException(v8::Exception::TypeError(
            v8::String::NewFromUtf8(isolate, "args[0] must be an Array")));
        return ;
    }

    Local<Object>  received_v8_obj = args[0]->ToObject();
    Local<Array> keys = received_v8_obj->GetOwnPropertyNames();
    int length = keys->Length();
    double sum = 0;
    for(int i=0;i<length;i++){
        sum += received_v8_obj->Get(keys->Get(i))->NumberValue();
    }
    args.GetReturnValue().Set(sum);
}
```

先判断是否是数组，没什么问题。

然后我们定义了一个Object类型的received_v8_obj属性，将其赋值为`args[0]->ToObject()`。这里调用ToObject()方法将其转换为一个对象。
然后调用这个对象的GetOwnPropertyNames()方法获取所有的键，然后根据键获取对象的值，进行累加。

为什么不直接将其转换为数组，然后进行遍历呢？

我们都知道，js中的数组并不是真正的数组，其实质还是对象。其内部都是键值对存储的。因此这里也是一样，Value类型并不提供直接转换为数组的ToArray()方法，而是将其转换为Object对象，通过对象的形式进行操作。

那么对象有哪些操作呢，[看文档](http://bespin.cz/~ondras/html/classv8_1_1Object.html)。

但是你会发现，v8确实有个Array类，继承自Object类。那么Array有什么方法呢？
看文档就知道了，少的可怜：

![Array方法](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/V8__v8__Array_Class_Reference3.png)

所以，对数组的操作都将转换为对象操作。

## createObj()
说到对象了，那么我们就来写一个创建对象的方法。传递两个参数，一个name,一个age，创建一个对象，表示一个人，名叫啥，多大年纪。

```c++
void createObj(const FunctionCallbackInfo<Value> &args){
    Isolate *isolate = args.GetIsolate();
    Local<Object> obj = Object::New(isolate);
    obj->Set(String::NewFromUtf8(isolate,"name"),args[0]->ToString());
    obj->Set(String::NewFromUtf8(isolate,"age"),args[1]->ToNumber());
    args.GetReturnValue().Set(obj);
}
```

这个方法，参照文档，基本没啥可说的。

通过`Object::New(isolate)`创建一个对象，然后设置两个属性name,age，将参数依次赋值给这两个属性，然后返回这个对象即可。

如果用js写：

```js
let createObj = function(name,age){
    let obj = {};
    obj.name = name;
    obj.age = age;
    return obj;
};
```

## callback
上面说的，都没提到js中一个重要的东西，回调函数。如果参数中传一个回调函数，那么我们该如何执行呢？

来一个简单的例子。

```js
let cb = function(a,b,fn){
    if(typeof a !== 'number' || typeof b !== 'number'){
        throw new Error('参数类型错误，只能是Number类型');
    }
    if(typeof fn !== 'function'){
        throw new Error('参数fn类型错误，只能是Function类型');
    }
    fn(a,b);
};
```

这个例子很简单，我们传两个数字类型参数a,b和一个回调函数fn，然后将a,b作为fn的参数调用fn回调函数。这里我们对a,b的操作转交给回调函数。回调函数里我们可以求和，也可以求积，随你。

这个例子中，暂时还没涉及到的是如何调用回调函数。

先上代码：

```c++
void cb(const FunctionCallbackInfo<Value> &args){
    Isolate *isolate = args.GetIsolate();
    if(!args[0]->IsNumber()){
        isolate->ThrowException(v8::Exception::TypeError(
            v8::String::NewFromUtf8(isolate, "args[0] must be a Number")));
    }
    if(!args[1]->IsNumber()){
        isolate->ThrowException(v8::Exception::TypeError(
            v8::String::NewFromUtf8(isolate, "args[1] must be a Number")));
    }
    if(!args[2]->IsFunction()){
        isolate->ThrowException(v8::Exception::TypeError(
            v8::String::NewFromUtf8(isolate, "args[2] must be a Function")));
    }
    Local<Function> jsfn = Local<Function>::Cast(args[2]);

    Local<Value> argv[2] = { Number::New(isolate,args[0]->NumberValue()),Number::New(isolate,args[1]->NumberValue())};

    Local<Value> c = jsfn->Call(Null(isolate),2,argv);
    args.GetReturnValue().Set(c);

}
```

上面三个判断参数类型，略过。

我们定义一个Function类型属性jsfn，将args[2]强制转换为Function并赋值给jsfn。

然后定义一个具有两个值的参数argv，这两个值就是args[0],args[1]的数字值。

然后通过`jsfn->Call(Null(isolate),2,argv)`调用回调函数。

argv是一个数组，其个数我们在定义时指定，2个。

Call()方法为函数类型的值进行调用的方法。

```
Local< Value >  |  Call (Handle< Value > recv, int argc, Handle< Value > argv[])
```

查阅文档，可以看出，Call()方法传3个参数，第一个参数是执行上下文，用于绑定代码执行时的this，第二个参数为参数个数，第三个为参数列表，数组形式。



上面几个例子，只是冰山一角，连一角都算不上。只为了解一下nodejs使用C/C++编写原生模块，如果要编写一个可用的，高性能的C模块，那么，要求程序员一定要精通C/C++，并且对js底层也很精通，包括v8和libuv等等。

所以，简单了解一下，装装逼就好了。真要真枪实弹的上，我等渣渣就算了。至少，现在还不行，慢慢钻研吧。
