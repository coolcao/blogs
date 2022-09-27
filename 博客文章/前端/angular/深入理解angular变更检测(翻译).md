---
title: 深入理解angular变更检测(翻译).md
date: 2021-11-22 09:01:06
tags: [angular, 变更检测]
categories:
- 技术博客
- 翻译
---

> 翻译自[《The Last Guide For Angular Change Detection You'll Ever Need》](https://indepth.dev/posts/1305/the-last-guide-for-angular-change-detection-youll-ever-need)

Angular的Change Detection是该框架的一个核心机制，但（至少从我的经验来看）它非常难以理解。不幸的是，在官方网站上没有关于这个主题的官方指南。

在这篇博文中，我将为你提供所有你需要了解的关于变更检测的必要信息。我将通过使用我为这篇博文建立的一个演示项目来解释其机制。

## 什么是变更检测
Angular的两个主要目标是可预测性和性能。该框架通过结合状态和模板在UI上展现我们应用程序的状态。

![](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1637547305_20211122090548790_1248588036.png)

如果状态发生任何变化，也有必要更新DOM。这种将HTML与我们的数据同步的机制被称为 "变更检测"。每个前端框架都使用其实现，例如React使用虚拟DOM，Angular使用变更检测，等等。我可以推荐一篇文章[《Change And Its Detection In JavaScript Frameworks》](https://teropa.info/blog/2015/03/02/change-and-its-detection-in-javascript-frameworks.html)，它对这个话题做了很好的概括。

> 变更检测。当数据发生变化时，更新DOM的过程。

作为开发者，大多数时候我们不需要关心变更检测，直到我们需要优化应用程序的性能。如果处理不当，变更检测会降低大型应用程序的性能。

## 变更检测如何工作的
一个变更检测周期可以分成两部分。

1. 开发者更新应用模型
2. Angular通过重新渲染来同步DOM中更新的模型

让我们更详细地看一下这个过程。

开发者更新数据模型，例如，通过更新组件绑定
Angular检测到这个变化，变更检测从上到下检查组件树中的每个组件，看看相应的模型是否发生了变化，如果有一个新的值，它将更新组件的视图（DOM）。

下面的GIF以一种简化的方式演示了这个过程。

![](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1637547306_20211122090956979_1956406215.png)

图片显示了一个Angular组件树和它的变更检测器（CD），每个组件都是在应用启动过程中创建的。这个检测器将当前值与属性的先前值进行比较。如果值发生了变化，它将设置isChanged为真。看看[框架代码中的实现](https://github.com/angular/angular/blob/885f1af509eb7d9ee049349a2fe5565282fbfefb/packages/core/src/util/comparison.ts#L13)，这只是一个===比较，对NaN有特殊处理。

> 变更检测并不进行深入的对象比较，它只比较模板所使用的属性的先前值和当前值。

## Zone.js
一般来说，一个区可以跟踪和拦截任何异步任务。

一个区通常有这些阶段。

- 它开始时是稳定的
- 如果任务在该区运行，它就变得不稳定
- 如果任务完成，它又变得稳定
- Angular在启动时修补了几个低级别的浏览器API，以便能够检测到应用程序中的变化。这是通过zone.js完成的，它修补的API包括EventEmitter、DOM事件监听器、XMLHttpRequest、Node.js的fs API等。

简而言之，如果发生以下事件之一，该框架将触发变更检测：

- 任何浏览器事件（click、keyup等）。
- setInterval()和setTimeout()。
- 通过XMLHttpRequest的HTTP请求

Angular使用它的区域称为NgZone。只存在一个NgZone，变更检测只针对在这个区域触发的异步操作。

## 性能
默认情况下，Angular变更检测从上到下检查所有组件的模板值是否有变化。

Angular对每一个组件进行变更检测的速度非常快，因为它可以在几毫秒内使用[内联缓存](http://mrale.ph/blog/2012/06/03/explaining-js-vms-in-js-inline-caches.html)执行数千次检查，从而产生VM优化的代码。

如果你想更深入地了解这个话题，我建议你观看Victor Savkin关于[变更检测重塑的演讲](https://www.youtube.com/watch?v=jvKGQSFQf10)。

尽管Angular在幕后做了很多优化工作，但在大型应用中，性能仍然会下降。在下一章中，你将学习如何通过使用不同的变更检测策略来主动提高Angular的性能。

## 变更检测策略
Angular提供了两种策略来运行变更检测。

- Default
- OnPush

让我们来看看这些策略。

### Default（默认的变更检测策略）
默认情况下，Angular使用ChangeDetectionStrategy.Default变更检测策略。这个默认策略在每次事件触发变更检测（如用户事件、定时器、XHR、Promise等）时，从上到下检查组件树中的每个组件。这种保守的检查方式没有对组件的依赖性做任何假设，被称为`脏检查`。在由许多组件组成的大型应用程序中，它可能对你的应用程序的性能产生负面影响。

![](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1637547307_20211122091910774_1197136288.png)

### OnPush
我们可以通过在组件装饰器元数据中添加changeDetection属性来切换到ChangeDetectionStrategy.OnPush变更检测策略。

```typescript
@Component({
    selector: 'hero-card',
    changeDetection: ChangeDetectionStrategy.OnPush,
    template: ...
})
export class HeroCard {
    ...
}
```

这个变更检测策略提供了跳过对这个组件和它的所有子组件的不必要检查的可能性。

下一个GIF演示了通过使用OnPush变更检测策略来跳过部分组件树。

![](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1637547307_20211122092043746_495303673.png)

使用这个策略，Angular知道只有在以下情况下才需要更新组件。

- 输入的引用发生了变化
- 该组件或它的一个子节点触发了一个事件处理程序
- 变更检测被手动触发了
- 通过异步管道链接到模板的观测器（observable）发出了一个新的值

让我们仔细看看这些类型的事件。

1. 输入引用发生变化
在默认的变更检测策略中，Angular会在`@Input()`数据被改变或修改时运行变更检测器。使用OnPush策略，只有当一个`新的引用`作为@Input()值被传递时，才会触发变更检测。

JavaScript中的所有东西都是通过引用传递的，但所有的基元都是不可改变的，它们的字面表述都指向同一个基元实例/引用。修改对象属性或数组条目不会创建一个新的引用，因此不会触发OnPush组件的变更检测。要触发变更检测器，你需要传递一个新的对象或数组引用来代替。

你可以通过[这个简单的例子](https://angular-change-detection-demo.netlify.com/simple-demo)来测试其行为：

- 用ChangeDetectionStrategy.Default修改HeroCardComponent的年龄。
- 确认使用`ChangeDetectionStrategy.OnPush`的HeroCardOnPush组件没有反映出改变后的年龄（组件周围的红色边框是直观的）。
- 在 "修改英雄 "面板上点击 "创建新的对象参考"。
- 验证带有`ChangeDetectionStrategy.OnPush`的HeroCardOnPushComponent是否被变更检测所检查。

![](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1637547308_20211122092654984_922216339.png)

为了防止变更检测的错误，通过只使用不可变的对象和列表，使用OnPush变更检测到处构建应用程序是很有用的。不可变的对象只能通过创建一个新的对象引用来修改，所以我们可以保证：
- 每次变化都会触发OnPush变更检测
- 我们不会忘记创建一个新的对象引用，这可能会导致错误。

[Immutable.js](https://facebook.github.io/immutable-js/)是一个不错的选择，该库为对象（Map）和列表（List）提供了持久的不可变数据结构。通过npm安装该库提供了类型定义，这样我们就可以在IDE中利用类型泛型、错误检测和自动完成的优势。

2. 事件处理被触发
如果OnPush组件或它的一个子组件触发了一个事件处理程序，如点击一个按钮，就会触发变更检测（对组件树中的所有组件）。

请注意，以下动作不会触发使用OnPush变更检测策略的变更检测：
- `setTimeout`
- `setInterval`
- `Promise.resolve().then()`, (当然，`Promise.reject().then()`也是一样)
- `this.http.get('...').subscribe()` (一般来说，任何RxJS 可观察的订阅)

你可以通过这个[简单的例子](https://angular-change-detection-demo.netlify.com/simple-demo)来测试其行为：
1. 点击HeroCardOnPushComponent中的 "Change Age "按钮，它使用`ChangeDetectionStrategy.OnPush`
2. 验证变更检测是否被触发并检查所有的组件

![](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1637547308_20211122093527364_1300529174.png)

3. 手动触发变更检测
有三种方法可以手动触发变更检测。

- `ChangeDetectorRef#detectChanges()`，该方法通过牢记变更检测策略在该视图及其子视图上运行变更检测。它可以与detach()结合使用，以实现局部的变更检测检查。
- `ApplicationRef.tick()`，通过尊重组件的变更检测策略来触发整个应用的变更检测。
- `ChangeDetectorRef#markForCheck()`，它不触发变更检测，但将所有OnPush祖先标记为被检查一次，作为当前或下一个变更检测周期的一部分。它将在被标记的组件上运行变更检测，即使它们正在使用OnPush策略。

> 手动运行变更检测不是一个黑客，但你应该只在合理的情况下使用它

下面的插图以可视化的方式展示了不同的ChangeDetectorRef方法:
![](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1637547309_20211122094421294_905268252.png)

你可以使用简单演示中的 "DC"（detectChanges()）和 "MFC"（markForCheck()）按钮测试其中一些动作。

4. 异步管道
内置的[AsyncPipe](https://angular.io/api/common/AsyncPipe)订阅了一个可观察变量，并返回它所发射的最新值。

在内部，AsyncPipe在每次发出新值时都会调用markForCheck，请看其[源代码](https://github.com/angular/angular/blob/5.2.10/packages/common/src/pipes/async_pipe.ts#L139)：
```typescript
private _updateLatestValue(async: any, value: Object): void {
  if (async === this._obj) {
    this._latestValue = value;
    this._ref.markForCheck();
  }
}
```

如图所示，AsyncPipe自动使用OnPush变更检测策略工作。因此，建议尽可能多地使用它，以方便以后从默认的变更检测策略切换到OnPush。

你可以在[异步演示](https://angular-change-detection-demo.netlify.com/async-pipe-demo)中看到这种行为的作用。

![](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1637547309_20211122094758016_2069356016.png)

第一个组件通过AsyncPipe直接将一个观察变量绑定到模板上:
```html
<mat-card-title>{{ (hero$ | async).name }}</mat-card-title>
```

```typescript
  hero$: Observable<Hero>;

  ngOnInit(): void {
    this.hero$ = interval(1000).pipe(
        startWith(createHero()),
        map(() => createHero())
      );
  }

```

而第二个组件订阅了观察器并更新了一个数据绑定值:
```html
<mat-card-title>{{ hero.name }}</mat-card-title>

```

```typescript
  hero: Hero = createHero();

  ngOnInit(): void {
    interval(1000)
      .pipe(map(() => createHero()))
        .subscribe(() => {
          this.hero = createHero();
          console.log(
            'HeroCardAsyncPipeComponent new hero without AsyncPipe: ',
            this.hero
          );
        });
  }

```

正如你所看到的，没有AsyncPipe的实现不会触发变更检测，所以我们需要为观察者发出的每个新事件手动调用 detectChanges()。

## 避免循环变更检测和ExpressionChangedAfterCheckedError错误
Angular包括一个检测变更检测循环的机制。在开发模式下，框架会运行两次变更检测，以检查值在第一次运行后是否有变化。在生产模式下，变更检测只运行一次，以获得更好的性能。

我在我的[ExpressionChangedAfterCheckedError演示](https://angular-change-detection-demo.netlify.com/expression-changed-demo)中强制执行了这个错误，如果你打开浏览器控制台就可以看到。

![](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1637547310_20211122095932509_1168897747.png)

在这个演示中，我通过更新ngAfterViewInit生命周期钩子中的hero属性来迫使错误发生。

```typescript
  ngAfterViewInit(): void {
    this.hero.name = 'Another name which triggers ExpressionChangedAfterItHasBeenCheckedError';
  }

```

为了理解为什么会出现这样的错误，我们需要看一下变更检测运行期间的不同步骤。

![](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1637547311_20211122100034774_484031073.png)


正如我们所看到的，AfterViewInit生命周期钩子是在当前视图的DOM更新被渲染后被调用的。如果我们改变这个钩子中的值，它在第二次变更检测运行中（如上所述，在开发模式下自动触发）会有不同的值，因此Angular会抛出ExpressionChangedAfterCheckedError。

我强烈推荐Max Koretskyi的文章[《Everything you need to know about change detection in Angular](https://blog.angularindepth.com/everything-you-need-to-know-about-change-detection-in-angular-8006c51d206f)》，它更详细地探讨了著名的ExpressionChangedAfterCheckedError的基础实现和使用案例。

## 运行代码而不进行变更检测
可以在NgZone之外运行某些代码块，这样就不会触发变更检测。
```typescript
  constructor(private ngZone: NgZone) {}

  runWithoutChangeDetection() {
    this.ngZone.runOutsideAngular(() => {
      // the following setTimeout will not trigger change detection
      setTimeout(() => doStuff(), 1000);
    });
  }

```

这个简单的演示提供了一个按钮来触发Angular区域外的动作。
![](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1637547311_20211122100259734_442189075.png)

你应该看到，该动作被记录在控制台中，但HeroCard组件没有被检查，这意味着它们的边框没有变成红色。

这种机制对Protractor运行的E2E测试非常有用，特别是当你在测试中使用browser.waitForAngular时。在每次向浏览器发送命令后，Protractor会等待，直到该区域变得稳定。如果你使用setInterval，你的区域将永远不会变得稳定，你的测试可能会超时。

同样的问题也会发生在RxJS的观测器上，但因此你需要在polyfill.ts中添加一个补丁版本，如Zone.js对非标准API的支持中所述。

```typescript
import 'zone.js/dist/zone';  // Included with Angular CLI.
import 'zone.js/dist/zone-patch-rxjs'; // Import RxJS patch to make sure RxJS runs in the correct zone

```

如果没有这个补丁，你可以在ngZone.runOutsideAngular里面运行可观察的代码，但它仍然会作为NgZone里面的一个任务运行。

## 停用变更检测
有一些特殊的用例，停用变更检测是有意义的。例如，如果你使用WebSocket从后端向前端推送大量的数据，而相应的前端组件应该每10秒才更新一次。在这种情况下，我们可以通过调用detach()来停用变更检测，并使用detectChanges()手动触发。

```typescript
constructor(private ref: ChangeDetectorRef) {
    ref.detach(); // deactivate change detection
    setInterval(() => {
      this.ref.detectChanges(); // manually trigger change detection
    }, 10 * 1000);
  }

```

在Angular应用程序的启动过程中，也可以完全停用Zone.js。这意味着自动变更检测被完全停用，我们需要手动触发UI变化，例如通过调用ChangeDetectorRef.detectChanges（）。

首先，我们需要从polyfills.ts中注释出Zone.js的导入。

```typescript
import 'zone.js/dist/zone';  // Included with Angular CLI.

```

接下来，我们需要在main.ts中传递noop区域。

```typescript
platformBrowserDynamic().bootstrapModule(AppModule, {
      ngZone: 'noop';
}).catch(err => console.log(err));

```

关于停用Zone.js的更多细节可以在[Angular Elements without Zone.Js](https://www.softwarearchitekt.at/aktuelles/angular-elements-part-iii/)一文中找到。

## Ivy
从Angular 9开始，Angular默认使用Ivy，Angular的下一代编译和渲染管道。

Ivy仍然以正确的顺序处理所有的框架生命周期钩子，所以变更检测工作与以前一样。所以你仍然会在你的应用程序中看到相同的ExpressionChangedAfterCheckedError。


Max Koretskyi在[这篇文章](https://blog.angularindepth.com/ivy-engine-in-angular-first-in-depth-look-at-compilation-runtime-and-change-detection-876751edd9fd)写道:
> 正如你所看到的，所有熟悉的操作仍然在这里。但操作的顺序似乎已经改变了。例如，现在Angular似乎首先检查子组件，然后才是嵌入式视图。由于目前还没有编译器产生适合测试我的假设的输出，所以我不能确定。

你可以在本博文末尾的 "推荐文章 "部分找到另外两篇与常春藤有关的有趣文章。

## 总结
Angular变更检测是一个强大的框架机制，它可以确保我们的UI以可预测和可执行的方式表示我们的数据。可以说，变更检测只是对大多数应用程序起作用，特别是如果它们不是由50多个组件组成的。

作为一个开发者，你通常需要深入研究这个话题，有两个原因。

- 你收到一个ExpressionChangedAfterCheckedError，需要解决它
- 你需要提高你的应用程序的性能

我希望这篇文章能帮助你更好地理解Angular的变更检测。欢迎使用我的演示项目来玩一玩不同的变更检测策略。

## 推荐文章
[Angular Change Detection - How Does It Really Work?](https://blog.angular-university.io/how-does-angular-2-change-detection-really-work/)
[Angular OnPush Change Detection and Component Design - Avoid Common Pitfalls](https://blog.angular-university.io/onpush-change-detection-how-it-works/)
[A Comprehensive Guide to Angular onPush Change Detection Strategy](https://netbasal.com/a-comprehensive-guide-to-angular-onpush-change-detection-strategy-5bac493074a4)
[Angular Change Detection Explained](https://blog.thoughtram.io/angular/2016/02/22/angular-2-change-detection-explained.html)
[Angular Ivy change detection execution: are you prepared?](https://blog.angularindepth.com/angular-ivy-change-detection-execution-are-you-prepared-ab68d4231f2c)
[Understanding Angular Ivy: Incremental DOM and Virtual DOM](https://blog.nrwl.io/understanding-angular-ivy-incremental-dom-and-virtual-dom-243be844bf36)

