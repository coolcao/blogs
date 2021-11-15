---
title: git多用户配置不同工作目录.md
date: 2021-09-17 10:10:40
tags: []
categories:
- 技术博客
- 原创
---

我们在开发使用git时，经常会遇到，一台电脑上会配置多个git用户，不同git用户可能在不同的工作目录开发，如果每次都重新设置用户信息，将会很麻烦。

git自2.13版本开始，可以使用`includeIf`进行配置。

首先，根据用户配置多个gitconfig文件：

![](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1631939233_20210917101412912_1786377412.png)


其中`.gitconfig`为全局配置，`.gitconfig-self`和`.gitconfig-work`为区分不同环境，不同目录创建的不同配置。

在`.gitconfig-self`中做如下配置：

```
[user]
	email = self-email
	name = self-name
```

同理，在`.gitconfig-work`中也做如下配置：

```
[user]
	email = work-email
	name = work-name
```

然后在`.gitconfig`全局配置文件中，添加引用：

```
[includeIf "gitdir:/home/code/self/"]
    path = .gitconfig-self
[includeIf "gitdir:/home/code/work/"]
    path = .gitconfig-work
```