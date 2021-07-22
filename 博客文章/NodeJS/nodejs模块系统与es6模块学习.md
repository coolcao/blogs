## nodejs模块系统
在nodejs中模块使用 `exports`,`module.exports`进行变量导出。
### exports
定义一个文件 `a.js`，代码如下：
```js
const name = 'coolcao';
const say = (name) => {
  console.log(`hello, ${name}`);
}

exports.name = name;
exports.say = say;
```
在另一个文件`b.js`中使用require引入a中导出的变量：
```js
const a = require('./a');
console.log(a); // { name: 'coolcao', say: [Function: say] }
```

### module.exports
a.js
```js
const name = 'coolcao';
const say = (name) => {
  console.log(`hello, ${name}`);
}
module.exports = {
    name,
    say,
}
```
同样b.js还是如上，效果是一样的。当然写法上其实还可以这样：
```js
const name = 'coolcao';
const say = (name) => {
  console.log(`hello, ${name}`);
}

module.exports.name = name;
module.exports.say = say;
```
这种写法和直接用`exports.name = name`是一致的。

那这两种方式，有什么异同，又有什么关系呢？

nodejs将单个独立的js文件作为一个模块，`exports`或`module.exports`用来将该模块的变量导出，以便其他模块可以引用。*没有导出的变量不能被其他模块引用*。
上面例子中，a.js就是一个单独的模块，我们可以称作"模块a"，当在文件b.js中使用`const a = require('./a');`引入时，是 **将a整个模块作为一个对象导入进来** ，在调用的时候，使用`a.name`,`a.say`来调用导出的变量。

其实，exports 只是 module.exports 的一个别名，详情见[nodejs文档exports shortcut部分](https://nodejs.org/dist/latest-v9.x/docs/api/modules.html#modules_exports_shortcut)。

区别在于，module.export 可以使用`module.exports.name='coolcao'`或`module.exports={name: 'coolcao'}`两种语法导出，而 exports 只能用`exports.name='coolcao'`语法进行导出。
原因是，exports 是 module.exports 的一个别名，在nodejs里也是一个内置的变量名，如果使用 `exports = {name: 'coolcao'}`这种语法，相当于重新给`exports`赋值，而这个值却不能绑定到`module.exports`。
文档中有这么描述：*However, be aware that like any variable, if a new value is assigned to exports, it is no longer bound to module.exports.*

## es6的export, import
export命令用于规定模块的对外接口，import命令用于输入其他模块提供的功能。

es6使用 export, import 重新规范了js的模块系统。与nodejs采用的commonjs的区别在于，
* commonjs模块为对象，单个文件为单个对象，导入时要将整个对象导入。而es6模块不是对象。
* commonjs为运行时加载，而es6模块为静态加载，可做静态分析优化。
* es6模块强制使用严格模式，不管在文件头是否写`use strict`

### export
export有两种语法：
```js
// 1
export const firstName = 'Michael';
export const lastName = 'Jackson';
export const year = 1958;

//2
const firstName = 'Michael';
const lastName = 'Jackson';
const year = 1958;
export {firstName, lastName, year};
```
这两种方式是等价的。
同时，export导出的变量可以设置别名：
```js
const firstName = 'Michael';
const lastName = 'Jackson';
const year = 1958;
export {firstName as first, lastName as last, year};
```

#### 注意：
export不支持下面语法：
```js
export 1; // 不能直接导出常量

const name = 'coolcao';
export name;  // 也不支持
```
这里可能会有点让人疑惑，为啥上面可以直接导出对象形式的 `export { name }`，而直接导出上面定义好的name变量也不可以呢？
其实， `export { name }`这里`{}`并不是表示对象，而是语句块，这里被解析为语句。

export只能导出两种：
* 导出声明，即上面 export const name = 'coolcao';
* 导出语句，即上面 export { name };

### 还有一个需要注意的是
export语句输出的接口，与其对应的值是动态绑定关系，即通过该接口，可以取到模块内部实时的值。

```js
export var foo = 'bar';
setTimeout(() => foo = 'baz', 500);
```
上面代码输出变量foo，值为bar，500 毫秒之后变成baz。这一点与 CommonJS 规范完全不同。CommonJS 模块输出的是值的缓存，不存在动态更新。

### import
import可用于导入其他模块中导出的变量。

```js
// a.js
export const name = 'coolcao';

// b.js
import { name } from './a';
```
