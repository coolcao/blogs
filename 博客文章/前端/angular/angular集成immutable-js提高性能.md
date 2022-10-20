---
title: angular集成immutable-js提高性能.md
date: 2021-11-30 08:48:48
tags: [Angular]
categories:
- 技术博客
- 原创
---

Angular有两种变更检测策略：`Default`和`OnPush`。在一些中小型项目中，直接使用默认的`Default`策略即可，但如果在一些大型项目中，数据量比较大，而且变动比较频繁，一般会使用`OnPush`策略来优化性能。

`OnPush`策略提供了跳过对这个组件和它的所有子组件的不必要检查的可能性，以提高性能。
其中有一项很重要的就是**输入的引用发生了变化**。
JavaScript中的所有东西都是通过引用传递的，但所有的基元(基础类型数据)都是不可改变的，它们的字面表述都指向同一个基元实例/引用。修改对象属性或数组条目不会创建一个新的引用，因此不会触发OnPush组件的变更检测。要触发变更检测器，你需要传递一个新的对象或数组引用来代替。
这就要求输入使用`不可变`对象，每次对象属性发生变化都产生一个新对象引用，以便能够触发变更检测。

## 不可变对象

其实，要使用不可变对象，最简单的方式就是，每次当对象属性发生变化时，都重新生成一个新对象即可。像下面这样：

```typescript
let user = {
    name: 'coolcao',
    age: 18,
}

// 不要这样
user.name = 'jack';

// 这样
user = {...user, name: 'jack'};
```

相当于每次更改user对象都会重新生成一个新的对象。这样做的缺点就是，**如果应用比较大，数据量比较大，会消耗更多的内存，而且降低性能**。

## immutable-js

[Immutable-js](https://immutable-js.com/ "Immutable-js")是一个高性能的`不可变`数据第三方库，是由Facebook开发维护的。

`immutable-js`提供了很多结构，比如`Record`,`Map`, `List`，`Set`, `Stack`等，一般我们常用的便是`Map`和`List`， `Map`用以表示对象， `List`用以表示数组，有多个对象。

在使用`immutable-js`时，可以直接使用`Map`模拟对象，像下面这样：

```typescript
const immutable = require('immutable');
const Map = immutable.Map;

let user = Map({
    name: 'coolcao',
    age: 18,
});

user.set('name', 'jack');
console.log(user.get('name'));  // coolcao

user = user.set('name', 'jack');
console.log(user.get('name'));  // jack
```

也可以使用 `Record` 将对象包装一层，如下：
```ts
import { Record } from 'immutable';

class User {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }
}

// UserRecord是一个工厂函数，用于创建User类型的Record
const UserRecord = Record({ name: '', age: 0 }, 'UserRecord');

// 普通的User对象
const user = new User('coolcao', 18);

// 将user对象包装成Record，uRecord是不可变的
const uRecord = UserRecord(user);
console.log(uRecord.name);
```

使用 `Record` 的好处是，编辑器能通过类型分析，自动进行代码补全提示，像下面这样：
![Pasted image 20220930105440](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1664508083_20220930112035358_1813379688.png)
![Pasted image 20220930105906](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1664508085_20220930112050291_479457721.png)

而直接使用 `Map` 却无法做到属性的自动补全提示。所以建议对于单个对象的提示，使用 `Record`。

## Angular中使用immutable

如上所述，Angular中我们可以使用 `Record` 对对象包装一层，在组件中，使用不可变的 `Record` ， 监听事件里改的也是 `Record` ：

```ts
// 定义User类
export class User {
    name: string;
    age: number;
    constructor(name: string, age: number) {
        this.name = name;
        this.age = age;
    }
}

// 定义UserRecord
import { Record } from 'immutable';
import { User } from '../models/User';


const UserRecord = Record<User>({ name: '', age: 0 }, 'UserRecord');

export {
    UserRecord
}

// 组件
@Component({
  selector: 'cc-root',
  template: `
  <h1>姓名：{{userRecord.name}}</h1>
  <h2>年龄： {{userRecord.age}}</h2>
  <button (click)="changeAge()">涨一岁</button>
  `,
  styleUrls: ['./app.component.css']
})
export class AppComponent implements OnInit {
  userRecord = UserRecord({ name: 'coolcao', age: 23 });

  constructor() {
  }

  ngOnInit(): void {
  }

  changeAge() {
    this.userRecord = this.userRecord.set('age', this.userRecord.age + 1);
  }
}

```

这样不管是在组件中，还是在模板中，都可以进行代码提示补全：
![Pasted image 20220930111737](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1664508086_20220930112107509_275958753.png)


![Pasted image 20220930111808](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1664508087_20220930112119690_2122500556.png)
