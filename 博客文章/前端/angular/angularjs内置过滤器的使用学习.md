---
title: angularjs内置过滤器的使用学习
date: 2015-02-05 18:48:36
tags: [angular,angularjs,前端]
categories:
- 技术博客
- 原创
---
{: id="20210312121354-m3ex676"}

在angular中内置了几个常用的filter,可以简化我们的操作。
过滤器使用 '|' 符号，概念有点类似于Linux中的管道。
{: id="20210312121354-5uo1c4i"}

<!-- more -->

### filter （过滤）
{: id="20210312121354-9z4hjh7"}

filter可以根据条件过滤数据，例子：
{: id="20210312121354-rnmkhnm"}

```
{{[{name:'coolcao',age:23},{name:'lily',age:20},{name:'tom',age:22}] | filter:'coolcao'}}
```
{: id="20210312121354-p70yjtq"}

结果：
{: id="20210312121354-qdu868a"}

```
[{"name":"coolcao","age":23}]
```
{: id="20210312121354-2q2p2jl"}

这里是过滤含有'coolcao'的对象，不论是哪个属性中含有'coolcao'都可以。
如果要精确过滤，例如只要name为coolcao的可以使用如下：
{: id="20210312121354-1a9ispb"}

```
{{[{name:'coolcao',age:23},{name:'lily',age:22},{name:'tom',age:22}] | filter:{'name':'coolcao'} }}
```
{: id="20210312121354-czut8ia"}

注意：filter 对象使用的大括号和angularjs取值所用的大括号之间要留至少一个空格（就是最后三个大括号倒数第三个和倒数1，2两个大括号之前留至少一个空格，不然angularjs会解析错误）
{: id="20210312121354-6x9ndtw"}

### date : 日期格式化
{: id="20210312121354-r5unvqg"}

在系统后台返回的数据中，时间字段，我们可能使用的是时间戳，Long型，在页面显示中肯定格式化为类似于‘2012-12-12 12:12:12’的字符串，使用date过滤器即可
{: id="20210312121354-oans29n"}

```
{{1423130269432 | date:'yyyy-MM-dd HH:mm:ss'}}
```
{: id="20210312121354-zekq448"}

显示结果：
2015-02-05 17:57:49
{: id="20210312121354-j4bf5og"}

注意：Long型的时间戳字段是以毫秒为单位的，如果系统后台使用的是以秒为单位的，那么在angular里要乘以1000转换为以毫秒为单位。这里一定要分清到底是秒还是毫秒
{: id="20210312121354-mbr3587"}

### number : 数字格式化
{: id="20210312121354-zdx69z8"}

```
{{ 3.1415926 | number:1 }}
{{ 3.1415926 | number:2 }}
{{ -3.1415926 | number:2 }}
{{ 3 | number:2 }}
{{ 0.002 | number:2 }}
{{ 0.009 | number:2 }}
{{100 | number}}
{{1000 | number}}
{{1000 | number:2}}
```
{: id="20210312121354-3mb9vos"}

啥也不说了，直接看结果：
{: id="20210312121354-ifvxnyo"}

```
3.1

3.14

-3.14

3.00

0.00

0.01

100

1,000

1,000.00
```
{: id="20210312121354-vu1rjoa"}

### orderBy   排列
{: id="20210312121354-0wpiyoo"}

```
{{[{name:'coolcao',age:23},{name:'lily',age:20},{name:'tom',age:22}] | orderBy:'age'}}
```
{: id="20210312121354-v9wcc1y"}

结果：
{: id="20210312121354-zzwioy4"}

```
[{"name":"lily","age":20},{"name":"tom","age":22},{"name":"coolcao","age":23}]
```
{: id="20210312121354-e0yhjuz"}

默认是升序排列，如果要倒序：
{: id="20210312121354-n4un21e"}

```
{{[{name:'coolcao',age:23},{name:'lily',age:20},{name:'tom',age:22}] | orderBy:'age':true}}
```
{: id="20210312121354-6k11kmf"}

### json格式化
{: id="20210312121354-wibqblf"}

```
{{[{name:'coolcao',age:23},{name:'lily',age:22},{name:'tom',age:22}] | json}}
```
{: id="20210312121354-5kykfxj"}

结果：
{: id="20210312121354-lrib5o0"}

```
[ { "name": "coolcao", "age": 23 }, { "name": "lily", "age": 22 }, { "name": "tom", "age": 22 } ]
```
{: id="20210312121354-4snnuf0"}

注意：输入是js的对象（非标准json），输出的是标准的json字符串（属性名称会用双引号）
{: id="20210312121354-01fjzu2"}

### 大小写转换： uppercase,lowercase
{: id="20210312121354-2xahp5q"}

```
{{'abc' | uppercase}}
```
{: id="20210312121354-9g6u0lj"}

将输出大写的 ABC
注意：uppercase,lowercase只能对字符串进行过滤转换
{: id="20210312121354-6rm4zml"}

### currency : 货币的格式化
{: id="20210312121354-fggkec6"}

有时候我们需要把数字显示为货币的形式方便展示，可以使用currency进行格式化
{: id="20210312121354-v88nsci"}

```
{{1000 | currency }}
{{1000 | currency:"RMB ￥" }}
```
{: id="20210312121354-eqzvsfw"}

显示：
{: id="20210312121354-ypms3b3"}

```
$1,000.00

RMB ￥1,000.00
```
{: id="20210312121354-oqdetie"}


{: id="20210312121354-sxvpdig" type="doc"}
