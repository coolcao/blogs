---
title: ecmasrcipt类型转换
date: 2016-08-12 21:57:03
tags: [js,ecmasrcipt]
categories:
- 技术博客
- 原创
---

# ecma类型转换

之前写了一篇 [《js中==和===的区别.md》](https://github.com/coolcao/blogs/blob/master/js/js%E4%B8%AD%3D%3D%E5%92%8C%3D%3D%3D%E7%9A%84%E5%8C%BA%E5%88%AB.md)，其中提到了 `ToPrimitive()`，转换为原始类型的方法。这个方法中，其实也引用到了其他的类型转换，比如，`ToBoolean()`转换为Boolean类型，`ToNumber()`转换为数字类型等等，但是这些方法具体是怎么执行的，并未做说明，今天就看看ecma中对类型转换的几个方法的说明。


## ToBoolean ( argument )

|Argument Type|Result|
|:----|:----|
|Undefined|Return false.|
|Null   |Return false.|
|Boolean    |Return argument.|
|Number |Return false if argument is +0, -0, or NaN; otherwise return true.|
|String |Return false if argument is the empty String (its length is zero); otherwise return true.|
|Symbol |Return true.|
|Object |Return true.|

从表格中的说明，我们可以看出，ToBoolean()转换过程如下图：

![ToBoolean](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/ToBoolean.png)

## ToString(argument)

|Argument Type|Result|
|:----|:----|
|Undefined|Return "undefined".|
|Null   |Return "null".|
|Boolean    |1. If argument is true, return "true". <br>2. If argument is false, return "false".|
|Number |see `ToString Applied to the Number Type`|
|String |Return argument.|
|Symbol |Throw a TypeError exception.|
|Object |Apply the following steps: <br>1. Let primValue be ? ToPrimitive(argument, hint String). <br>2. Return ? ToString(primValue).|

### ToString Applied to the Number Type
The abstract operation ToString converts a Number m to String format as follows:

<ol>
    <li>If
        <var>m</var> is
        <emu-val>NaN</emu-val>, return the String <code>"NaN"</code>.</li>
    <li>If
        <var>m</var> is
        <emu-val>+0</emu-val> or
        <emu-val>-0</emu-val>, return the String <code>"0"</code>.</li>
    <li>If
        <var>m</var> is less than zero, return the String concatenation of the String <code>"-"</code> and
        <emu-xref aoid="ToString"><a href="#sec-tostring">ToString</a></emu-xref>(-
        <var>m</var>).</li>
    <li>If
        <var>m</var> is
        <emu-val>+∞</emu-val>, return the String <code>"Infinity"</code>.</li>
    <li>Otherwise, let
        <var>n</var>,
        <var>k</var>, and
        <var>s</var> be integers such that
        <var>k</var> ≥ 1, 10<sup><var>k</var>-1</sup> ≤
        <var>s</var> &lt; 10<sup><var>k</var></sup>, the Number value for
        <var>s</var> × 10<sup><var>n</var>-<var>k</var></sup> is
        <var>m</var>, and
        <var>k</var> is as small as possible. Note that
        <var>k</var> is the number of digits in the decimal representation of
        <var>s</var>, that
        <var>s</var> is not divisible by 10, and that the least significant digit of
        <var>s</var> is not necessarily uniquely determined by these criteria.</li>
    <li>If
        <var>k</var> ≤
        <var>n</var> ≤ 21, return the String consisting of the code units of the
        <var>k</var> digits of the decimal representation of
        <var>s</var> (in order, with no leading zeroes), followed by
        <var>n</var>-
        <var>k</var> occurrences of the code unit 0x0030 (DIGIT ZERO).</li>
    <li>If 0 &lt;
        <var>n</var> ≤ 21, return the String consisting of the code units of the most significant
        <var>n</var> digits of the decimal representation of
        <var>s</var>, followed by the code unit 0x002E (FULL STOP), followed by the code units of the remaining
        <var>k</var>-
        <var>n</var> digits of the decimal representation of
        <var>s</var>.</li>
    <li>If -6 &lt;
        <var>n</var> ≤ 0, return the String consisting of the code unit 0x0030 (DIGIT ZERO), followed by the code unit 0x002E (FULL STOP), followed by -
        <var>n</var> occurrences of the code unit 0x0030 (DIGIT ZERO), followed by the code units of the
        <var>k</var> digits of the decimal representation of
        <var>s</var>.</li>
    <li>Otherwise, if
        <var>k</var> = 1, return the String consisting of the code unit of the single digit of
        <var>s</var>, followed by code unit 0x0065 (LATIN SMALL LETTER E), followed by the code unit 0x002B (PLUS SIGN) or the code unit 0x002D (HYPHEN-MINUS) according to whether
        <var>n</var>-1 is positive or negative, followed by the code units of the decimal representation of the integer
        <emu-xref aoid="abs"><a href="#sec-algorithm-conventions">abs</a></emu-xref>(
        <var>n</var>-1) (with no leading zeroes).</li>
    <li>Return the String consisting of the code units of the most significant digit of the decimal representation of
        <var>s</var>, followed by code unit 0x002E (FULL STOP), followed by the code units of the remaining
        <var>k</var>-1 digits of the decimal representation of
        <var>s</var>, followed by code unit 0x0065 (LATIN SMALL LETTER E), followed by code unit 0x002B (PLUS SIGN) or the code unit 0x002D (HYPHEN-MINUS) according to whether
        <var>n</var>-1 is positive or negative, followed by the code units of the decimal representation of the integer
        <emu-xref aoid="abs"><a href="#sec-algorithm-conventions">abs</a></emu-xref>(
        <var>n</var>-1) (with no leading zeroes).
    </li>
</ol>


ToString()方法比ToBoolean()稍微复杂点，对于`Undefined`,`Null`,`String`,`Symbol`,`Boolean`这几种类型，转换比较简单，可以直接看说明。
而对于`Object`类型的转换，执行两步，先将参数通过方法`ToPrimitive(argument, hint String)`转换成原始类型的值 primValue，再将primValue调用`ToString(primValue)`转换成字符串。
但是，对于Number类型，转换就有点复杂了，一点一点来看：
调用ToString()方法将Number类型的值m转换成字符串的步骤：


<ol>
    <li>如果
        <var>m</var> 是
        <emu-val>NaN</emu-val>, 返回字符串 <code>"NaN"</code>.</li>
    <li>如果
        <var>m</var> 是
        <emu-val>+0</emu-val> 或者
        <emu-val>-0</emu-val>, 返回字符串 <code>"0"</code>.</li>
    <li>如果
        <var>m</var> 比0小, 返回由 <code>"-"</code> 和
        <emu-xref aoid="ToString"><a href="#sec-tostring">ToString</a></emu-xref>(-
        <var>m</var>)组成的字符串.</li>
    <li>如果
        <var>m</var> 是
        <emu-val>+∞</emu-val>, 返回字符串 <code>"Infinity"</code>.</li>
    <li>否则, 令
        <var>n</var>,
        <var>k</var>,
        <var>s</var> 为整数，并且
        <var>k</var> ≥ 1, 10<sup><var>k</var>-1</sup> ≤
        <var>s</var> &lt; 10<sup><var>k</var></sup>,
        <var>s</var> × 10<sup><var>n</var>-<var>k</var></sup> 的值是
        <var>m</var>, 并且
        <var>k</var> 越小越好。 请注意，
        <var>k</var> 是
        <var>s</var>的十进制表示中数字的位数, 并且
        <var>s</var> 不能被10整除, 并且
        <var>s</var> 的至少要求的有效数字位数不一定要被这些标 准唯一确定.</li>
    <li>如果
        <var>k</var> ≤
        <var>n</var> ≤ 21, 返回由
        <var>k</var> 位数字组成的
        <var>s</var> 的十进制表示(有序的, 开头没有0), 后面跟着
        <var>n</var>-
        <var>k</var> 位字符 0x0030 (数字0的字符表示).</li>
    <li>如果 0 &lt;
        <var>n</var> ≤ 21, 返回由
        <var>n</var> 位数字表示的
        <var>s</var>的十进制数字, 后面跟着 0x002E (小数点 . ), 再后面跟着
        <var>k</var>-
        <var>n</var> 位
        <var>s</var>的剩余部分组成的字符串.</li>
    <li>如果 -6 &lt;
        <var>n</var> ≤ 0, 返回由 0x0030 (数字0), 后面跟着 0x002E (小数点 . ), 再后面跟着 -
        0x0030 (数字 0 ) <var>n</var>   次出现, 再后面跟着
        <var>k</var> 位数字表示的
        <var>s</var>的十进制表示所组成的字符串.</li>
    <li>否则, 如果
        <var>k</var> = 1, 返回由单个数字
        <var>s</var>, 后面跟着字符 0x0065 (小写的拉丁字母e), 再后面跟着字符 0x002B (加号 + ) or 或者字符 0x002D (减号) ，这取决于
        <var>n</var>-1 是正数还是负数, 最后再跟着
        <emu-xref aoid="abs"><a href="#sec-algorithm-conventions">abs</a></emu-xref>(
        <var>n</var>-1)十进制 (前面没有0)所组成的字符串.</li>
    <li>返回由
        <var>s</var>在十进制表示中的、最多的有效数字组成的字符串, 后面跟随字符 0x002E (小数点 . ), 再后面是余下的是
        <var>k</var>-1 位数字的
        <var>s</var>的十进制表示, 再往 后是字符 0x0065 (小写拉丁字母e), 再后面跟着字符 0x002B (加号 + ) 或者字符 0x002D (减号 - )， 这取决于
        <var>n</var>-1 是正数还是负数, 最后面跟着整数
        <emu-xref aoid="abs"><a href="#sec-algorithm-conventions">abs</a></emu-xref>(
        <var>n</var>-1) (前置没有0)的十进制表示所组成的字符串.
    </li>
</olo>


看完ecma中的这个描述，是不是有点迷茫，不就是个将数字转换成字符串嘛，这么麻烦，这么多规则？
好吧，其实也不麻烦，ecma毕竟是规范，它要全面考虑各种情况，将各种情况进行说明，不是吗？
我们来找几个例子对照着说明看看差不多就明白了。

#### 实例1.将 `3500` 转换成字符串
首先，将整数`3500`拆解，这里 s=35,k=1,n=3,即 3500=35x10^(3-1)，这些数字怎么来的呢，看第一步的说明，s不能被10整除，还是整数，很容易得出35,k要足够小，那么，足够小的整数，1啦，n就是3.要转换成字符串，继续看，k<=n<=21，被第一个条件命中，返回 35后面跟着(3-1)位0组成的字符串，即 `"3500"`


#### 实例2.将 小数 `3.56` 转换成字符串
将 `3.56`拆解，这里的 s=356,k是s的位数，为3,那么n的值也就是 -2+3=1,即 s=356,k=3,n=1.这里进入条件  0 < n ≤ 21 ,将返回的字符串表示方法为：先是s的十进制表示的前1位，后面跟小数点，然后是 k-n=2位剩余的s的数位组成的字符串。即 `"3.56"`

其余的实例就不看了，无非就是小数，有几个0的问题，大家可以自行研究一下，0.0000032,0.000000000000032这种小数转为字符串的表示方法。剩余的部分说明的是当小数位数过多时，用e去表示，如此而已。


## ToNumber
|Argument Type|Result|
|----|----|
|Undefined|Return NaN|
|Null|Return +0.|
|Boolean|Return 1 if argument is true. Return +0 if argument is false.|
|Number|Return argument (no conversion).|
|String|See grammar and conversion algorithm below.|
|Symbol|Throw a TypeError exception.|
|Object|Apply the following steps:<br>Let primValue be ? ToPrimitive(argument, hint Number).<br>Return ? ToNumber(primValue).|

从上面这个表中，我们可以看出，对于Undefined,Null,Boolean,Number,Symbol等类型的转换说明，比较简洁明了，可以简单看看记住就可以了。
对于Object类型，规则如下：
* 另 primValue为 ToPrimitive(argument,hint Number).
* 返回 ToNumber(primValue)
也就是说，对于Object类型的值，要想转换成Number类型，首先，转换成原始类型的值，然后在将这个原始类型的值转换为Number类型。

下面我们来看一个例子，关于一个Object转换成Nunber类型的值。

```
> var obj = {name:'coolcao',age:23};
undefined
> Number(obj)
NaN
>
```
最后输出的是，NaN，怎么来的呢，跟着说明一步一步的来看：
首先，将obj转换成原始类型的值。ToPrimitive(obj,hint:Number),这里转换成原始类型的值时，hint默认使用的Number，则调用对象的 valueOf()方法，由于我们没有重写valueOf()方法，对象默认的valueOf()方法返回的是其本身，则继续使用hint为String调用toString()方法，toString()方法返回的是'[object Object]'，然后再对字符串'[object Object]'调用ToNumber()进行转换，很显然，返回值为NaN

这里，如果我们重写一下valueOf()方法呢？
```
obj.valueOf = function(){
    return this.age;
}
```
如果重写了valueOf()方法，在第一步，转换成原始类型的值时，valueOf()返回的是age的值23，然后再对23进行转换，其最终结果还是23.
对于Object类型的值，先将其转换为原始类型的值，不管调用的是valueOf()方法还是toString()方法，如果不能转换成原始类型的值，则直接抛出不能转换错误。然后再对这个原始类型的值转换成Number类型，转换规则如上表所示。

对于，字符串类型的值转换成Number类型，上表中没有具体指明规则，在ecma规范中，对这一块的规范说明还是蛮复杂的，规范嘛，考虑到各种的字符的可能性，因此说明会复杂很多。如果有兴趣可以看看ecma规范中的说明。
