---
title: jser做了几道简单而有趣的算法题
date: 2016-11-24 00:31:05
tags: [算法]
categories:
- 技术博客
- 原创
---

作为js开发人员，既然被称作工程师，那么，还是有必要对算法有所了解的，时不时的找几道简单的算法题目练练脑，训练下逻辑，也是对编程基础的巩固吧。
这几天看到几道有趣但很简单的算法题，和大家分享下。

<!--more-->

## 判断2的乘方

> *实现一个方法，判断一个正整数是否是2的乘方（比如16是2的4次方，返回true,18不是2的乘方，返回false），要求性能尽可能地高*

题目初看，很简单的啊，我们只需要循环除2，判断最后余数是否为0即可。是的，有过上小学经历的人，应该都能想到这个办法。可是呢，这是最优解么？
当然不是，看到一个题目，脑中首先想到的方法，肯定不是最优解。
仔细分析一下题目，**判断2的次方**，2在计算机中，是一个很特别的数字，因为计算机是按照二进制做运算的。
那么，是不是可以按照二进制的思路来解决？
我们先看看2的乘方在二进制表示上有什么特别之处吧：
```
1       2的0次方
10      2的1次方
100     2的2次方
1000    2的3次方
10000   2的4次方
...
```
2的n次方在二进制表示上，就是1后面有n个0。
既然发现这个小“规律”，那么如何解这道题目呢？
既然考虑到使用二进制计算，那么肯定要用的就是位运算。按位与`&`，按位或`|`,按位非`~`,按位异或`^`。
在二进制中，一定要对2的n次方，以及2的n次方减1特别敏感。
在2的n次方基础上减去1，我们就得到了“错位”的数：
```
0        2的0次方减1
1        2的1次方减1
11       2的2次方减1
111      2的3次方减1
1111     2的4次方减1
...
```
好了，在“错位”的两个数进行位运算，会有各种想不到的结果。
进行异或运算`^`，我们得到的是从高位到低位一律为1的二进制数。
进行与运算`&`,我们得到的是0。
好了，我们可以根据这两条规律，编写相应的方法了。

```js
//使用异或运算
const is2power = function is2bit(num) {
    return (num ^ (num - 1)) === ((num << 1) - 1);
}
//使用与运算
const is2power = function is2bit(num) {
    return (num & (num - 1) ) === 0;
}
```
当然，从效率上讲，当然是第二种使用与运算更高，因为使用异或运算还要进行移位运算。

## 计算二进制中1的个数

> *实现一个方法，计算出一个正整数转换成二进制后的数字'1'的个数，比如，3的二进制为11,1的个数为2。要求性能尽可能高。*

首先想到的，就是转换成二进制，然后循环计算：

```js
const countOneOfBit = function countOneOfBit(num){
    let bs = num.toString(2);

    let count = Array.prototype.reduce.call(bs,(pre,current)=>{
        if((current >>> 0) === 1){
            pre ++;
        }
        return pre;
    },0)
    return count;
}
```
当然，这里还是关于二进制的题目，我们再考虑考虑如何利用二进制位运算解决这个题目。
在二进制中，如何判断一位是1，将此位和1做与`&`运算，如果结果是1，那就是1啦。
那这个该如何解决这个题目呢？

```js
const countOneOfBit = function countOneOfBit(num) {
    let count = 0;
    while (num > 0) {
        if(num & 1 === 1){
            count ++;
        }
        num = num >> 1;
    }
    return count;
}
```
这里我将循环和二进制中每位进行&运算，如果遇到1，计数加1。
上面这个还是存在浪费的情况，因为对每位都做了运算，不管是0还是1。
按道理来讲，我们只需要拿出1的位，然后累加就好了。
怎么讲？如何逐个拿出1呢？
我们来看一下一个二进制：
```
10010
```
如何拿到最右边的1呢（倒数第二位），我们将这个数减1，得到：
```
10001
```
我们可以看到，从最右边的1那一位开始，到最高位所有位都没发生变化，只有后面的发生了变化，由于进位关系，原先1的那一位变成0，我们将两个数进行与`&`运算得到：
```
10000
```
这样就消除并拿到最右边的一位。因此，我们可以继续循环，直到num变为0,代码如下：

```js
const countOneOfBit = function countOneOfBit(num){
    let count = 0;
    for (count =0; num; ++c){
        num &= (num -1) ; // 清除最低位的1
    }
    return count ;
}
```
这种方法，循环的次数会比上面一种方法要少，因为这里只循环了位是1的次数。


## 计算数组中元素乘积

> *给定一个数组，数组中n个正整数（n>1），实现一个方法，输入这个数组，输出一个数组，数组中每个元素是除了当前元素其他元素的乘积。*
> **要求：**
> * 不要用 **除法** 运算
> * 复杂度为O(n)
> 例如，输入[1,2,3,4]，输出[24,12,8,6]

如果先不看要求，我们能想到的是，把数组中元素逐个相乘，得到结果除以当前元素，此时复杂度为O(n)。
但是不能用除法，好吧。这个方法不合要求。
那如果不用除法，再想到的是，遍历数组，将除去当前元素剩下所有元素做乘，但是此时复杂度为O(n2)。
这个方式，在复杂度上不合要求。

那该怎么办呢？

我们可以进行两遍循环（注意，不是两层），第一遍循环，求得当前元素左边的元素的乘积，第二遍循环，求得当前元素右边的乘积，然后将左右两边的乘积相乘，即可得到答案。

```js
const productOfArrayExceptSelf = function productOfArrayExceptSelf(array){
    if(!Array.isArray(array)){
        throw new Error('参数错误，只能为数组');
    }
    let length = array.length;
    let pre_of_product = new Array(length);
    let back_of_product = new Array(length);
    pre_of_product[0] = 1;
    back_of_product[length - 1] = 1;

    for(let i=1;i<length;i++){
        pre_of_product[i] = pre_of_product[i - 1] * array[i - 1];
    }
    for (let i = length - 2; i >= 0; i--) {
        back_of_product[i] = back_of_product[i + 1] * array[i + 1];
    }
    let result = [];
    for(let i=0;i<length;i++){
        result.push(pre_of_product[i] * back_of_product[i]);
    }
    return result;
}
```
注意，这里总共使用了三遍循环，第一，二遍是求左右两边的乘积，第三遍是将左右做乘，求最终的结果。
虽然达到了要求，没有使用除法，而且也满足O(n)，但是感觉，是不是还可以更高效呢？

```js
const productOfArrayExceptSelf = function productOfArrayExceptSelf(array) {
    if(!Array.isArray(array)){
        throw new Error('参数错误，只能为数组');
    }
    let length = array.length;
    let res = [1];

    for(let i=1;i<length;i++){
        res[i] = res[i - 1] * array[i - 1];
    }
    let rightValue = 1;
    for(let i=length - 1;i>=0;i--){
        res[i] = res[i] * rightValue;
        rightValue = array[i] * rightValue;
    }
    return res;
}
```

这段代码，看上去好像更高达上了是吧，其实做法和上面一个思路是一样的，只不过是将第二个循环和第三个循环并作一个循环了。


## 找出出现奇数次的数
> *一个无序数组里由若干个正整数，范围从1到100，其中99个整数都出现了偶数次，只有一个整数出现了奇数次，如何找出这个出现了奇数次的整数？例如，[1,2,2,3,3,3,3,4,4]其中只有1出现了奇数次，1次*

### 方案1:
使用HashMap，遍历整个数组，以数组元素作为键，元素出现的次数作为值，每出现一次，值加1，最后遍历HashMap里哪个键的值是奇数即可找到出现奇数次的元素。

### 方案2:
分析一下题目，所有元素都出现了偶数次，只有一个元素出现了奇数次。联想一下位运算，异或`^`运算，相同值异或得0，不同值异或得1。好了，那么我们可以遍历数组，将数组中的元素依次做异或运算，最后得出的值，一定是出现奇数次的值。
因为，出现偶数次的值在异或运算过程中都抵消了。
```js
const findOddTimesNum = function findOddTimesNum(array) {
    if(!Array.isArray(array)){
        throw new Error('参数必须为数组');
    }
    return array.reduce(function(pre,current){
        pre = pre ^ current;
        return pre;
    },array.shift());
}
```



## 后记
算法方面，作为jser的我也是小白，上面几个题目，可能会有更优的解法，欢迎指正批评，大家共同进步。
日后，也要鞭策自己，多多训练算法题目。
当然，这篇文章以后遇到更有意思的题目，也会持续更新。
