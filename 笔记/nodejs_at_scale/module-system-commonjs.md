---
title: 【译】【3】模块系统
date: 2016-12-18 11:02:05
tags: [nodejs]
categories:
- 技术博客
- 翻译
- Nodejs At Scale
---

本章为《Node.js At Scale》系列的第三章，本章中你将学习到nodejs的模块系统，CommonJS以及`require`是如何工作的。
在ES2015标准之前，JavaScript本身并没有一种组织代码的方式。Node.js使用[CommonJS](http://requirejs.org/docs/commonjs.html)的格式填补了这个空白。在本文中，我们将学习Nodejs的模块系统是如何工作的，你将如何组织代码，以及新的ES标准对Nodejs的未来意味着什么。

<!--more-->


### 什么是模块系统
模块是代码结构的基本构建块。模块系统允许你组织代码，隐藏信息，并且只用module.exports暴露组件的公共接口。
每次调用require时，就会加载另一个模块。
下面是使用CommonJS的一个最简单的示例：

```js
// add.js
function add (a, b) {
  return a + b
}

module.exports = add
```

要使用add.js，我们必须引入它：

```js
// index.js
const add = require('./add')

console.log(add(4, 5))
//9
```

表象之下，Nodejs将使用这种方式将add.js包裹进来：

```js
(function (exports, require, module, __filename, __dirname) {
  function add (a, b) {
    return a + b
  }

  module.exports = add
})
```

这就是为什么你可以访问全局变量，比如require和module。它还确保你的变量的作用域在你模块内而不是全局对象上。

### require是如何工作的
Nodejs的模块加载机制是在第一次调用`require`时将模块进行缓存。这意味着你每次调用`require('awesome-module')`时，你将会得到`awesome-module`的相同实例，这样就确保了模块是单例的，在你的整个应用中拥有相同的状态。

你可以从文件系统或已安装的模块加载本地模块和路径引用。如果传递给require函数的标识符不是本地模块或文件引用（以 /,../,./或者类似开头），那么nodejs将会查找已安装的模块。它将会遍历你的文件系统，在node_modules文件夹中查找引用的模块。它从当前模块的父目录开始，然后移动到父目录，直到找到正确的模块位置或这直到到达文件系统的根目录。

#### 表象下的require：module.js
在node的核心中处理模块加载的模块被称作`module.js`，可以在node仓库的[lib/module.js](https://github.com/nodejs/node/blob/master/lib/module.js)找到。
这里要检查的最重要的函数是`_load()`和`_compile()`函数。

##### Module._load
这个函数会检查模块在缓存中是否已存在，如果存在，它将直接返回导出的对象。
如果模块在本地，它将会以文件名为参数，调用`NativeModule.require()`函数，然后返回结果。
否则它将会它将会为这个文件创建一个新的模块，然后将其保存到缓存。在返回它的导出对象之前加载完文件内容。

##### Module._compile
compile函数在正确的作用域或沙箱中运行文件弄荣，以暴露帮助变量，例如require,module,或exports。
![How Require Works](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/node-js-at-scale-how-require-works.png)
require是如何工作的，来自[ James N. Snell](https://hackernoon.com/node-js-tc-39-and-modules-a1118aecf95e#.z1plueqbn)

### 如何组织代码
在我们的应用中，当我们创建模块时，要在内聚和耦合间找到一种平衡。期望的情况是，实现模块的高内聚，低耦合。
模块必须只关注功能的单个部分，以具有高内聚力。低耦合意味着模块不应该具有全局或共享状态。它们应该只通过传递参数进行通信，并且它们可以轻松替换，而无序触及更广泛的代码库。
我们通常以以下方式导出命名函数或常量：

```js
'use strict'

const CONNECTION_LIMIT = 0

function connect () { /* ... */ }

module.exports = {
  CONNECTION_LIMIT,
  connect
}
```

### node_modules里有什么
`node_modules`目录是nodejs查找模块的地方。`npm v2`和`npm v3`安装依赖时会有不同。你可以运行以下命令查看你当前的npm版本：

```js
npm --version
```

#### npm v2
npm 2 安装所有依赖通过一个嵌套的方式，在你的主包依赖在他们的node_modules目录。
#### npm v3
npm 3 试图把这些次要的依赖扁平化，安装在根node_modules目录。这意味着通过查看你的mode_modules，你不会知道哪个是你的显式依赖，哪个是隐式依赖。
你可以确定的是，通过同一个package.json文件安装的包，你的node_modules目录都是一样的。这种情况下，它将按字母顺序安装你的依赖，这也意味着你将得到相同的目录树。这是非常重要的，因为模块在缓存时使用它们的路径作为查找键。每个包都有自己的子node_modules目录，这可能导致一个相同的包或模块会有多个实例。

### 如何处理你的模块
装载模块有两种主要的方式。其中一种是使用硬编码依赖，使用require调用明确加载一个模块到另一个模块。另一种方式是使用依赖注入模式，我们通过组件作为参数，或者我们有一个全局容器（被称为IoC,控制反转容器），它集中了各模块的管理。

我们通过使用硬编码方式加载模块，来允许nodejs管理模块的生命周期。它以直观的方式组织你的包，这使得理解和调试变得容易。

依赖注入在Node.js环境中很少使用，但它是一个很有用的概念。依赖注入模式可以导致一个改进的去耦模块。与其明确定义模块的依赖关系，不如从外部接收它们。因此它们很容易得更换具有相同接口的模块。

让我们看一个使用工厂模式的依赖注入模块的例子：

```js
class Car {
  constructor (options) {
    this.engine = options.engine
  }

  start () {
    this.engine.start()
  }
}

function create (options) {
  return new Car(options)
}

module.exports = create
```

### ES2015模块系统
正如我们看到的，CommonJS模块系统使用运行时评估模块，在执行之前将它们包装成一个函数。ES2015模块系统不需要包裹直到 `import/export`绑定创建模块。这种不兼容性是目前没有Javascript运行时支持ES模块的原因。很多关于这个话题的讨论，一个[提案](https://github.com/nodejs/node-eps/blob/master/002-es6-modules.md)在草案状态，所以希望我们在未来的node版本有它的支持。

想要更深入了解CommonJS和ESM之前的区别，可以阅读 [James M Snell的文章](https://hackernoon.com/node-js-tc-39-and-modules-a1118aecf95e#.z1plueqbn)
