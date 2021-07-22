---
title: 开发必备工具
date: 2016-11-20 10:45:43
tags: [开发工具]
categories:
- 闲聊杂谈
- 开发工具
---

作为一个开发者，工具是提升效率的一大法器，一款好的开发工具，能使开发效率大大加快，而且还能保持心情愉悦。本篇文章，记录一下自己在平时开发的过程中遇到的比较喜欢的开发工具，我在选择工具时，会尽量选择同时兼容linux和mac平台的，如果不兼容，会在每个平台找出替代。会随时不定时更新。

<!--more-->

## 编辑器&&IDE
### sublime text
编辑器首推sublime，轻量高效，插件丰富，足以应对各种开发需求。
#### 插件推荐
##### emmet:前端开发者必备插件，写html神器。
##### jsFormat:js格式化插件
##### Pretty JSON:格式化json插件
##### HTMLBeauty:html格式化工具
##### CSS Format:css格式化工具
##### DocBlockr:自动生成文档
##### Clang Format:将代码格式化成Clang样式
##### Block Cursor Everywhere:将光标转换成块光标样式
##### BrackHighlighter:括号高亮显示
##### HtmlMinifier:html压缩工具
##### Less:支持Less
##### Markdown Preview:markdown预览工具
##### MarkdownEditing:markdown编辑
##### SqlBeautifier:sql格式化

### atom
github出的一款编辑器，由于使用的是electron技术，资源占用有点多，但是插件丰富。

### vscode
vscode是微软出的一款编辑器，基于atom，除了插件外，最大的亮点在于，集成了node调试工具，能直接调试node项目。

### WebStorm
很强大的js开发调试工具

### Clion
C语言IDE

## 终端工具
### iterm2(mac)
很强大的终端工具，可替代系统自带的终端
### terminator(linux)
linux下一款牛叉的终端
### autojump
目录自动跳转工具，可用`j 目录名`直接跳转到目录，根据使用目录频率自动设置优先级。相当牛。
### zsh
替换自带的bash

## git 图形化工具
### gitk,git-gui
git自带的两个工具，gitk可以查看提交历史记录，git-gui图形化查看文件变化，非常方便。在linux下体验很棒，但是在mac下体验非常糟糕。
### GitKraken
使用electron构建的一个图形化工具，跨平台，体验还不错，除了占用资源有点多，而且会有小bug。

## mongodb工具
### robomongo
这个工具最大的特色就是，直接使用shell查询，结果有三种不同的样式展示，表格，树，json
### MongoChef
和robomongo不相上下，但是收费的，个人可是使用 Non-Commercial-License 版本。

## mysql工具
### datagrip
jetbrains出的一款数据库连接工具，使用jdbc，因此支持多种数据库
### dbeaver
基于eclipse的，也支持多种数据库
### MysqlWorkBench
mysql官方出的工具

## markdown编辑器
### [Haroopad](http://pad.haroopress.com/user.html#download)
md编辑器，编辑模式，预览模式，左右分栏，支持vim模式，同时跨平台，超赞的md编辑器
### CMD Editor
作业部落出的md编辑器，支持vim模式，emacs模式，可以直接同步到作业部落，印象笔记等，可以选择在线版本，也可以选择客户端。
### Typora
不支持vim模式，但是轻量，可以作为md的预览工具。

### MWebLite（mac）
不支持vim模式，但是轻量，可以作为md的预览工具。

## diff工具
### [Beyond Compare](http://www.scootersoftware.com/download.php)
很棒的一款diff工具，支持文件和目录比较，合并，还支持直接文本比较合并，但是收费的，有30天试用。mac下可以将`/Users/coolcao/Library/Application Support/Beyond Compare`目录下文件删除，即可恢复30天试用，或者，暴力点，直接将该目录设置为不可写即可。
### meld
也是一款很棒的diff工具，支持文件，目录，linux下体验超棒，mac下体验不如linux
### [diffMerge](http://www.sourcegear.com/diffmerge/downloads.php)

## 脑图
### freeMind
### xmind
很强大的工具，但是收费，基础功能够用了。

## 抓包
### wireshark
超级牛的一个抓包工具
### charles

## RSS阅读器
### QuiteRSS
跨平台的一款RSS阅读工具，内置浏览器，体验不错。但是在mac上不知道怎么回事，内置浏览器方面有点问题。
### thunderbird
邮件客户端，可以支持RSS订阅。也是内置一个浏览器，体验不错。推荐。缺点也是不支持二级目录。
### Feedline(chrome插件)
做的非常漂亮的一个RSS阅读器，唯一缺点是不支持分类目录，很崩溃。
### NewsFox(firefox插件)
### LuckNews(mac)
mac上的一款体验不错的免费的RSS阅读器，推荐。

## chrome 插件
### postman
rest调试工具，推荐，此外，如果不想用chrome插件，可以直接下载其使用electron封装的独立app
### DHC Rest Client
Rest调试工具
### devdocs
开发者文档插件，这里集成了丰富的文档，不用为了各种文档东跑西凑了
### StackEdit
markdown编辑工具，支持TOC,数学公式，UML图，流程图等，很强大。
### 马克飞象
一款markdown编辑工具，可以支持连接evernote进行同步，不过现在同步功能需要收费了。
### JSON-handle
格式化JSON
### WEB前端助手
包括字符串编码，代码压缩，美化，JSON格式化，正则表达式，时间转换，二维码生成等等
### 工具箱
有用的工具集，常用的文本处理工具，排序，去重，修整，MD5摘要，SHA1摘要，代码格式化等
### ng-inspector for AngularJS
调试angularjs的工具
### Mobile/Responsive Web Design Tester
相应式页面调试工具
### Ripple Emulator
相应式页面调试工具
### 掘金
开发资讯插件
