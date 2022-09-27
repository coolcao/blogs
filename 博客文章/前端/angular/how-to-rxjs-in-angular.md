---
title: Angular中使用RxJS的几种常用方法（翻译）
date: 2022-09-27 16:41:41
tags: [Angular, RxJS]
categories:
  - 技术博客
  - 翻译
---

> [How to RxJS in Angular](https://www.matthiasmeier.io/blog/how-to-rxjs-in-angular/)

经过几年的 Angular 前端开发和 RxJS 的大量使用，我决定把我个人的一些关键经验写成一篇简洁的文章。在这篇文章中，我假设你已经对 Observable-streams 以及不同的 Subject-types 的工作方式有了一些基本的了解。如果是这样的话，这可能会帮助你掌握 Angular 中 RxJS 的最常见的使用情况。

<!--more-->

当你的组件输入的值随时间变化时，你可能想在你的组件内对这些数据做一些处理。所以你通常做的是实现 `NgOnChanges` 方法来对这些变化做出反应。现在，当你依赖其他异步数据时，这就变得有点模糊了，这正是 RxJS 出现的地方。不幸的是，目前还不支持组件的输入流。

因此，每当我需要 RxJS 在改变组件输入上的能力时，我就会使用这种模式。

```ts
@Input() public amount: number;
public amount$ = new Subject();

public ngOnChanges(changes: SimpleChanges): void {
  if (changes.amount && changes.amount.currentValue !== undefined) {
    this.amount$.next(changes.amount.currentValue);
  }
}
```

## 按钮点击（Button Click）

假设你有一些基于 RxJS 的计数器，你想在点击 "重置 "按钮时重置这个计数器。最直接的方法（我知道）是，创建一个新的`onReset$`主题，然后你可以把它编入你的计数器流设置。

```html
<button (click)="onReset$.next()">Reset</button>
```

## Http Requests

对于数据服务的基本设置，你可以遵循 [Angular 指南](https://angular.io/guide/http)。
但通常情况下，有些用例中，检索数据并不那么简单。很多时候，你要依靠其他的异步数据。所以这方面的心理模型可以是。

> 一旦我得到数据 X，我想加载数据 Y（它与 X 有关）。

很多时候，你可以用以下方式来扩展上述语句

> 如果仍有一个开放的 Y 请求（与**之前的**X 有关），我想取消那个请求。

如果是这样，那么 RxJS 的 `switchMap` 操作符正是你想要找的。考虑一下下面的例子，检索我们当前用户的书签。

```ts
public bookmarks$ = this.currentUser$.pipe(
  switchMap(user => this.bookmarksService.getBookmarks(user.id)),
);
```

## Async Pipe

Angular 包含了一个非常有用的 `async-pipe`，这使得在模板中消费你的可观察流变得容易。因此，与其在你的 TypeScript 组件中订阅并将值分配给一个公共属性，你可以直接使用 `async-pipe` 来检索你的异步值。

```html
<ul>
  <li *ngFor="let bookmark of bookmarks$|async">{{bookmark}}</li>
</ul>
```

## Sharing Expensive Data

使用上面提到的技术，你可以继续使用 `async-pipe` 多次订阅你的 bookmarks$流。当看一下你的网络控制台时，你很快就会意识到，多个请求正在进行。这不是我们想要的。相反，我们希望与任何对它感兴趣的人分享响应。

有多种方法可以实现这一点。其中之一是 `publishReplay`操作符，它将源流作为 ReplaySubject 发布，然后是 `refCount`操作符，它处理（取消）订阅，只要有至少一个监听器。

```ts
public bookmarks$ = this.currentUser$.pipe(
  switchMap(user => this.bookmarksService.getBookmarks(user.id)),
  publishReplay(1),
  refCount(),
);
```

## Unsubscribing

关于这篇文章的最后一点。如果你总是使用 `async-pipe`，就不需要担心取消订阅的问题，因为管道会自己处理这个问题。所以，当然，尽可能地使用它是非常有意义的。但在某些情况下，你可能需要手动订阅可观察流。在这些情况下，重要的是你要正确地取消订阅，否则你会遇到内存泄漏。使用 `takeUntil`操作符 是我发现的完成这个任务的最干净的方法。

前面的设置看起来像这样。

```ts
private unsubscribe$ = new Subject();

public ngOnDestroy(): void {
  this.unsubscribe$.next();
  this.unsubscribe$.complete();
}
```

因此，在任何需要关闭的流上，你可以在调用 "subscribe "之前使用 `takeUntil`，当你的组件即将被销毁时，你的订阅就会被关闭。

```ts
this.bookmarks$.pipe(takeUntil(this.unsubscribe$)).subscribe((bookmarks) => {
  // do something
});
```

我真的希望这能帮助你们中的一些人开始进入 Angular 和 RxJS 的世界，因为一开始它可能是相当吓人的。

我们非常欢迎任何问题和建议。更多的内容将陆续推出。