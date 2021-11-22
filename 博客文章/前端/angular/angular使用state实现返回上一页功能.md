---
title: angular使用state实现返回上一页功能.md
date: 2021-11-16 14:18:33
tags: [angular]
categories:
- 技术博客
- 原创
---

返回上一页可能是web应用中最常用的一个功能，实现方式也有很多种。
一般在设计返回上一页功能时，都要记住上一页的状态。
比如从一个列表页点击到详情页时，列表页可能有一些状态，比如当前的搜索条件，或者是列表的分页等状态，再从详情页点击“返回上一页”时，除了要跳转到列表页，还要记住之前的搜索条件或分页等状态。
<!-- more -->

但如果只是简单的使用`history.back()`，会丢失之前页面的状态。

所以要记住之前页面的状态，就必须在跳转时，带上这些状态数据。

进行路由跳转时携带数据，可以使用`queryParams`，将参数以query的形式拼装在url后面，这种方式能达到效果，但也有一定的限制，页面之间的跳转都是一一对应写死的，并不灵活。并且由于会在url后面携带参数，直接刷新页面时，并不能“真正刷新”页面。

所以比较常用的就是使用History.state，各大框架对其也有一定的封装应用。

在angular中，其路由模块在使用`routerLink`进行路由跳转时可附加state进行状态传递。

我们可以抽象出一个公共的Service`GobackService`来监听路由变化并存储页面跳转的state数据：

```typescript
import { Injectable } from '@angular/core';
import { NavigationEnd, Router } from '@angular/router';
import { filter } from 'rxjs/operators';

import { BackState } from '../interface/back-state';

@Injectable({
  providedIn: 'root'
})
export class GobackService {

  // 标记要返回的页面url
  url?: string;
  backState?: BackState;

  constructor(private router: Router,
  ) {

    this.router.events.pipe(
      filter(event => event instanceof NavigationEnd)
    ).subscribe(() => {
      const navigation = router.getCurrentNavigation();
      if (navigation) {
        if (navigation.previousNavigation && navigation.previousNavigation.finalUrl) {
          this.url = navigation.previousNavigation.finalUrl.toString();
        }
        if (navigation.extras && navigation.extras.state) {
          this.backState = navigation?.extras.state as BackState;
        }
      }
    });
  }

  // 返回上一页
  goBack() {
    if (this.url) {
      this.router.navigate([this.url], { state: { ...this.backState, goback: true } });
    }

  }

  // 获取上一页传来的状态参数
  getState(): BackState | undefined {
    return this.backState;
  }
}

```

然后在需要返回上一页功能的页面中导入该Service。
比如这里我们有两个页面：A-List和A-Detail两个页面，从A-List页面中某个具体的一项点击进入A-Detail页面查看详情。在A-Detail页有一个“返回上一页”的按钮。

我们只需要在A-List页面中使用:

```html
<a [routerLink]="['/prizes', data.id]" [state]="toBackState()">
```

```typescript
  fromBackState() {
    const backState = this.gobackService.getState();
    if (backState && backState.goback) {
      if (backState.pageIndex) {
        this.page.index = backState.pageIndex;
      }
      if (backState.extraParams) {
        this.prizeName = backState.extraParams['prizeName'];
      }
    }
  }
  toBackState() {
    const state = {
      pageIndex: this.page.index,
      extraParams: {
        prizeName: this.prizeName
      }
    };
    return state;
  }
```

在A-Detail页面中直接使用`service.goBack()`即可。

```typescript
  fromBackState() {
    const backState = this.gobackService.getState();
    if (backState && backState.goback) {
      if (backState.pageIndex) {
        this.page.index = backState.pageIndex;
      }
      if (backState.extraParams) {
        this.prizeName = backState.extraParams['prizeName'];
      }
    }
  }
  toBackState() {
    const state = {
      pageIndex: this.page.index,
      extraParams: {
        prizeName: this.prizeName
      }
    };
    return state;
  }
```
