---
title: 「垃圾回收的算法与实现-增量式垃圾回收」.md
date: 2021-07-28 10:47:23
tags: [垃圾回收, GC, 增量式垃圾回收]
categories:
- 技术博客
- 原创
---

增量式垃圾回收(Incremental GC)是一种通过逐渐推进垃圾回收来控制 mutator 最 大暂停时间的方法。

<!-- more -->


## 什么是增量式垃圾回收

![](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1627444533_20210728104948430_2125899621.png)

通常GC处理很繁重，一旦GC开始，就会阻碍mutator的执行，这种叫做 `停止型GC` 。根据应用程序的不同，停止型GC有时是很要命的。

因此有人提出了 `增量式GC` 。增量式垃圾回收是将 GC 和 mutator 一点点交替运行的手法。 增量式垃圾回收的示意图如图所示：
![](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1627444534_20210728105538815_162011361.png)

### 三色标记算法
这个算法就是将 GC 中的对象按照各自的情况分成三种，这三种颜色和所包含的意思分别如下所示：

- 白色:还未搜索过的对象 
- 灰色:正在搜索的对象
- 黑色:搜索完成的对象

我们以 `标记-清除算法` 为例向大家再详细地说明一下。
GC 开始运行前所有的对象都是白色。GC 一开始运行，所有从根能到达的对象都会被标记，然后被堆到栈里。GC 只是发现了这样的对象，但还没有搜索完它们，所以这些对象就成了灰色对象。

灰色对象会被依次从栈中取出，其子对象也会被涂成灰色。当其所有的子对象都被涂成灰色时，对象就会被涂成黑色。

当 GC 结束时已经不存在灰色对象了，活动对象全部为黑色，垃圾则为白色。

这就是三色标记算法的概念。有一点需要我们注意，那就是为了表现黑色对象和灰色对象，不一定要在对象头里设置标志(事实上也有通过标志来表现黑色对象和灰色对象的情况)。 在这里我们根据对象的情况，更抽象地把对象用三个颜色表现出来。每个对象是什么样的状况，意味着什么颜色，这些都根据算法的不同而不同。

此外，虽然本书中没有为大家详细说明，不过三色标记算法这个概念不仅能应用于 GC 标记-清除算法，还能应用于其他所有搜索型 GC 算法。

### 标记-清除算法的分隔
增量式的 GC 标记 - 清除算法可分为以下三个阶段。

- 根查找阶段  
我们在根查找阶段把能直接从根引用的对象涂成灰色。
- 标记阶段 
在标记阶段查找灰色对象，将其子对象也涂成灰色，查找结束后将灰色对象涂成黑色。
- 清除阶段
在清除阶段则查找堆，将白色对象连接到空闲链表，将黑色对象变回白色。

```c
incremental_gc() {
   // 检查 gc_phase 变量，判断该进入哪个阶段
   case $gc_phase
   when GC_ROOT_SCAN:    // 根查找阶段
      root_scan_phase();
   when GC_MARK:        // 标记阶段
      incremental_mark_phase();
   else                 // 清除阶段
      incremental_sweep_phase();
}
```

在根查找阶段中，我们将直接从 根引用的对象打上标记，堆放到标记栈里。根查找阶段只在 GC 开始时运行一次。

当根查找阶段结束后，incremental_gc() 函数也告一段落，mutator 会再次开始运行。接下来再次执行 incremental_gc() 函数时会进入标记阶段。在标记阶段中会调用到 incremental_mark_phase() 函数。这个函数会从标记栈中取出和搜索对象。当操作进行过一定次数后，mutator 会再次开始运行。然后周而复始，直到标记栈为空时，标记阶段就结束了。 之后的 GC 就到清除阶段了。

在清除阶段也会进行增量。incremental_sweep_phase() 函数不是一次性清除整个堆， 而是每次只清除一定个数，然后中断 GC，再次运行 mutator。

### 根查找阶段
```c
root_scan_phase() {
    // 从根引用的对象，直接做标记
   for(r: $roots) {
      mark(*r)
   }
   // 将阶段标记改为 GC_MARK
   $gc_phase = GC_MARK
}

mark(obj) {
    // 如果对象没有被标记，做下标记，并将其推到标记栈。
   if (obj.mark == FALSE) {
      obj.mark == TRUE
      push(obj, $mark_stack)
   }
}
```

> 执行完根查找阶段后，gc_phase 变成 GC_MARK ，下一次GC时会进入标记阶段。

### 标记阶段

```c
incremental_mark_phase() {
   for(i:1..MARK_MAX) {
      if (is_empty($mark_stack) == FALSE) {
         obj = pop($mark_stack)
         for(child: children(obj)) {
            mark(*child)
         }
      } else {
         for(r: $roots) {
            mark(*r)
         }
         while(is_empty($mark_stack) == FALSE) {
            obj = pop($mark_stack)
            for(child: children(obj)) {
               mark(obj)
            }
         }
      }
   }
   $gc_phase = GC_SWEEP
   $sweeping = $heap_start
   return;
}
```

在第3行到第7行，从标记栈中取出对象，将其子对象标记成灰色，将这一系列操作执行 MARK_MAX 次。
在这里 MARK_MAX次是重点，不是一次处理所有的灰色对象，而是只处理一个定数，然后暂停GC，再次开始执行mutator。这样就能缩短mutator的最大暂停时间。

第 9 行到第 17 行是即将结束标记阶段时进行的处理，在这里重新标记能从根引用的对象。 因为 GC 中会把来自于根的引用更新，所以这项处理是用来应对这次更新的。在这里如果有 很多没被标记的活动对象，可能会导致 mutator 的暂停时间延长。

> 如果把标记阶段暂停，那么再次执行mutator会发生什么？
![](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1627444534_20210728112748670_2063163578.png)

图 8.3(a) 是刚刚暂停标记阶段后的状态，A 被涂成黑色，B 被涂成灰色。所以接下来就要对 B 进行搜索了。在这里我们继续执行 mutator 吧。
图 8.3(b) 是 mutator 把从 A 指向 B 的引用更新为从 A 指向 C 之后的状态，然后再删除从 B 指向 C 的引用，就成了图 8.3(c) 这样。
那么，这个时候如果重新开始标记阶段会发生什么事呢? B 原本是灰色对象，经过搜索后被涂成了黑色。然而尽管 C 是活动对象，程序却不会对它进行搜索。这是因为已经搜索完有唯一指向 C 的引用的 A 了。
像这样单纯将 GC 标记 - 清除算法进行增量，搞不好会造成活动对象的“标记遗漏”。一旦发生标记遗漏，就会造成在清除阶段中错误回收活动对象这种重大的问题。
在这里我们回头看图 8.3(c)，问题的原因出在从黑色对象指向白色对象的指针上。一旦产生这种指针，活动对象就不会被标记。 为了防止发生这种情况，在改写指针时需要进行一些处理，于是我们在第 7 章中介绍的写入屏障又再次登场了。

### 写入屏障
分代垃圾回收中用到的写入屏障，事实上在增量式垃圾回收里也起着重要的作用。

```c
write_barrier(obj, field, newobj) {
   if (newobj.mark == FALSE) {
      newobj.mark = TRUE
      push(newobj, $mark_stack)
   }
   *field = newobj
}
```
如果新引用的对象 newobj 没有被标记，那么就将其标记后堆到标记栈里。换句话说， 如果 newobj 是白色对象，就把它涂成灰色。
下面让我们来看看如何通过这个写入屏障来防止图 8.3 中的标记遗漏问题。请看图:

![](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1627444534_20210728113112785_171807337.png)

请大家注意图 8.4(b)。可以看到，这里不仅将从 A 指向 B 的指针更新为了从 A 指向 C， 还将 C 从白色涂成了灰色。
即使在 mutator 更新指针后的图 8.4(c) 中，也没有产生从黑色对象指向白色对象的引用。 这样一来我们就成功地防止了标记遗漏。

### 清除阶段
```c
incremental_sweep_phase() {
   swept_count = 0
   while(swept_count < SWEEP_MAX) {
      if ($sweeping < $head_end) {
         if ($sweeping.mark == TRUE) {
            $sweeping.mark = FALSE
         } else {
            sweeping.next = $free_list
            $free_list = $sweeping
            $free_list += $sweeping.size
         }

         $sweeping += sweeping.size
         swept_count++
      } else {
         $gc_phase = GC_ROOT_SCAN
         return;
      }
   }
}
```

基本上讲，该函数所进行的操作就是把没被标记的对象连接到空闲链表，取消已标记的 对象的标志位。
然而，为了只对一定个数的对象执行清除操作，需要事先使用 swept_count 变量来记录已清除的对象的数量。当 swept_count >= SWEEP_MAX 时，就暂停清除阶段，再次执行 mutator。当把堆全部清除完毕时，就将 gc_phase 设为 GC_ROOT_SCAN，结束 GC。

### 分配

```c
newobj(size) {
   if ($free_size < HEAP_SIZE * GC_THRESHOLD) {
      incremental_gc()
   }
   chunk = pickup_chunk(size, $free_list)
   if (chunk != NULL) {
      chunk.size = size
      $free_size -= size
      if ($gc_phase == GC_SWEEP && $sweeping <= chunk) {
         chunk.mark = TRUE
         return chunk
      } else {
         allocation_fail()
      }
   }
}
```

首先在第 2 行调查分块的总量，如果分块的总量 free_size 少于一定的量(HEAP_SIZE 的 GC_THRESHOLD 倍)，就执行 GC。
停止型 GC 是在分块完全枯竭后才启动的。然而，因为增量式垃圾回收是逐步推进 GC 的， 所以只调用一次 incremental_gc() 函数，分块的量不会增加。因此在分块枯竭前，我们需要静下心来，稳步推进 GC 的处理。
第 5 行的 pickup_chunk() 函数被用于搜索空闲链表，返回大小大于等于 size 的分块。 
在分配了分块的情况下，接下来在第 7 行设定分块的大小，在第 8 行对 free_size 进 行 size 大小的减量。如果在这里将这个分块返回 mutator 就危险了。
在清除阶段中进行这项分配的情况下，如果不给分配的对象设置标志位，它们就有可能 被 GC 回收掉。因此在第 9 行调查现在 GC 有没有进入清除阶段，且 chunk 在不在已清除完 毕的空间里。如果 chunk 在已清除完毕的空间里，就不用做什么处理。如果 chunk 在没有被 清除完毕的空间里，就要在第 10 行明确地设置标志位。
这两种情况下的处理分别如图 8.5(a) 和图 8.5(b) 所示。 如果在代码清单 8.7 的第 5 行没能分配分块，分配就失败了。

![](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1627444535_20210728114106198_1347891391.png)


## 优点和缺点
### 缩短最大暂停时间
### 降低了吞吐量