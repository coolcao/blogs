---
title: 使用nativefier打包网站为桌面应用
date: 2016-11-21 19:22:58
tags: [开发工具]
categories:
- 闲聊杂谈
- 开发工具
---

近两天发现一个工具，nativefier，可以将网站应用打包成桌面应用。
nativefier是基于electron的，当然，你也可以直接使用electron打包，灵活性更高。

<!--more-->

## 安装node-icns（非必须，转换图标）
```js
npm install -g node-icns
```

## 转换图标
```js
nicns --in app-icon.png --out app-icon.icns
```

## 安装nativefier
nativefier是基于nodejs和electron的打包工具，安装nativefier之前请确保已安装nativefier和electron。
```js
sudo npm install -g nativefier  //全局安装nativefier
```

## 打包
```js
nativefier --name 'ccsite' --icon ccsite.icns 'http://coolcao.com'
```

然后我们就在当前目录下打包了应用
![ccsite](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/websites2app.png)
打开应用看一下，其实就是electron架了个壳子。
你可以根据具体的平台打包成相应的二进制包，linux,windows,mac随意。
唯一不足的地方在于，这只能打包在线网站，如果想做本地应用，可以直接使用electron。
