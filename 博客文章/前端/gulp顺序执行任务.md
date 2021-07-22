---
title: gulp顺序执行任务
date: 2015-08-31 23:41:14
tags: [gulp]
categories:
- 技术博客
- 原创
---

在gulp中定义多个任务，可能一个任务要依赖其他某个任务完全结束后才能开始进行，比如，我们先定义两个任务，一个coffee任务，一个clean任务，coffee用于编译coffee代码到js代码，clean用于清理已经编译的代码，在编译之前先clean一下：

<!-- more -->

```javascript
gulp.task('coffee', function() {
    gulp.src('server/coffee/*.coffee')
        .pipe(coffee())
        .pipe(gulp.dest('server/js'));
});
gulp.task('clean', function() {
    gulp.src(['server/js/*.js'])
        .pipe(clean());
});
```

在官方的文档中找到如下方式,在coffee任务中添加一个参数，标记该任务所依赖的其他任务，依赖的任务先于该任务执行：

```javascript
gulp.task('coffee',['clean'], function() {
    gulp.src('server/coffee/*.coffee')
        .pipe(coffee())
        .pipe(gulp.dest('server/js'));
});
```

即，在coffee任务中添加一个任务依赖数组，数组里的任务clean要先执行，再执行任务coffee.
但是在实际操作中发现，如果文件特别多，还是会报错：

```javascript
events.js:85
      throw er; // Unhandled 'error' event
            ^
Error: ENOENT, stat '/home/coolcao/mycode/node/NutWeb/server/dao/blogDao.js'
  at Error (native)
```

这令我不解，网上各种搜，都说这样做没问题，但我这里确实不行。
又不停的问了一下谷老师，最终找到问题了：
对’clean’定义的function而言，虽然函数本身已经执行完毕了，但是文件删除操作可能仍在进行 — gulp任务中的操作大多数都是数据流(Stream)的操作，其操作进度与函数执行无关。
如果需要在文件彻底清理后才开始执行’less’任务，则需要在’clean’任务中进行特殊编码：令其返回最终的数据流(Stream)对象：

```javascript
gulp.task('clean', function() {
    return gulp.src(['server/js/*.js'])
            .pipe(clean());
});
```

对于数据流而言，代码语句的执行结束仅仅意味着数据操作的开始，唯一能确定数据操作结束的是最后一个数据流所触发的end事件；因此，只有想办法监听到这个end事件，才有可能实现真正意义上的任务依赖。而在任务定义的函数中返回最后一个数据流，是一个相对来说使用起来最方便的方案。
