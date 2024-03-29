---
title: 「垃圾回收的算法与实现-保守式GC与准确式GC」.md
date: 2021-07-18 13:45:07
tags: [GC, 垃圾回收, 保守式GC]
categories:
- 技术博客
- 原创
---

## 保守式GC

简单来说，保守式GC指的是“不能识别指针和非指针的GC”。

### 不明确的根

不明确的根（ambiguous roots）指的是什么呢？我们在第 1 章中讲过下面这些空间都是根。

- 寄存器
- 调用栈
- 全局变量空间

事实上它们都是不明确的根。

###  指针和非指针的识别

在不明确的根这一条件下，GC 不能识别指针和非指针。也就是说，不明确的根里所有的值都有可能是指针。然而这样一来，在 GC 时就会大量出现指针和被错误识别成指针的非指针。因此保守式 GC 会检查不明确的根，以“某种程度”的精度来识别指针。

1. 是不是被正确对齐的值？（在 32 位 CPU 的情况下，为 4 的倍数）
2. 是不是指着堆内？
3. 是不是指着对象的开头？

### 貌似指针的非指针

当基于不明确的根运行 GC 时，偶尔会出现非指针和堆里的对象的地址一样的情况，这时 GC 就无法识别出这个值是非指针。这就是“貌似指针的非指针”（false pointer）

![](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1626589706_20210718140117283_342845672.png)

保守式 GC 将这种“貌似指针的非指针”看成“指向对象的指针”，我们把这种情况叫作“指针的错误识别”。

因为被错误识别的对象不会被废弃而会被保留，所以遵守了 GC 的原则 —“不废弃活动对象”。像这样，在运行 GC 时采取的是一种保守的态度，即“把可疑的东西看作指针，稳妥处理”，所以我们称这种方法为“保守式 GC”。

### 不明确的数据结构

当基于不明确的根运行 GC 时，我们就要从对象的头部获取对象的类型信息。打个比方，将 C 语言结构体的信息作为标志（flag）放到对象的头里，请想象一下这种情况。

如果能从头的标志获得结构体的信息（对象的类型信息），GC 就能识别对象域里的值是指针还是非指针。以 C 语言为例，所有的域里面都包含了类型信息，只要程序员没有放入与类型不同含义的值（比如把指针转换类型，放入 int 类型的域），就是有可能正确识别指针的。

不过 C 语言的数据结构要是像下面这样，就会变成不明确的数据结构（ambiguous data structures）了。

```c
union {
    long n;
    void *ptr;
} ambiguous_data;
```

因为 ambiguous_data 是联合体，所以它可能包括指针 ptr，或者包括非指针 n。如果 n里包括“貌似指针的非指针”，那么 GC 就无法识别出它是非指针。当对象具有这样的数据结构时，GC 不仅会错误识别不明确的根，也会错误识别域里的值。

## 优点

### 语言处理程序不依赖于GC

保守式 GC 的优点在于容易编写语言处理程序。处理程序基本上不用在意 GC 就可以编写代码。语言处理程序的实现者即使没有意识到 GC 的存在，程序也会自己回收垃圾。因此语言处理程序的实现要比准确式 GC 简单。

## 缺点

### 识别指针和非指针需要付出成本

识别不明确的根和数据结构的值为“指针”或“非指针”时，我们需要付出一定的成本。

### 错误识别指针会压迫堆

当存在貌似指针的非指针时，保守式 GC 会把被引用的对象错误识别为活动对象。如果这个对象存在大量的子对象，那么它们一律都会被看成活动对象。因为程序把已经死了的非活动对象看成了活动对象，所以垃圾对象会严重压迫堆。

### 能够使用的GC算法有限

在无法正确识别指针的环境中，我们基本上不能使用 GC 复制算法等移动对象的 GC 算法。就如我们之前在第 4 章中所讲的那样，GC 复制算法在复制对象的时候，会将根的值重写到新空间。要是我们想用不明确的根这么办的话，就可能把非指针重写了。此外，在对象内重写指针时，也有可能因为不明确的数据结构而重写了非指针。一旦重写了非指针，就会产生意想不到的 BUG。

## 准确式GC

准确式GC和保守式GC正好相反，它是能够正确识别指针和非指针的GC。

### 正确的根

准确式 GC 和保守式 GC 不同，它是基于能精确地识别指针和非指针的“正确的根”（exact roots）来执行 GC 的。创建正确的根的方法有很多种，不过这些方法有个共通点，就是需要“语言处理程序的支援”，所以正确的根的创建方法是依赖于语言处理程序的实现的。

### 打标签

第一个方法是打标签（tag），目的是将不明确的根里的所有非指针都与指针区别开来。打标签的方法种类繁多，在这里我们为大家解说的是用最基本的低 1 位作为标签的方法。

在 32 位 CPU 的情况下，指针的值是 4 的倍数，低 2 位一定是 0，我们就利用这个特性。第 3 章中提到的 1 位引用计数法曾经将这部分用作计数器，这次我们则用它来识别指针和非指针。

打标签的具体方法如下所示。

1. 将非指针（int 等）向左移动 1 位（a << 1）
2. 将低 1 位置位（a|1）

打标签的时候我们需要注意一些地方，比如在对数值（非指针）打标签时，要注意不要让数据溢出。在向左移动 1 位时，如果数据溢出，我们就得再变换一个大的数据类型。

如果用这种方法打标签的话，处理程序里的数值就会都是奇数。因此，在处理程序内进行计算时，必须取消标签后再计算数值。为此，处理程序在计算过程中会向右移动 1 位，取消标签，最后再跟之前一样在计算结果上打标签。

基本上打标签和取消标签的操作都是由语言处理程序执行的，这就是我们之前所说的“需要语言处理程序支援 GC”。

为不明确的根里的所有非指针打标签后，GC 就能正确地识别指针和非指针了，也就是我们所说的“正确的根”。

### 不把寄存器和栈当作根

还有一种方法是不把寄存器和栈等不明确的根的关键因素当作根，而在处理程序里创建根。

具体思路就是创建一个正确的根来管理，这个正确的根在处理程序里只集合了 mutator可能到达的指针，然后以它为基础来执行 GC。

举个例子，当语言处理程序采用 VM（虚拟机）这种构造时，有时会将 VM 里的调用栈和寄存器当作正确的根来使用。

### 优点

首先，准确式 GC 完全没有保守式 GC 固有的问题 — 错误识别指针。也就是说，它不会认为已经死了的对象还活着。GC 之后，堆里只会留下活动对象。

此外，它可以实现 GC 复制算法等移动对象的算法。因为准确式 GC 能明确识别指针跟非指针，所以即使移动对象，重写根的值，这个对象也不可能是非指针。

### 缺点

当创建准确式 GC 时，语言处理程序必须对 GC 进行一些支援。也就是说，在创建语言处理程序时必须顾及 GC。排除某些特殊的实现方法，这会给实现者带来相当沉重的负担。可见在处理程序的实现上，准确式 GC 比保守式 GC 麻烦。

此外，要创建正确的根就必须付出一定的代价，而这种代价在保守式 GC 中是不存在的。好比打标签，因为每次进行计算时都需要取消标签，然后再重新设置，这就关系到语言处理程序整体的执行速度了。

## 间接引用

保守式GC有个缺点：*不能使用 GC 复制算法等移动对象的算法*。解决这个问题的方法之一就是“间接引用”。

### 经由句柄引用对象

**保守式GC无法使用复制算法，这是因为在重写不明确的根时，重写对象有可能是非指针**。解决这个问题的办法就是经由句柄来间接的处理对象。

根和对象之间有句柄。每个对象都有一个句柄，它们分别持有指向这些对象的指针。并且局部变量和全局变量这些不明确的根里没有指向对象的指针，只装着指向句柄的指针。也就是说，由 mutator 操作对象时，要通过经由句柄的间接引用来执行处理（图 6.3）。

![](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1626589706_20210718141408927_994449330.png)

只要采用了间接引用，那么即使移动了引用目标的对象，也不用改写关键的值 — 不明确的根的值，改写句柄里的指针就可以了。也就是说，我们只要采用间接引用来处理对象，就可以移动对象。请大家留意图 6.4，在复制完对象之后，根的值并没有重写。

而且，在对象内没有经由句柄指向别的对象。只有在从根引用对象时，才会经由句柄。

此外，使用了间接引用的 GC 算法不是准确式 GC。因为间接引用是以不明确的根为基础运行 GC 的，所以还是不能正确识别指针和非指针。也就是说，还是会发生错误识别指针的情况，所以这是保守式 GC。

![](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1626589706_20210718141520817_361647039.png)

### 优点

因为在使用间接引用的情况下有可能实现 GC 复制算法，所以可以得到 GC 复制算法所带来的好处，例如消除碎片化等。

另外，我们还能实现 GC 标记 - 压缩算法。

### 缺点

因为必须将所有对象都（经由句柄）间接引用，所以会拉低访问对象内数据的速度，这会关系到整个语言处理程序的速度。

## 黑名单

保守式 GC 的缺点之一就是指针的错误识别，即本来应该被看作垃圾的对象却被保留了下来。改善这个问题的办法就是采用 Hans J. Boehm [18] 发明的黑名单。

### 指针的错误识别带来的害处

在指针的错误识别中，被错误判断为活动对象的那些垃圾对象的大小及内容至关重要。具体来说，主要有下面几项。

1. 大小
2. 子对象的数量

首先是大小。打个比方，有个巨大的对象死掉了，而保守式 GC 却把它错误识别成“它还活着”，这样当然就会压迫到堆了。

然后是子对象的数量。就算保守式 GC 错误识别了一个子对象也没有的小对象，由此带来的损失也并不大。对于整个堆来说，留下一个对象并不会有什么大的影响。

但是要换成错误识别了有一堆子对象的对象，这损失就大了。保守式 GC 会错误识别子对象的子对象，以及子对象的子对象的子对象，错误就会像多米诺骨牌一样连续下去。

![](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1626589707_20210718141730315_169653106.png)

### 黑名单

顾名思义，黑名单就是一种创建“需要注意的地址的名单”的方法。这个黑名单里记录的是“不明确的根内的非指针，其指向的是有可能被分配对象的地址”。我们将这项记录操作称为“记入黑名单”。

有可能被分配对象的地址指的又是什么呢？举个有代表性的例子，就是“堆内未使用的对象的地址”。

mutator 无法引用至今未使用过的对象。也就是说，如果根里存在有这种地址的指针，那它肯定就是“非指针”，就会被记入黑名单中。

我们在 GC 标记 - 清除算法中的 mark() 函数里导入记入黑名单的操作。

```c
mark(obj) {
    if ($heap_start <= obj && obj <= $heap_end) {
        if (!is_used_object(obj)) {
            obj.next = $blcklist
            $blcklist = obj
        } else {
            if (obj.mark == FALSE) {
                obj.mark = TRUE
                for (child: children(obj)) {
                    mark(*child)
                }
            }
        }
    }
}
```

### 面向黑名单内的地址的分配

黑名单里记录的是“需要注意的地址”。一旦分配程序把对象分配到这些需要注意的地址中，这个对象就很可能被非指针值所引用。也就是说，即使分配后对象成了垃圾，也很有可能被错误识别成“它还活着”。

为此，在将对象分配到需要注意的地址时，所分配的对象有着如下限制条件。

- 小对象
- 没有子对象的对象

为什么只能分配上述这样的对象呢？因为如果这样的对象成了垃圾，即使被错误识别了，也不会有什么大的损失。

因为它们足够小，也没有子对象，所以能把对堆的压迫控制在最低限度。

### 优点和缺点

这样一来，保守式 GC 因错误识别指针而压迫堆这个保守式 GC 的一大问题就得到了缓解，堆的使用效率也得到了提升。因此，这里的 GC 对象少于通常的保守式 GC 里的对象，GC 的执行速度也大大提升了。

不过相应地，在分配对象时则需要花功夫来检查黑名单。

## 总结

1. 所谓保守式GC就是，在收集器无法识别`指针`和`非指针`的情况下，有可能会出现指针识别错误，保守起见，会保留这些“识别错误的活动对象”。
2. 保守式GC的优点是语言处理程序不依赖于GC，即语言处理程序不用特意为GC而作出一些特殊的处理。
3. 准确式GC是能够正确识别 `指针` 和 `非指针`。它是基础“正确的根”来执行GC的，创建“正确的根”需要语言处理程序的支持。
4. 创建正确的根，语言处理程序有两种方法：打标签和不把寄存器和栈当作根
5. 准确式GC的优点是能够正确识别指针和非指针，不会对堆造成压迫。另外就是，可以使用复制算法等需要移动对象的算法。
6. 针对保守式GC不能使用复制算法等移动对象的算法，可以采用间接引用的方法，经由句柄引用对象来优化。
7. 针对保守式GC指针识别错误，可以采用黑名单的方式来优化。