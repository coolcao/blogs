---
title: 深入理解Angular2的单向数据流（翻译）
date: 2022-09-29 10:11:23
tags: [Angular, 单向数据流]
categories:
  - 技术博客
  - 翻译
---

> 原文地址：[Angular - What is Unidirectional Data Flow? Learn How the Angular Development Mode Works, why it's important to use it and how to Troubleshoot it](https://blog.angular-university.io/angular-2-what-is-unidirectional-data-flow-development-mode/)

Web 框架的一个重要的特征就是 **单向数据流**。让我们来谈谈这个术语是什么意思，以及它在 Angular 中的对应关系。

与此相关，我们也来了解一下 Angular 的开发模式，为什么使用它很重要，如果需要的话，我们可以如何排除故障。让我们首先从单向数据流这个术语的数据部分开始。

<!--more-->

## 我们指的数据是什么

在单向数据流这个术语中，我们所指的数据只是我们应用程序的数据。更具体地说，它是被传递到视图中在屏幕上呈现的视图模型数据。

这里有一个例子，我们可以看到它只是一个普通的 JavaScript 对象。

```ts
export interface Course {
  id: number;
  description: string;
}

const course: Course = {
  id: 1,
  description: "Angular For Beginners",
};
```

如果我们想在屏幕上显示课程对象，那么我们就需要通过一个组件定义一个模板，模板将定义数据的显示方式。

下面是一个 Angular 模板的例子:

```ts
import { Component } from "@angular/core";
import { Course } from "./course";

@Component({
  selector: "app-root",
  template: `
    <div class="course">
      <span class="description">{{ course.description }}</span>
    </div>
  `,
})
export class AppComponent {
  course: Course = {
    id: 1,
    description: "Angular For Beginners",
  };
}
```

所以我们可以看到在 Angular 中，模板可以简单地只是一个内联字符串，但也可以把它放在一个外部文件中。

你会注意到，在 Angular 中，只要模板出了问题，我们就会得到有用的错误信息。所以很明显，Angular 不是简单地把模板当作一个字符串来处理。那么它是如何工作的呢？

## Angular 内部是如何使用模板的？

Angular 是如何使用模板来显示屏幕的数据的，在引擎盖下发生了什么？

这其实很简单。Angular 正在获取数据，并将其应用于一个函数。该函数的输出是与该 HTML 模板相对应的 DOM 数据结构。

因此，我们最初可能会认为，发生的事情是如下:

- Angular 获取数据，并通过替换一些变量在模板的基础上创建实际的 HTML
- 然后把这个 HTML 传给浏览器
- 浏览器解析 HTML 并生成一个 DOM 数据结构
- 浏览器的渲染引擎在屏幕上渲染该视图

但实际情况并非如此，而是发生了与此相近的事情，但有一个重要区别。

> Angular 不是将 HTML 生成为字符串，而是直接生成 DOM 数据结构。

例如，在我们的案例中，Angular 会通过一个组件渲染器直接从数据中生成 DOM 数据结构。

如果该应用程序是一个本地移动应用程序或服务器端应用程序，那么呈现器的输出将是其他东西。但在这种情况下，组件渲染器的输出是代表组件视图的 DOM 树。

这个 DOM 组件渲染器是通过类似于以下代码的东西来定义的:

```js
View_AppComponent_0.prototype.createInternal = function (rootSelector) {
  var self = this;

  var parentRenderNode = self.renderer.createViewRoot(self.parentElement);

  self._text_0 = self.renderer.createText(
    parentRenderNode,
    "\n",
    self.debug(0, 0, 0)
  );

  self._el_1 = jit_createRenderElement5(
    self.renderer,
    parentRenderNode,
    "div",
    new jit_InlineArray26(2, "class", "course"),
    self.debug(1, 1, 0)
  );

  self._text_2 = self.renderer.createText(
    self._el_1,
    "\n\n    ",
    self.debug(2, 1, 20)
  );

  self._el_3 = jit_createRenderElement5(
    self.renderer,
    self._el_1,
    "span",
    new jit_InlineArray26(2, "class", "description"),
    self.debug(3, 3, 4)
  );

  self._text_4 = self.renderer.createText(self._el_3, "", self.debug(4, 3, 30));

  self._text_5 = self.renderer.createText(
    self._el_1,
    "\n\n",
    self.debug(5, 3, 59)
  );

  self._text_6 = self.renderer.createText(
    parentRenderNode,
    "\n",
    self.debug(6, 5, 6)
  );

  self.init(
    null,
    self.renderer.directRenderer
      ? null
      : [
          self._text_0,
          self._el_1,
          self._text_2,
          self._el_3,
          self._text_4,
          self._text_5,
          self._text_6,
        ],
    null
  );
  return null;
};
```

这是初始化我们上面展示的 AppComponent 的渲染器的函数。变量的名字看起来很奇怪，但我们可以从这里看一下大体的逻辑：

- 我们创建一个 DIV
- 我们为其添加 CSS 类 `counrse`
- 我们在它里面添加一个带有 CSS 类 `description` 的 span
- 在这个 span 内，我们添加一些文本

所以，我们可以看到，这个函数确实对应于我们上面刚刚定义的模板。

## 我在哪里可以为我的组件找到这个函数？

如果你想要看到你某个组件的类似的函数，只需进入 Chrome 开发工具的 `Sources` 选项卡，按 `Ctrl+O` 键，输入你的组件名称。

你应该能找到一个与你的组件名称相同的文件夹，里面有一个名为 `component.factory.js` 的文件。

这时你可能会想，Angular 是从哪里得到的这个函数的？这个函数只是 Angular 编译器根据组件的模板生成的。

## 这个代码是什么时候生成的？

根据我们运行 Angular 的方式，这些代码将在不同的时间点上生成。如果我们在 JIT 模式下使用 Angular，那么这些函数将在浏览器中的应用程序启动时生成。

这意味着我们需要将编译器运送到浏览器，而且我们需要等待这个过程完成后才能在屏幕上显示数据。

我们也可能在 AOT 模式下运行 Angular，或者说 AOT。这意味着这个函数是在构建时生成的，而不是在应用程序启动时生成的。这也意味着，由于两个原因，应用程序的速度会更快。

- 我们不需要向浏览器发送编译器，因为模板编译是在内置时完成的
- 我们不需要在启动时编译模板，所以应用程序的启动速度会更快

那么，这与单向数据流有什么关系？

## 我们所指的数据流是什么？

现在我们知道 Angular 它是如何将数据转换为 DOM 数据结构的，让我们看看整个过程是如何进行的。

举例来说。如果我们触发了一个事件处理程序，比如说按钮点击处理程序，我们就可以在组件树的任何一层随意修改应用程序的数据。但是一旦点击处理程序的代码返回，控制权就被交还给 Angular 框架。

- Angular 将从我们应用程序的根组件开始，遍历整个组件树。
- 对于每个组件，Angular 将运行与该组件相关的变更检测机制，并确定该组件是否需要重新渲染。
- 如果该组件需要重新渲染，Angular 将运行其 DOM 生成函数，生成一个新的 DOM 数据结构，与新版本的组件视图相对应
- 这个过程从组件树的根部开始，一直到应用程序的叶子组件。
- 在这个过程中，每个组件的几个生命周期方法被调用，例如`ngAfterViewChecked`。

**因此，我们所说的单向数据流是指在组件树从上到下的渲染过程中，从组件类进入由渲染过程产生的输出 DOM 数据结构的应用数据流**。

## 数据流的单向性是什么意思？

在渲染过程中，模板表达式被评估，生命周期钩子在整个组件树上被调用。这意味着我们编写的代码在这个过程中被调用。

我们想确保在我们将数据转化为视图的过程中，视图生成过程本身不会进一步修改数据。

数据从组件类流向代表它们的 DOM 数据结构，但生成这些 DOM 数据结构的行为本身并不会导致数据的进一步修改。

## 为什么要避免这种情况？

因为这将导致以下情况：

- 多次重复这一过程，直到数据稳定下来。
- 或者让数据和视图处于不一致的状态，即渲染过程结束后的视图并不反映数据的实际状态

这两种结果都是要避免的，让我们看看这是为什么。

- 让数据和视图处于不一致的状态，会导致应用程序难以推理。
- 多次重复这个过程，等待数据稳定下来，会导致性能问题

这就是为什么在渲染过程中，数据只从组件类流向视图，而不是反过来--这解释了单向数据流这个术语的重要性。

## 为什么单向的数据流很重要？

我们可以看到知道为什么这个属性很重要：首先是因为它有助于从渲染过程中获得巨大的性能。

但最重要的是，因为它确保了应用程序的简单推理：它确保了每当我们的事件处理程序返回和框架接管渲染结果时，没有任何不可预测的事情发生。

使用 Angular 意味着我们有内置的框架保护，可以防止一大类的 bug，否则就很难排除故障：数据与视图不一致的 bug。

## 让我们做个演示-如何打破单向数据流

例如，以下组件将打破单向数据流规则--尝试发现原因。

```ts
import { Component } from "@angular/core";
import { Course } from "./course";

@Component({
  selector: "app-root",
  template: `
    <div class="course">
      <span class="description">{{ description }}</span>
    </div>
  `,
})
export class AppComponent {
  course: Course = {
    id: 1,
    description: "Angular For Beginners",
  };

  get description() {
    return this.course.description + Math.random();
  }
}
```

你发现问题所在了么？
问题是，我们有一个模板表达式 `description` ，这是一个 TypeScript getter，由于使用了 `Math.random()` ，所以每次调用它时都会返回不同的值。

这只是一个简单的例子，但在一般情况下，在应该只读的操作里面做修改操作，很容易导致同样的情况。

如果我们运行这个程序，我们将得到以下错误:

```
EXCEPTION: Error in ./AppComponent class AppComponent - inline template:2:34 caused by: Expression has changed after it was checked. Previous value: 'Angular For Beginners0.4769352143577472'. Current value: 'Angular For Beginners0.8202702497824956'.
```

因此，正如你所看到的，Angular 开发模式阻止了我们编写一个很难推理的程序--这样的程序在运行时就会有各种不一致的地方。

## 打破单向数据流的另一种常见方式

另一种经常发生这类问题的情况是使用某些生命周期钩子时。让我们在这里看到另一个同样打破单向数据流的应用程序的例子。

```ts
import { Component, AfterViewChecked } from "@angular/core";
import { Course } from "./course";

@Component({
  selector: "app-root",
  template: `
    <div class="course">
      <span class="description">{{ course.description }}</span>
    </div>
  `,
})
export class AppComponent implements AfterViewChecked {
  course: Course = {
    id: 1,
    description: "Angular For Beginners",
  };

  ngAfterViewChecked() {
    this.course.description += Math.random();
  }
}
```

这个应用程序将触发一个与上面相同的错误。这里发生的情况是，`ngAfterViewChecked`方法是在视图已经生成后被调用的，所以如果我们在这里进一步修改数据，我们将陷入同样的 "视图在渲染时更新数据 "的情况。

那么 Angular 是如何在我们的应用程序中发现并防止这个问题的呢？

## Angular 开发模式

Angular 检测到这个问题要感谢它的开发模式。开发模式的工作方式如下。Angular 会第二次运行整个渲染过程，以确保第一次的渲染过程不会对应用程序的数据造成进一步的改变。

如果 angular 发现任何模板表达式在第一次和第二次渲染过程中发生了变化，它就会抛出一个与上面类似的错误。

## 我们如何才能避免这种错误

这些类型的情况会比较少，但如果你遇到类似的错误信息，这里有一个潜在的解决方案。

我们总是可以通过使用 setTimeout 将数据修改推迟到下一个 JavaScript VM 轮回，而不会有任何延迟。

```ts
ngAfterViewChecked() {

	setTimeout(() => {
		this.course.description += Math.random();
	});

}
```

这将安排 setTimeout 里面的函数在当前的 JavaScript 轮回结束后执行。一旦 setTimeout 回调被执行，控制权被交还给框架，渲染过程将再次发生。

这时，任何进一步的数据变化都会反映在屏幕上--但这次不会抛出错误。

使用 setTimeout 并不总是解决这类问题的方法，例如我们可以尝试重构我们的代码，以不同的方式实现相同的功能，如果可能的话，可以在给定的生命周期方法之外。

## 总结

通过这最后一个例子，我们可以看到为什么我们在开发过程中一直使用 Angular 开发模式，而只在生产中关闭它，这确实有帮助。

这是因为开发模式可以帮助我们建立一个易于推理和维护的应用程序，其中数据总是以透明的方式被框架保持与视图同步--而且框架有力地保证我们不会遇到数据与视图不一致的错误。
