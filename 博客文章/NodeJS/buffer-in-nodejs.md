---
title: 理解nodejs中的Buffer
date: 2017-01-06 16:56:01
tags: [js,nodejs]
categories:
- 技术博客
- 原创
---
{: id="20210312121354-lmcn7iu"}

nodejs的优势在于编写高性能的网络服务，而网络请求中，Stream和Buffer是其基础，因此理解这两个概念至关重要。
而Buffer又是Stream的基础，所以，先来看看Buffer吧。然后再去搞Steam。
{: id="20210312121354-68cpctc"}

<!--more-->

## Buffer是什么
{: id="20210312121354-8rw765c"}

Buffer是一个类数组对象，里面存储的是字节，有点类似于字节数组，主要用于操作字节的。
{: id="20210312121354-zkicman"}

来个例子：
{: id="20210312121354-vlyom88"}

```js
let str = 'Hello World!';
let buffer = Buffer.from(str);
console.log(buffer);
```
{: id="20210312121354-mvn4kzm"}

将输出`<Buffer 48 65 6c 6c 6f 20 57 6f 72 6c 64 21>`
{: id="20210312121354-5686vwk"}

我们可以看出其是十六进制的字节编码。Buffer.from(string[,encoding])方法接收两个参数，第二个是编码方法，如果不传，默认将使用UTF-8编码。
要想了解更多关于编码的内容，推荐阮老师的文章：[《字符编码笔记：ASCII，Unicode和UTF-8》](http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html)
{: id="20210312121354-q8s3a05"}

为什么说Buffer是一个类数组对象呢？
因为Buffer有几个和数组相似的属性和方法，比如length属性，indexOf()方法，includes()方法等等。最重要的是它有一个length属性，以及可以直接使用下标访问，例如：
{: id="20210312121354-2ff0dcm"}

```js
console.log(buffer.length); //12
console.log(buffer[0]);     //72
```
{: id="20210312121354-eo03ezj"}

上面例子中，buffer的长度是12，有12个字节。位置0的字节值为72，对应16进制的 48 。
{: id="20210312121354-st1vyre"}

Buffer中的每一个元素（Buffer不是数组，因此严格意义上说，‘每一个元素’这种描述是错的，这里方便描述）都是一个16进制的二位数，因此它的大小在0~255之间。
为什么呢？刚才说过了Buffer存的是字节，一个字节是由8个位组成的，因此它的大小是0~255之前，而使用16进制表示出来就是2位。
{: id="20210312121354-kjw1i5y"}

nodejs的Buffer有丰富的接口供使用，例如分配空间，填充Buffer，赋值等等，具体可参阅[官方文档](https://nodejs.org/dist/latest-v7.x/docs/api/buffer.html)。
{: id="20210312121354-t60tkc6"}

Buffer的每个元素都是0~255的整数，那么如果赋值给小数甚至负数会怎样呢？例如：
{: id="20210312121354-5nbh85h"}

```js
let bf2 = Buffer.alloc(2);
bf2[0] = 72.5;
bf2[1] = -72;
console.log(bf2); //<Buffer 48 b8>
```
{: id="20210312121354-l5zz9eq"}

它的两个元素分别是`0x48`和`0xb8`，对应十进制是 72,184。可以看出，如果赋值小数，则直接截断为整数，负数的话，184和-72啥关系呢？哦，-72+256=184。
{: id="20210312121354-l392bm7"}

> 给元素赋值如果小于0，就将该值逐次加256，直到得到一个0到255之间的整数。如果赋值大于255，就逐次减去256，直到得到0到255区间内的数值。如果是小数，舍弃小数部分，只保留整数部分。
> {: id="20210312121354-p0rp1ej"}
{: id="20210312121354-dm3en1h"}

## Buffer与字符串的转换
{: id="20210312121354-7t5v4g8"}

字符串可以通过Buffer.from(string[,encoding])来转换为Buffer，如果不指定encoding，默认使用utf-8编码。
{: id="20210312121354-ah38r0m"}

目前支持的编码有如下几种:
{: id="20210312121354-4bb8auf"}

* {: id="20210312121354-055b09u"}ASCII
  {: id="20210312121354-0z97gvt"}
* {: id="20210312121354-2euh06w"}UTF-8
  {: id="20210312121354-ktzekis"}
* {: id="20210312121354-2jb3i5m"}UTF-16LE/UCS-2
  {: id="20210312121354-50uxx4p"}
* {: id="20210312121354-0gclwgp"}Base64
  {: id="20210312121354-jzw3rr5"}
* {: id="20210312121354-2s8keb6"}Binary
  {: id="20210312121354-srt4u2w"}
* {: id="20210312121354-jcw3meh"}Hex
  {: id="20210312121354-4qipirs"}
{: id="20210312121354-n80u0bz"}

由于nodejs内置的转换编码并不支持GBK，因此如果要处理编码为GBK的文档，要借助第三方的插件，推荐[iconv-lite](https://github.com/ashtuchkin/iconv-lite)。
{: id="20210312121354-y7zlf3s"}

```js
let str = 'Hello,你好';

console.log(Buffer.from(str,'utf-8'));
console.log(Buffer.from(str,'ascii'));
console.log(Buffer.from(str,'utf-16le'));
console.log(Buffer.from(str,'base64'));
console.log(Buffer.from(str,'binary'));
console.log(Buffer.from(str,'hex'));

//<Buffer 48 65 6c 6c 6f 2c e4 bd a0 e5 a5 bd>
//<Buffer 48 65 6c 6c 6f 2c 60 7d>
//<Buffer 48 00 65 00 6c 00 6c 00 6f 00 2c 00 60 4f 7d 59>
//<Buffer 1d e9 65 a3>
//<Buffer 48 65 6c 6c 6f 2c 60 7d>
//<Buffer >
```
{: id="20210312121354-4jctwt9"}

注意最后一个，hex出来怎么是空呢？因为hex只支持十六进制的字符串。
{: id="20210312121354-m19z6mo"}

```js
console.log(Buffer.from('48656c6c6f2ce4bda0e5a5bd','hex').toString('utf-8')); //Hello,你好
```
{: id="20210312121354-62qudhg"}

## Buffer的拼接
{: id="20210312121354-gbud7ks"}

上面例子我们看出了，字符串和Buffer的转换和编码息息相关。而且即使是相同编码，如果Buffer被截断，那么也有可能出现乱码。
{: id="20210312121354-u4lsadj"}

```js
let str = 'Hello,你好';
let bf = Buffer.from(str);
console.log(bf.slice(0,7).toString());  //Hello,��
```
{: id="20210312121354-fjwx7qk"}

从上面例子中我们看出，'你好'两个汉字，分别占用3个字节，这里我取bf的前8个字节，很明显是将'你'字的3个字节给分开了，只取了前2个字节，那么，在转换为字符串时，由于不能正确识别出现乱码。
{: id="20210312121354-91lnib1"}

我们在做网络请求或者使用流读取文件时，由于可能会读取多次，不可避免的出现这种情况，一个汉字的字节被截断，导出出现乱码：
{: id="20210312121354-x2q2re0"}

```js
'use strict';
const fs = require('fs');

//test.txt
//Hello,你好

let rs = fs.createReadStream('./test.txt', {
    highWaterMark: 7
});
let txt = '';
rs.on('data',(chunk) => {
    txt += chunk;
});

rs.on('end',() => {
    console.log(txt);
});

//Hello,���好
```
{: id="20210312121354-8s0acfd"}

这里出现乱码了，是因为`txt += chunk`隐含了一个操作，即 `chunk.toString()`，因为txt是String类型的。因此相当于是`txt += chunk.toString()`。
由于每次限定只读取7个字节，因此'你'字被截断，解析时成乱码。
{: id="20210312121354-0ordt5d"}

使用http发送网络请求时，也是同样的原理。
{: id="20210312121354-6mrb7fi"}

不要紧，这里我们可以直接将chunk拼接成一个大的Buffer,然后再转换成字符串。
{: id="20210312121354-5lwpyqn"}

```js
let bfs = [];
rs.on('data',chunk => {
    bfs.push(chunk);
});
rs.on('end',() => {
    console.log(bfs.toString());
});
```
{: id="20210312121354-95ios0c"}

如果是网络请求这样做还好，因为每次请求的数据不会太大，不至于出现内存不够的情况。但是如果是读取一个大的文件，比如几百M或者几个G的情况下，很明显不能这样拼接了，因为内存可能不够。
{: id="20210312121354-d3bsgfr"}

例如，有一个很大的纯文本文件，utf-8编码，如何正确读取其内容然后显示在终端？
{: id="20210312121354-l18jkyi"}

这个问题只能分段读取，然后分段显示。那么，问题又来了，上面的乱码问题该如何解决？
{: id="20210312121354-3hii6so"}

nodejs有一个神奇的string_decoder模块，神奇在哪，来个实验：
{: id="20210312121354-m0j58m3"}

```js
const StringDecoder = require('string_decoder').StringDecoder;
const decoder = new StringDecoder('utf-8');
let str = 'Hello,你好';
let bf = Buffer.from(str);
let bf1 = bf.slice(0,7);
let bf2 = bf.slice(7,bf.length);

console.log(bf1.toString());
console.log(bf2.toString());
console.log(decoder.write(bf1));
console.log(decoder.write(bf2));

//Hello,�
//��好
//Hello,
//你好
```
{: id="20210312121354-dkrsoty"}

bf1和bf2分别是bf的前后两段，但是直接toString输出乱码，使用string_decoder输出则正常？何解？
{: id="20210312121354-kvgev08"}

将Buffer传给StringDecoder解析写入时，前面`Hello,`能够正常解析，最后一个字节不能解析，留在decoder里，第二次解析bf2时，会和bf2拼接起来一起解析，因此'你'在第二次解析中输出。
{: id="20210312121354-grds90j"}

但是string_decoder有个问题，只能处理utf-8，base64，和UCS-2/UTF-16LE三种编码。对于其他编码无能为力。因此，如果是其他编码的，不能直接使用它。
{: id="20210312121354-0aapeh0"}

对于GBK编码的文件或者网络请求，我们该如何处理呢？上面提到一个第三方的转换插件iconv-lite模块，我们可以使用这个模块进行转换。
{: id="20210312121354-gr0gw6x"}

例如，再有一个很大的文本文件，编码是GBK的，如何正确读取并显示在终端？
{: id="20210312121354-f9jk8iv"}

```js
'use strict';
const fs = require('fs');
const iconv = require('iconv-lite');

let rs = fs.createReadStream('./中文测试.md', {
    highWaterMark: 7
});
let StringDecoder = require('string_decoder').StringDecoder;
let decoder = new StringDecoder('utf-8');

let readable = rs.pipe(iconv.decodeStream('GBK'))
    .pipe(iconv.encodeStream('utf-8'));

readable.on('readable', () => {
  var chunk;
  while (null !== (chunk = readable.read())) {
    console.log(decoder.write(chunk));
  }
});
```
{: id="20210312121354-8b652p8"}

中文测试.md是一个GBK编码的纯文本文件，我们创建一个读取流rs，然后通过管道将该流扔给iconv，iconv使用GBK编码解码流，并将其转换成utf-8编码的流，最后通过string_decoder输出。
{: id="20210312121354-ps8jfs0"}

## 小结
{: id="20210312121354-mg2ugih"}

对于Buffer的操作，一定要注意几点：
{: id="20210312121354-5eursn5"}

* {: id="20210312121354-coeoqmk"}编码一定要统一。不管是网络服务还是文件读取，一定要统一编码。如果编码不统一，则先进行转码。
  {: id="20210312121354-vjqie7p"}
* {: id="20210312121354-zds1k8c"}注意字节完整性。对于多字节的字符，千万不要出现截断字节的情况。否则会乱码。
  {: id="20210312121354-yiymyul"}
{: id="20210312121354-qxs07qt"}


{: id="20210312121354-0gths3o" type="doc"}
