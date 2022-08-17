---
title: angular集成immutable-js提高性能.md
date: 2021-11-30 08:48:48
tags: []
categories:
- 技术博客
- 原创
---

angular有两种变更检测策略：`Default`和`OnPush`。在一些中小型项目中，直接使用默认的`Default`策略即可，但如果在一些大型项目中，数据量比较大，而且变动比较频繁，一般会使用`OnPush`策略来优化性能。

<!-- more -->
`OnPush`策略提供了跳过对这个组件和它的所有子组件的不必要检查的可能性，以提高性能。
其中有一项很重要的就是**输入的引用发生了变化**。
JavaScript中的所有东西都是通过引用传递的，但所有的基元都是不可改变的，它们的字面表述都指向同一个基元实例/引用。修改对象属性或数组条目不会创建一个新的引用，因此不会触发OnPush组件的变更检测。要触发变更检测器，你需要传递一个新的对象或数组引用来代替。
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
[Immutable-js](https://immutable-js.com/)是一个高性能的`不可变`数据第三方库，是由Facebook开发维护的。
`immutable-js`提供了很多结构，比如`Map`, `List`，`Set`, `Stack`等，一般我们常用的便是`Map`和`List`， `Map`用以表示对象， `List`用以表示数组，有多个对象。

在使用`immutable-js`时，可以直接使用`Map`模拟对象即可，像下面这样：

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

但上面这种方式，对象的属性，比如上面的name和age都是无类型的，即使用typescript也是无类型的。这对开发速度会带来一定的阻碍，因为没有类型的辅助，在使用像VSCode或webstorm这种IDE时无法做到自动提示。

这里我们可以封装一下，定义一个实体类，将`immutable`中的Map作为底层数据即可。

```typescript
import {Map, List} from 'immutable';
export interface IUser {
	id: number;
	name: string;
	age: number;
}

export class User {
	private _data: Map<string, any> = Map({});

	constructor() {
		this._data = Map();
	}

	static from(iuser: IUser): User {
		const user = new User();
		user._data = user._data.set('id', iuser.id);
		user._data = user._data.set('name', iuser.name);
		user._data = user._data.set('age', iuser.age);
		return user;
	}

	public toJSObject(): User {
		return this._data.toObject() as User;
	}

	public toString(): string {
		return `User(id=${this.getId()}, name=${this.getName()}, age=${this.getAge()})`;
	}

	public getId(): number {
		return this._data.get('id');
	}
	public setId(value: number){
		this._data = this._data.set('id', value);
		return this.clone();
	}
	public getName(): string {
		return this._data.get('name');
	}
	public setName(value: string): User {
		this._data = this._data.set('name', value);
		return this.clone();
	}
	public getAge(): number {
		return this._data.get('age');
	}
	public setAge(value: number): User {
		this._data = this._data.set('age', value);
		return this.clone();
	}

	private clone(): User {
		const user = new User();
		user._data = this._data;
		return user;
	}

}
```

这样，当我们使用`User`时，直接像下面方式使用：

```typescirpt
let user = User.from({id: 1, name: 'coolcao', age: 18});
// 修改user.name并返回新的user引用
user = user.setName('jack');
```

