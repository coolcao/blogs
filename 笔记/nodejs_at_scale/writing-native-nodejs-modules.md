---
title: 【译】【6】编写原生模块
date: 2016-12-27 20:35:34
tags: [js,nodejs]
categories:
- 技术博客
- 翻译
- Nodejs At Scale
---

有些时候，javascript的性能可能会显得有些不足，因此你需要更多的依赖原生的模块。

<!--more-->

> 原生模块的拓展并不是一个入门者的主题，我强烈建议阅读本文章的任何一个Nodejs的开发者，有一点它们是如何工作的基础知识。

## 原生Nodejs模块的一般使用场景
原生模块的知识是简单的，当你添加一个原生拓展作为依赖时，那么你已经做到了。

看一下列表中流行的使用原生模块的拓展。你至少使用过它们中的一个，不是吗？

* [https://github.com/wadey/node-microtime](https://github.com/wadey/node-microtime)
* [https://github.com/node-inspector](https://github.com/node-inspector)
* [https://github.com/node-inspector/v8-profiler](https://github.com/node-inspector/v8-profiler)
* [http://www.nodegit.org/](http://www.nodegit.org/)

为什么需要考虑编写原生nodejs模块，有几个原因，包括但不限于：

* 对性能有苛刻要求的应用：实在讲，Nodejs善于处理异步I/O操作，但实数运算时，它不是一个好的选择。
* 挂钩底层（例如操作系统）API时
* 在C或C++和nodejs之间创建桥梁时

## 原生模块是什么

> Nodejs插件是动态链接的共享对象，使用C或C++编写，可以使用require()函数加载到Nodejs，就像它们是普通的Nodejs模块一样。
>                   出自[Nodejs文档](https://nodejs.org/en/docs/)

这意味着（如果做的对的话）那些晦涩的C/C++模块对于使用者是不可见的。他们看到的是的模块就是nodejs模块，就像你使用javascript编写的模块一样。

从之前的博客中我们已经学习到，Nodejs是运行在v8 javascript引擎上的，v8引擎本身是一个C程序。我们可以使用C语言编写直接与这个C程序（v8引擎）交互的代码，这是极棒的，因为我们可以避免很多昂贵的序列化和通信开销。

而且，在之前的博客中我们也学习到Nodejs垃圾回收器的开销。尽管如果你决定你自己管理内存，垃圾回收是可以避免的，因为C/C++没有垃圾回收器，你会更容易的创建内存问题。

编写原生拓展需要下面的一个或多个主题的知识：

* [libuv](http://docs.libuv.org/en/v1.x/)
* [V8](https://github.com/v8/v8/wiki)
* [Nodejs internals](著://nodejs.org/api/)

它们都有极为优秀的文档，如果你想深入了解某个领域，我建议你去阅读它们的文档。

好，让我们开始接下来的学习：

### 提前准备
#### Linux
* python (建议v2.7,v3.x.x不支持)
* make
* C/C++编译工具，比如GCC

#### Mac
* 安装Xcode:确保你不仅仅是安装了，而且至少启动过一次，并且接受它的条款和条件，否则它将无法工作。

#### Windows
* 以管理员身份运行cmd.exe,键入命令`npm install --global --production windows-build-tools`，这样你就安装了所有需要的东西。
* 或者，安装Visual Studio(它预置有所有的C/C++构建工具)
* 或者，使用最新版本的Windows提供的Linux子系统。然后按照上面的Linux指令。

## 创建原生Nodejs拓展
让我们创建原生拓展的第一个文件。我们既可以使用`.cc`拓展名，表示带类的C语言C，也可以使用`.cpp`做拓展名。[Google Style Guide](https://google.github.io/styleguide/cppguide.html)建议使用`.cc`，那么，我们就使用它。

首先，让我们看一下整个文件，然后，我将为你逐行解释它：

```c
#include <node.h>

const int maxValue = 10;
int numberOfCalls = 0;

void WhoAmI(const v8::FunctionCallbackInfo<v8::Value>& args) {
  v8::Isolate* isolate = args.GetIsolate();
  auto message = v8::String::NewFromUtf8(isolate, "I'm a Node Hero!");
  args.GetReturnValue().Set(message);
}

void Increment(const v8::FunctionCallbackInfo<v8::Value>& args) {
  v8::Isolate* isolate = args.GetIsolate();

  if (!args[0]->IsNumber()) {
    isolate->ThrowException(v8::Exception::TypeError(
          v8::String::NewFromUtf8(isolate, "Argument must be a number")));
    return;
  }

  double argsValue = args[0]->NumberValue();
  if (numberOfCalls + argsValue > maxValue) {
    isolate->ThrowException(v8::Exception::Error(
          v8::String::NewFromUtf8(isolate, "Counter went through the roof!")));
    return;
  }

  numberOfCalls += argsValue;

  auto currentNumberOfCalls =
    v8::Number::New(isolate, static_cast<double>(numberOfCalls));

  args.GetReturnValue().Set(currentNumberOfCalls);
}

void Initialize(v8::Local<v8::Object> exports) {
  NODE_SET_METHOD(exports, "whoami", WhoAmI);
  NODE_SET_METHOD(exports, "increment", Increment);
}

NODE_MODULE(module_name, Initialize)

```

好了，现在让我们逐行看看它。

```c
#include <node.h>
```

C++中的`include`类似于javascript中的`require()`。它将会从给定的文件中拉取任何东西。但是，区别与直接链接到源代码，在C++中，我们有头文件的概念。

我们可以在头文件中声明确切的接口，而不实现它，然后我们通过头文件包含实现。C++链接器负责将这两个链接在一起。将其视为一个描述其内容的文档文件，可以在代码中复用。

```c
void WhoAmI(const v8::FunctionCallbackInfo<v8::Value>& args) {
  v8::Isolate* isolate = args.GetIsolate();
  auto message = v8::String::NewFromUtf8(isolate, "I'm a Node Hero!");
  args.GetReturnValue().Set(message);
}
```

由于这是一个原生拓展，v8命名空间是可用的。注意，`v8::`注解，它被用于访问v8接口。如果你在使用任何v8提供的类型之前不想包含`v8::`，你可以添加`using v8`到文件的顶部。然后你就可以删除所有的`v8::`命名空间指示符来指定类型，但这样可能导致代码中的命名冲突，因此使用它们的时候要小心啦。为了100%明示，在我的代码中，我将使用`v8::`注解使用所有的v8类型。

在我们的样例代码中，我们访问了调用函数（从javascript调用）的参数，通过`args`对象，该对象也提供了我们所有的调用相关的信息。

使用`v8::Isolate*`，我们的函数中访问了当前的javascript作用域。这里的作用域就像javascirpt中类似，我们可以给变量赋值，并且将它们绑定到特定代码的生命周期上。我们不必担心释放这些内存块，因为像在javascript分配它们一样，垃圾回收器会自动处理它们。

```js
function () {
  var a = 1;
} // SCOPE
```

我们通过`args.GetReturnValue()`获取了我们函数的返回值。只要我们愿意，我们可以将它设置为'v8::'命名空间中的任何值。

C++为存储整数和字符串有内建类型，但是javascript只能理解它的`v8::`类型对象。只要我们在C++作用域世界，我们可以免费使用这些C++内建的类型，但是当我们处理javascript对象并且和javascript代码交互时，我们必须将C++类型转换为javascript上下文能理解的类型。这些是v8::命名空间暴露出来的类型，比如`v8::String`或`v8::Object`。

```c
void WhoAmI(const v8::FunctionCallbackInfo<v8::Value>& args) {
  v8::Isolate* isolate = args.GetIsolate();
  auto message = v8::String::NewFromUtf8(isolate, "I'm a Node Hero!");
  args.GetReturnValue().Set(message);
}
```

让我们看一下我们的第二个方法，它提供一个参数，累加了计数器，直到计数器到达10。

这个函数还接受一个来自javascript的参数。当你从javascript接受参数时，你必须小心了，因为它们是松散类型的对象。（你可能已经习惯了在javascript中使用）

arguments数组包含v8::Objects，所以它们都是Javascript对象，但是要小心这些，因为在上下文中，我们永远不能确定它们可能包含什么。我们必须显示检查这些对象的类型。幸运的是，在这些类中添加了辅助方法。以在类型转换之前确定它们的类型。

为了保持与现有Javascript代码的兼容性，如果参数类型错误，我们必须抛出一些错误。要抛出类型错误，我们必须使用`v8::Exception::TypeError()`构造器创建一个Error对象。如果第一个参数不是数字，下面的块将抛出TypeError。

```c
if (!args[0]->IsNumber()) {
  isolate->ThrowException(v8::Exception::TypeError(
        v8::String::NewFromUtf8(isolate, "Argument must be a number")));
  return;
}
```

在javascript中，类似的代码片段：

```js
If (typeof arguments[0] !== ‘number’) {
  throw new TypeError(‘Argument must be a number’)
}
```

我们也必须处理，如果计数器越界。我们可以创建一个自动以的异常，就像我们在javascript中所做的：`new Error(error message)`。在使用C++的v8的api中类似于：`v8::Exception:Error(v8::String::NewFromUtf8(isolate, "Counter went through the roof!")));`，其中，isolate是当前的作用域，我们必须首先使用`v8::Isolate* isolate = args.GetIsolate();`来获取它的引用。

```c
double argsValue = args[0]->NumberValue();
if (numberOfCalls + argsValue > maxValue) {
  isolate->ThrowException(v8::Exception::Error(
        v8::String::NewFromUtf8(isolate, "Counter went through the roof!")));
  return;
 }
```

当我们处理了所有可能出现的错误，我们就将参数添加到我们C++作用域中可用的counter变量中。这看起来就像是javascript代码。为了将这个新的值返回给javascript代码，我们首先必须将它从C++中的`integer`类型转换为`v8::Number`类型，这样我们就可以在javascript代码中访问它了。首先我们必须使用`static_cast<double>()`将我们的integer转换为double类型，然后我们将结果传递给`v8::Number`构造器。

```c
auto currentNumberOfCalls =
  v8::Number::New(isolate, static_cast<double>(numberOfCalls));
```

`NODE_SET_METHOD`是一个宏，我们可用来在导出的对象上分配一个方法。这和我们在javascript中导出的对象一样。这相当于：

```js
exports.whoami = WhoAmI
```

事实上，所有的nodejs插件都必须按照以下的模式导出初始化函数：

```c
void Initialize(v8::Local<v8::Object> exports);
NODE_MODULE(module_name, Initialize)
```

所有的C==模块必须将它们注册到nodejs的模块系统。如果没有这些代码，你在javascript中将无法访问它们。如果你意外的忘记注册你的模块，它们仍将被编译，但是当你试图在javascript代码中访问它时，你将会抛出如下异常：

```
module.js:597
  return process.dlopen(module, path._makeLong(filename));
                 ^

Error: Module did not self-register.
```

到现在为止，如果你看到这样的错误，你就知道该怎么办了。

## 编译模块
现在，我们有一个C++的nodejs模块了，让我们编译一下它。我们用到的编译器被称为 `node-gyp`，它可以使用npm来获取。我们所需要做的就是添加一个`binding.gyp`文件，它看起来如下：

```js
{
  "targets": [
    {
      "target_name": "addon",
      "sources": [ "example.cc" ]
    }
  ]
}
```

`npm install`将会处理剩下的事情。你也可以通过全局安装node-gyp来使用它，使用`npm install node-gyp -g`将它全局安装到你的系统。

现在，我们就已经准备好了C++部分了，剩下的就是让他在nodejs代码中工作。借助于node-gyp编译器，我们可以无缝的调用这些插件。它通过`require`调用：

```js
const myAddon = require('./build/Release/addon')
console.log(myAddon.whoami())
```

这可以工作，但是每次指定路径有一点繁琐，我们都知道，相对路径很难使用。有一个模块可以帮助我们处理这个问题。

`bindings`模块被构建为满足我们更少工作的需求。首先，让我们使用`npm install bindings --save`来安装`bindings`模块，然后在我们的代码片段中做一个晓得调整。我们引入bindings模块，它将会暴露出我们在`binding.gyp`文件中通过`target_name`指定的所有的`.node`原生模块。

```js
const myAddon = require('bindings')('addon')
console.log(myAddon.whoami())
```

这两种方式是等价的。

这就是创建原生绑定到nodejs并桥接到javascript代码方式。但是有一个小问题：Nodejs是不断发展的，而接口也有不兼容的趋势。这意味着，指定特定版本可能不是一个好主意，因为你的插件可能很快将过期。

**Nodejs的本地抽象(NaN)**

NaN库开始是由独立的个人编写的第三方模块，但是从2015年末，它变为nodejs的一个孵化项目。

NaN为我们提供了一个在Nodejs API之上的抽象层，并在所有版本上创建了一个通用接口。使用NaN而不是本机nodejs接口被认为是一个最佳实践，所以你可以始终保持领先。

为了使用NaN，我们必须重写我们应用的部分，但是首先，让我们先安装它`npm install nan --save`。首先，首先我们必须将以下一行添加到`ndings.gyp`文件的targets字段。这样就确保引用NaN头文件到我们的程序，使用NaN函数。

```js
{
  "targets": [
    {
      "include_dirs" : [
        "<!(node -e \"require('nan')\")"
      ],
      "target_name": "addon",
      "sources": [ "example.cc" ]
    }
  ]
}
```

在我们的例子中，可以使用NaN的抽象来替换一些v8的类型。它提供我们一些辅助方法调用参数，使得使用v8类型有更好的体验。

你可能注意的第一件事是，我们不必通过`v8::Isolate* isolate = args.GetIsolate()`明确的访问javascript的作用域。NaN为我们自动的处理了。它的类型将隐式的绑定到当前的作用域，因此我们不必费心使用它们。

```c
#include <nan.h>

const int maxValue = 10;
int numberOfCalls = 0;

void WhoAmI(const Nan::FunctionCallbackInfo<v8::Value>& args) {
  auto message = Nan::New<v8::String>("I'm a Node Hero!").ToLocalChecked();
  args.GetReturnValue().Set(message);
}

void Increment(const Nan::FunctionCallbackInfo<v8::Value>& args) {
  if (!args[0]->IsNumber()) {
    Nan::ThrowError("Argument must be a number");
    return;
  }

  double argsValue = args[0]->NumberValue();
  if (numberOfCalls + argsValue > maxValue) {
    Nan::ThrowError("Counter went through the roof!");
    return;
  }

  numberOfCalls += argsValue;

  auto currentNumberOfCalls =
    Nan::New<v8::Number>(numberOfCalls);

  args.GetReturnValue().Set(currentNumberOfCalls);
}

void Initialize(v8::Local<v8::Object> exports) {
  exports->Set(Nan::New("whoami").ToLocalChecked(),
      Nan::New<v8::FunctionTemplate>(WhoAmI)->GetFunction());
  exports->Set(Nan::New("increment").ToLocalChecked(),
      Nan::New<v8::FunctionTemplate>(Increment)->GetFunction());
}

NODE_MODULE(addon, Initialize)
```

现在我们有了一个可以工作的，也是常用的例子，说明一个nodejs原生拓展应该是什么样子。

首先，我们学习了如何结构化代码，然后是编译过程，然后逐行了解代码。最后，我们研究了NaN提供的v8 API的抽象。

我们还做了一个小调整，这是使用NaN提供的宏。

宏是编译器在编译代码将展开的代码片段。有关宏的更多信息，请[参见本文档](http://en.cppreference.com/w/cpp/preprocessor/replace)。我们已经使用了这些宏的之一：`NODE_MODULE`，但是NaN一些其他的我们可以引用的宏。当我们创建原生拓展时，这些宏可以为我们节省不少时间。

```c
#include <nan.h>

const int maxValue = 10;
int numberOfCalls = 0;

NAN_METHOD(WhoAmI) {
  auto message = Nan::New<v8::String>("I'm a Node Hero!").ToLocalChecked();
  info.GetReturnValue().Set(message);
}

NAN_METHOD(Increment) {
  if (!info[0]->IsNumber()) {
    Nan::ThrowError("Argument must be a number");
    return;
  }

  double infoValue = info[0]->NumberValue();
  if (numberOfCalls + infoValue > maxValue) {
    Nan::ThrowError("Counter went through the roof!");
    return;
  }

  numberOfCalls += infoValue;

  auto currentNumberOfCalls =
    Nan::New<v8::Number>(numberOfCalls);

  info.GetReturnValue().Set(currentNumberOfCalls);
}

NAN_MODULE_INIT(Initialize) {
  NAN_EXPORT(target, WhoAmI);
  NAN_EXPORT(target, Increment);
}

NODE_MODULE(addon, Initialize)
```

第一个NaN_METHOD将节省我们输入长方法名的负担，并且也会为我们节省编译器展开此宏的时间。注意，如果你使用宏，你必须使用宏本身提供的命名，所以，参数对象`info`将被调用，而不是`args`，我们必须在每个地方都改过来。

我们使用的下一个宏`NAN_MODULE_INIT`，提供了初始化函数，它将它的参数命名为`target`，而不是exports，同样，我们也必须改过来这个地方。

我们使用的最后一个宏`NAN_EXPORT`，设置了我们模块的接口。你可以看到，在这个宏中，我们不能指定对象的键，它将会使用各自的名称给它们赋值。

这就像是在javascript中的：

```js
module.exports = {
  Increment,
  WhoAmI
}
```

如果你想在我们之前的例子中这样使用，确认你将函数名改为小写，像这样：

```js
'use strict'

const addon = require('./build/Release/addon.node')

console.log(`native addon whoami: ${addon.WhoAmI()}`)

for (let i = 0; i < 6; i++) {
  console.log(`native addon increment: ${addon.Increment(i)}`)
}
```

更多文档，请参考 [NaN的github页面](https://github.com/nodejs/nan)。

**样例代码仓库**

我创建了一个仓库，包含了本博客所有的代码。代码使用git管理，可以在[github上访问](https://github.com/peteyy/native-addon-example)。**每一步都有它们自己的分支**，master分支是第一个例子，nan分支是第二个，最后一步的分支叫做macros。

## 小结

我希望你和我一样有乐趣，因为我已经写了这个话题。我不是一个C/C++专家，但是我已经做nodejs有足够的时间，有兴趣使用一个名为C的伟大的语言编写和实验我自己的快速原生插件。

我强烈建议至少了解一点C/C++，以理解平台本身底层的东西。你一定会找到你感兴趣的东西。

正如你看到的，它不想第一眼看起来那么可怕，所以继续使用C++构建一些东西，如果你需要我们的帮助，请在tweet上[@risingstack](https://twitter.com/risingstack)，或者在下面留言。

在《Node.js at Scales》系列的下一章节中，我们将学习《深入Nodejs项目结构》。
