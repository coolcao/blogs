---
title: 【译】【1】npm最佳实践
date: 2016-12-17 00:42:21
tags: [nodejs]
categories:
- 技术博客
- 翻译
- Nodejs At Scale
---

我们创建了一个新的系列，叫做《Node.js At Scale》，我们正在编写一个文章集合，专注于安装大型Nodejs项目的公司以及已经学习过基础nodejs的开发者。

在《Node.js At Scale》系列的第一章中，你将会学习到使用npm的最佳实践，这些提示和技巧可以在你的日常开发中节约不少时间。

<!--more-->

`npm install` 是最常用的npm命令行工具，但是它提供了更多的功能。在本章中，你将会学习到，npm在你的应用的整个周期中是如何帮助你的，从启动一个新的工程到开发，一直到部署。

## 认识npm
在进入主题前，先让我看一下一些帮助你检查你所运行npm版本的命令：

### npm versions
要想获取你当前正在运行的npm版本，你可以运行以下命令：

```shell
npm --version
2.13.2
```

npm可以返回的更多，而不仅仅是它的版本。它可以返回当前包的版本，你正在使用的nodejs的版本，以及OpenSSL和V8的版本：

```shell
$ npm version
{ bleak: '1.0.4',
  npm: '2.15.0',
  ares: '1.10.1-DEV',
  http_parser: '2.5.2',
  icu: '56.1',
  modules: '46',
  node: '4.4.2',
  openssl: '1.0.2g',
  uv: '1.8.0',
  v8: '4.5.103.35',
  zlib: '1.2.8' }
```

### npm help
和大多数命令行工具一样，npm也有一套强大的内建帮助系统。描述和概要都已提供。它们本质上和[man-pages](https://www.kernel.org/doc/man-pages/)一样

```shell
$ npm help test
NAME
       npm-test - Test a package

SYNOPSIS
           npm test [-- <args>]

           aliases: t, tst

DESCRIPTION
       This runs a package's "test" script, if one was provided.

       To run tests as a condition of installation, set the npat config to true.
```

## 使用 `npm init`初始化一个新项目
当开始一个新的项目时，`npm init`可以通过互动的形式帮你创建一个`package.json`文件。它将会时时的询问你一些项目的配置问题，包括项目的名称以及描述等等。当然，这里有一个更加快速的解决方案：

```shell
npm init --yes
```

如果你使用了`npm init --yes`，它将不会询问任何东西，它会默认的创建`package.json`文件。要设置默认值，你可以使用如下命令：

```shell
npm config set init.author.name YOUR_NAME
npm config set init.author.email YOUR_EMAIL
```

## 查找npm包
从成千上万个模块中查找到合适你的包将会是非常有挑战性的事。根据我们的经验，以及我们上一个系列[《Node.js survey》](https://blog.risingstack.com/node-js-developer-survey-results-2016/)的开发者的实践，告诉我门，挑选合适的npm包是一件令人烦恼的事。下面让我们来尝试者挑选一个能够帮助我们发送HTTP请求的模块。

有一个网站[npms.io](https://npms.io)可以让这个任务变的简单。它展示了一些包的指标，例如质量，流行度，以及维护。通过这些指标我们可以估算出一个模块是否含有过时的模块依赖，是否有代码检查配置，是否包含测试以及最近的提交是什么时候进行的。

![npms.io](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/node-js-best-practices-finsing-npm-packages.png)

## 调研npm包
一旦我们选择了我们的模块（我们将选择request作为例子），我们应该看一下它的文档，检查一下它的公开问题（open issues），以得到我们将要引入到我们项目的模块的大概了解。不要忘了，你引用了越多的npm包，你的项目将会面临更高的脆弱和恶意的风险。如果你想了解更多的关于npm安全的风险，请阅读我们的相关指南[npm-related-security-risks](https://blog.risingstack.com/controlling-node-js-security-risk-npm-dependencies/)。

如果你想在命令行中打开模块的主页，你可以使用：

```shell
npm home request
```

如果要检查公开问题或者公开可用的roadmap（如果有），你可以尝试：

```shell
npm bugs request
```

或者，如果你想看一下模块的git仓库，你可以这样：

```shell
npm repo request
```

## 保存依赖
一旦你确定你要引入到你项目中的模块，你可以安装保存它。最常用的命令是`npm install request`。

如果你想一步到位并且自动添加到你的package.json文件中，你可以这样：

```shell
npm install request --save
```

npm默认将会使用`^`作为前缀保存你的依赖。这意味着在下一个npm安装期间，将安装没有主版本碰撞的最新模块。要更改此行为，你可以：

```shell
npm config set save-prefix='~'
```

如果你要保存确切的版本，你可以：

```js
npm config set save-exact true
```

## 锁定依赖关系
即使你保存的模块具有完整的版本号，像上一节所示。你应该知道，大多数npm模块的作者并不这样做。这样做完全没问题，他们不这样做的目的是自动获取补丁和最新的功能。

这样情况下很容易会对于生产环境的部署造成问题。如果某个人刚刚发布了一个新的版本，那么在生产环境中可能会有不同的版本。当这个新版本有一些bug时，将会影响到你的生产系统。

要解决这个问题，你可能会使用`npm shrinkwrap`。它会生成一个npm-shrinkwrap.json文件，它不仅包含你机器上安装的模块的确切版本，还包含其依赖项的版本，等等。一旦你有这个文件，npm将会使用它来重现相同的依赖关系树。

## 检查过时的依赖
要想检查过时的依赖项，npm有一个内置的工具方法`npm outdated`命令。你可以在要检查的项目目录运行它：

```shell
$ npm outdated
conventional-changelog    0.5.3   0.5.3   1.1.0  @risingstack/docker-node
eslint-config-standard    4.4.0   4.4.0   6.0.1  @risingstack/docker-node
eslint-plugin-standard    1.3.1   1.3.1   2.0.0  @risingstack/docker-node
rimraf                    2.5.1   2.5.1   2.5.4  @risingstack/docker-node
```

一旦你维护更多的项目，要在每个项目中所有依赖保持最新，将会成为一个压倒一切的任务。要自动化此任务，你可以使用Greenkeeper，一旦有一个依赖更新，Greenkeeper将会自动向你的存储库发送拉取请求。

## 项目中没有`devDepenendencies`
开发过程中的依赖被称作开发依赖，这是因为你可能不必在生产环境中安装它们。它将会使你的部署影响更小，更安全。因为你有更少的模块依赖在生产环境中。

要安装生产依赖，你可以这样：

```shell
 npm install --production
```

或者，你可以设置`NODE_ENV`环境变量为production:

```shell
$ NODE_ENV=production npm install
```

## 保护你的项目和token
在使用已登录用户使用npm的情况下，你的npm令牌(token)将被保存在`.npmrc`文件中。由于很多开发者将隐藏文件（dotfiles）存储到githuh上，有时候这些令牌被无意发布。目前，在github上搜.npmrc文件时，将会有成千上万的结果，很大比例都包含令牌。如果你的仓库中含有隐藏文件，**再次确认你的凭据没有被推送**。

另一个可能的安全问题来源是意外发布到npm的文件。默认情况下，npm遵守.gitignore文件，并且不会发布与这些规则匹配的文件。但是如果添加.npmignore文件，它将覆盖.gitignore的内容，因此它们不会合并。

## 发布包
当在本地开发完成包后，你经常会在发布到npm前，想要在自己的项目中尝试一下。这就是需要`npm link`的救援了。

`npm link`的作用是在全局文件夹中创建一个符号链接，链接到执行`npm link`的包。

你可以在另一个位置运行`npm link package-name`，以创建从全局安装的软件包名称到当前文件夹的`/node_modules`目录的符号链接。

让我们用实际行动看看它：

```shell
# create a symlink to the global folder
/projects/request $ npm link

# link request to the current node_modules
/projects/my-server $ npm link request

# after running this project, the require('request')
# will include the module from projects/request
```
