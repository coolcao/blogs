---
title: angular使用RxJS实现全局loading.md
date: 2021-11-15 21:40:06
tags: [angular, RxJS]
categories:
- 技术博客
- 原创
---

我们在实现前端页面时，经常会遇到使用http加载远程数据的情况，为了友好的用户体验，一般在请求http远程数据时，都会用一个加载动画来减轻用户的等待焦虑。

但如果工程比较大，页面比较多，我们在每一个组件中都设置一个loading变量，然后在每次http请求时设置loading的值，不免显得有点麻烦。

在angular里，我们可以使用RxJs的BehaviorSubject来实现一个全局的loading发射器，配合http拦截器，实现全局的loading。

<!-- more -->

首先我们定义一个`LoadingService`，用于设置全局`loading`状态。

```typescript
import { Injectable } from '@angular/core';
import { BehaviorSubject } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class LoadingService {

  public loading$: BehaviorSubject<boolean> = new BehaviorSubject<boolean>(false);
  private loadingMap: Map<string, number> = new Map<string, number>();

  constructor() { }

  setLoading(url: string, loading: boolean): void {
    if (!url) {
      throw new Error('The request URL must be provided to the LoadingService.setLoading function');
    }

    if (loading === true) {
      const count = this.loadingMap.get(url) || 0;
      this.loadingMap.set(url, count + 1);
      this.loading$.next(true);
    } else if (loading === false && this.loadingMap.has(url)) {
      const count = this.loadingMap.get(url);
      this.loadingMap.set(url, count! - 1);
      if (this.loadingMap.get(url) == 0) {
        this.loadingMap.delete(url);
      }
    }
    if (this.loadingMap.size === 0) {
      this.loading$.next(false);
    }
  }

}

```

这里，我们定义一个`BehaviorSubject<boolean>`类型的变量`loading$`，用以表示全局的`loading`发射器。`loadingMap`用以保存某个url下loading状态的个数（这里要保存loading状态的个数是因为，有些页面可能不止一个http请求，一个loading状态对应一个http请求，当所有的http请求都处理结束时，整个页面的loading才算结束）。
然后定义`setLoading(url: string, loading: boolean)`函数用以设置某个url下的loading状态。当设置为true时，则计数器加1，反之则计数器减1，当计数器为0时，使用发射器`loading$.next(false)`发送false，表明整个页面的loading状态已结束。


然后，定义一个http拦截器，拦截http请求，请求发送前设置loading为true，请求结束时，设置loading为false。

```typescript
import { Injectable } from '@angular/core';
import {
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpInterceptor,
  HttpResponse
} from '@angular/common/http';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';
import { LoadingService } from '../service/loading.service';

@Injectable()
export class LoadingInterceptor implements HttpInterceptor {

  constructor(private loadingService: LoadingService) { }

  intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    this.loadingService.setLoading(request.url, true);
    return next.handle(request).pipe(map<HttpEvent<any>, any>((evt: HttpEvent<any>) => {
      if (evt instanceof HttpResponse) {
        this.loadingService.setLoading(request.url, false);
      }
      return evt;
    }));
  }
}

```

这样我们在每次发送http请求时，就会由拦截器自动拦截，并设置loading状态。我们只需要在组件中引用`LoadingService#loading$`即可收到其发送的状态值。

```html
<nz-spin [nzSpinning]="loading$|async">
    ...
</nz-spin>
```

