> 原文： [Angular Change Detection Explained](https://blog.thoughtram.io/angular/2016/02/22/angular-2-change-detection-explained.html)

## 到底什么是变更检测
变更检测的基本任务是获取程序的内部状态，并使其以某种方式在用户界面上可见。这种状态可以是任何类型的对象、数组、基本类型数据......就是任何类型的JavaScript数据结构。

这种状态可能最终成为用户界面中的段落、表单、链接或按钮，特别是在网络上，它就是文档对象模型（DOM）。因此，基本上我们把数据结构作为输入，并生成DOM输出，将其显示给用户。我们把这个过程称为渲染。

然而，当变化发生在运行时，它将变得更加棘手。当DOM已经被渲染一段时间后，我们如何找出我们的模型中的变化，以及我们需要更新DOM的地方？访问DOM树总是很昂贵的，所以我们不仅需要找出需要更新的地方，而且我们还希望保持这种访问尽可能的小。

这可以通过许多不同的方式来解决。例如，一种方法是简单地发出一个http请求并重新渲染整个页面。另一种方法是将新状态的DOM与之前的状态进行对比，只渲染其中的差异，这就是ReactJS用虚拟DOM所做的事情。

因此，基本上变更检测的目标始终是预测数据及其变化。

## 什么引起了变化
现在我们大致知道了变更检测是怎么回事，我们可能会想，到底什么时候会发生这种变化？又是什么时候Angular知道要更新视图？好吧，让我们看一下下面的代码：

```ts
@Component({
  template: `
    <h1>{{firstname}} {{lastname}}</h1>
    <button (click)="changeName()">Change name</button>
  `
})
class MyApp {

  firstname:string = 'Pascal';
  lastname:string = 'Precht';

  changeName() {
    this.firstname = 'Brad';
    this.lastname = 'Green';
  }
}
```

上面的组件只是显示了两个属性，并提供了一个方法，当模板中的按钮被点击时，可以触发这个方法来改变这两个属性。这个按钮被点击的时刻就是应用状态改变的时刻，因为它改变了组件的属性值。这就是我们想要更新视图的时刻。

我们再来看另外一个：
```ts
@Component()
class ContactsApp implements OnInit{

  contacts:Contact[] = [];

  constructor(private http: Http) {}

  ngOnInit() {
    this.http.get('/contacts')
      .map(res => res.json())
      .subscribe(contacts => this.contacts = contacts);
  }
}
```

这个组件拥有一个 `contacts` 列表，当组件初始化时，它会发送一个HTTP请求去获取数据。一旦请求返回，`contacts` 就会被更新。同样，在这时，我们的应用程序的状态已经改变，所以我们要更新视图。

基本上，应用程序状态的改变可由三类引起：
- 浏览器事件: 比如点击事件，表单提交事件等等
- **XHR**: 从远程服务器请求数据
- 定时器：`setTimeout()`, `setInterval()`

它们都是异步的。这给我们带来的结论是，基本上只要执行了一些异步操作，我们的应用状态就可能发生变化。这时就需要有人告诉Angular来更新视图。

## 谁通知的Angular
好了，我们现在知道是什么导致了应用程序状态的改变。但是，是什么告诉Angular，在这个特定的时刻，视图必须被更新？

Angular允许我们直接使用本地API。我们不需要调用拦截器方法，这样Angular就能得到更新DOM的通知。这就是纯粹的魔法吗？

如果你关注过我们最新的文章，你就知道Zones会照顾到这一点。事实上，Angular有自己的区域，叫做`NgZone`，我们已经在我们的文章[《Angular中的区域》](https://blog.thoughtram.io/angular/2016/02/01/zones-in-angular-2.html)中写到了这一点。你可能也想读读这篇文章。

简而言之，在Angular的源代码中，有一个叫做 `ApplicationRef` 的东西，它监听 `NgZones` 的 `onTurnDone` 事件。每当这个事件被触发，它就会执行一个 `tick()` 函数，这个函数基本上是执行变更检测。

```ts
// very simplified version of actual source
class ApplicationRef {

  changeDetectorRefs:ChangeDetectorRef[] = [];

  constructor(private zone: NgZone) {
    this.zone.onTurnDone
      .subscribe(() => this.zone.run(() => this.tick());
  }

  tick() {
    this.changeDetectorRefs
      .forEach((ref) => ref.detectChanges());
  }
}
```

## 变更检测
好吧，我们现在知道什么时候触发变更检测了，但它是如何执行的呢？嗯，我们首先需要注意的是，在Angular中，每个组件都有自己的变更检测器。

这是一个重要的事实，因为这允许我们对每个组件单独控制如何以及何时进行变更检测！这一点在后面会详细介绍。稍后再谈这个问题。

让我们假设在我们的组件树的某个地方，一个事件被触发了，也许是一个按钮被点击了。接下来会发生什么？我们刚刚了解到，zone执行给定的处理程序，并在完成后通知Angular，这最终导致Angular执行变更检测。

由于每个组件都有自己的变更检测器，而一个Angular应用程序由一个组件树组成，逻辑上的结果是我们也有一个 **变更检测器树** 。这棵树也可以被看作是一个有向图，数据总是从上到下流动。

数据从上到下流动的原因是，变更检测也总是从上到下对每一个组件进行，每一次都是从根组件开始。这是非常棒的，因为单向的数据流比循环更可预测。我们总是知道我们在视图中使用的数据来自哪里，因为它只能从其组件中产生。

另一个有趣的是，变更检测在单次通过后变得稳定。这意味着，如果我们的某个组件在变更检测过程中第一次运行后引起任何额外的副作用，Angular将抛出一个错误。

## 性能
在默认情况下，即使我们必须在每次事件发生时检查每一个组件，Angular的速度也非常快。它可以在几毫秒内执行数十万次检查。这主要是由于Angular生成了对VM友好的代码。  
  
那是什么意思呢？好吧，当我们说每个组件都有自己的变更检测器时，这并不是说Angular中有一个单一的通用的东西来为每个单独的组件进行变更检测。  
  
原因是，它必须以动态的方式编写，所以它可以检查每个组件，无论其模型结构是什么样子。虚拟机不喜欢这种动态代码，因为它们无法优化它。它被认为是多态的，因为对象的形状并不总是相同的。  
  
Angular在运行时为每个组件创建变更检测器类，这是单态的，因为他们确切地知道组件模型的形状是什么。虚拟机可以完美地优化这段代码，这使得它的执行速度非常快。好在我们不需要太在意这些，因为Angular会自动做到这一点。  
  
## 更智能的变更检测
同样，Angular必须在每次事件发生时检查每个组件，因为......嗯，也许应用程序的状态已经改变了。但是，如果我们可以告诉Angular只对应用程序中改变了状态的部分运行变更检测，那不是更好吗？

是的，事实上我们可以这样做! 事实证明，有一些数据结构可以为我们提供一些保证，即Immutables和Observables。如果我们碰巧使用了这些结构或类型，并且告诉Angular，变更检测就会快得多。好吧，很酷，但怎么说呢？

### 理解可变性

为了理解为什么以及不可变的数据结构会有帮助，我们需要理解可变性的含义。假设我们有以下组件:
```ts
@Component({
  template: '<v-card [vData]="vData"></v-card>'
})
class VCardApp {

  constructor() {
    this.vData = {
      name: 'Christoph Burgdorf',
      email: 'christoph@thoughtram.io'
    }
  }

  changeData() {
    this.vData.name = 'Pascal Precht';
  }
}
```

VCardApp使用作为一个子组件，它有一个输入属性vData。我们用VCardApp自己的vData属性向该组件传递数据。vData是一个有两个属性的对象。此外，还有一个方法 `changeData()`，可以改变vData的名字。这里没有什么神奇的事情发生。  
  
重要的是，`changeData()`通过改变vData的名称属性来改变它。尽管该属性将被改变，但vData的**引用本身却保持不变**。  
  
假设某个事件导致`changeData()`被执行，那么在进行变更检测时会发生什么呢？首先，vData.name被改变，然后它被传递给。的变更检测器现在检查给定的vData是否仍然和以前一样，是的，它是。引用并没有改变。然而，名称属性已经改变了，所以Angular还是会对该对象进行变更检测。  
  
因为在JavaScript中，对象默认是可变的（除了基本类型数据），Angular必须保守一点，在事件发生时，对每个组件都要运行变更检测。  
  
这就是不可变的数据结构发挥作用的地方。  
  
## 不可变对象
不可变的对象给我们提供了保证，即对象不能改变。意思是说，如果我们使用不可变的对象，并且我们想在这样的对象上做一个改变，我们总是会得到一个带有该改变的新引用，因为原始对象是不可变的。

```ts
var vData = someAPIForImmutables.create({
              name: 'Pascal Precht'
            });

var vData2 = vData.set('name', 'Christoph Burgdorf');

vData === vData2 // false
```

someAPIForImmutables可以是任何我们想用于不可改变的数据结构的API。然而，正如我们所见，我们不能简单地改变名称属性。我们会得到一个有这种特殊变化的新对象，这个对象有一个新的引用。或者简而言之：如果有变化，我们会得到一个新的引用。 

## 减少检查的次数
当输入属性没有变化时，Angular可以跳过整个变更检测子树。我们刚刚知道，"变化 "意味着 "新的引用"。如果我们在Angular应用程序中使用不可变的对象，我们需要做的就是告诉Angular，如果一个组件的输入没有变化，它可以跳过变更检测。

让我们看一下 `<v-card>` 的工作情况:

```ts
@Component({
  template: `
    <h2>{{vData.name}}</h2>
    <span>{{vData.email}}</span>
  `
})
class VCardCmp {
  @Input() vData;
}
```

我们可以看到，VCardCmp只依赖于它的输入属性。很好。我们可以告诉Angular，如果这个组件的输入没有变化，就跳过这个组件子树的变更检测，方法是这样设置变更检测策略为OnPush。

```ts
@Component({
  template: `
    <h2>{{vData.name}}</h2>
    <span>{{vData.email}}</span>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
class VCardCmp {
  @Input() vData;
}
```

这就是了! 现在想象一个更大的组件树。当使用不可变的对象时，我们可以跳过整个子树，Angular也会相应地被告知。

![](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1664433966_20220929144556733_648577653.png)

## Observables

如前所述，Observables也为我们提供了一些关于变化发生时间的保证。与不可变的对象不同，它们在发生变化时不会给我们新的引用。相反，它们会触发我们可以订阅的事件，以便对其做出反应。

所以，如果我们使用Observables，并且我们想使用OnPush来跳过变更检测器子树，但这些对象的引用永远不会改变，我们该如何处理呢？事实证明，Angular有一个非常聪明的方法，可以使组件树中的路径被检查出某些事件，这正是我们在这种情况下需要的。

为了理解这意味着什么，让我们看一下这个组件。

```ts
@Component({
  template: '{{counter}}',
  changeDetection: ChangeDetectionStrategy.OnPush
})
class CartBadgeCmp {

  @Input() addItemStream:Observable<any>;
  counter = 0;

  ngOnInit() {
    this.addItemStream.subscribe(() => {
      this.counter++; // application state changed
    })
  }
}
```


比方说，我们建立了一个带有购物车的电子商务应用程序。每当用户将产品放入购物车时，我们希望在用户界面中显示一个小计数器，这样用户就可以看到购物车中产品的数量。  
  
`CartBadgeCmp`正是这样做的。它有一个计数器和一个输入属性`addItemStream`，这是一个事件流，每当一个产品被添加到购物车中时，就会被触发。  
  
此外，我们将变更检测策略设置为OnPush，所以变更检测并不是一直在进行的，只有在组件的输入属性发生变化时才会进行。  
  
然而，如前所述，`addItemStream`的引用永远不会改变，所以变更检测永远不会对这个组件的子树执行。这是一个问题，因为该组件在其`ngOnInit`生命周期钩子中订阅了该流并增加了计数器。这是应用程序的状态变化，我们希望将其反映在我们的视图中，对吗？  
  
下面是我们的变更检测器树的样子（我们把所有的设置为OnPush）。当一个事件发生时，将不进行变更检测。  
  
我们如何通知Angular这个变化？我们如何告诉Angular需要对这个组件进行变更检测，即使整个树被设置为OnPush？

不用担心，Angular已经帮我们解决了。正如我们之前学到的，变更检测总是从上到下进行的。因此，我们需要的是一种方法来检测整个树的路径到发生变化的组件的变化。Angular不可能知道是哪条路径，但我们知道。

我们可以通过依赖注入来访问一个组件的`ChangeDetectorRef`，它带有一个叫做`markForCheck()`的API。这个方法做的正是我们需要的 它标记了从我们的组件到根的路径，以便在下一次变更检测运行时被检查。

让我们把它注入到我们的组件中。  

```ts
constructor(private cd: ChangeDetectorRef) {}
```

然后，告诉Angular标记从这个组件到根的路径，以便检查。

```ts
ngOnInit() {
  this.addItemStream.subscribe(() => {
    this.counter++; // application state changed
    this.cd.markForCheck(); // marks path
  })
}
```


现在，当进行变更检测时，它将简单地从上到下进行。

这多酷啊？一旦变更检测运行结束，它将恢复整个树的OnPush状态。

## 结束
希望这篇文章能让我们更清楚地了解到，使用不可变的数据结构或Observables能让我们的Angular应用更快。






