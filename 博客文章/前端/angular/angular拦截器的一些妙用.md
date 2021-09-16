# angular拦截器的一些妙用

---
title: angular拦截器的一些妙用
date: 2021-09-16 18:48:36
tags: [ angular, 拦截器, 前端 ]
categories:
- 技术博客
- 原创
---

Angular的HttpClient实现了拦截器机制，可以对请求进行拦截与修改，过滤等操作。
基于这种特性，我们可以很方便的将有关http请求的一些逻辑抽离出来，对代码进行解藕。

## 拦截器

```typescript
import { Injectable } from '@angular/core';
import {
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpInterceptor
} from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable()
export class AuthInterceptor implements HttpInterceptor {

  constructor() {}

  intercept(request: HttpRequest<unknown>, next: HttpHandler): Observable<HttpEvent<unknown>> {
    return next.handle(request);
  }
}

```

新建一个拦截器，继承自`HttpInterceptor`，我们需要做的就是使用自己的逻辑重写`intercept`方法。

其中参数`request`为接收到的请求，`next`为处理完毕，交给下一个拦截器处理。多个拦截器形成链式处理。

今天这篇文章，简单介绍下面几个场景，可以采用拦截器的例子。

## 登录认证

前端系统与后端api进行交互时，一般都要做用户认证。
用户登录完成，后端返回一个认证凭证token，然后前端每次请求时，都要带上这个凭证。

在这种情况下，我们就可以使用拦截器，在每一次请求上加上这个凭证，而不用每次请求手动添加。

```typescript

@Injectable()
export class TokenInterceptor implements HttpInterceptor {

  constructor(private authService: AuthService) { }

  intercept(request: HttpRequest<unknown>, next: HttpHandler): Observable<HttpEvent<unknown>> {

    let req = request.clone({
      setHeaders: { 'Content-Type': 'application/json;charset=utf-8' }
    });
    const authInfo = this.authService.getAuthInfo();
    if (authInfo) {
      const token = authInfo.accessToken;
      if (token) {
        req = req.clone({
          setHeaders: { 'x-auth-token': token }
        });
      }
    }
    return next.handle(req);
  }
}

```

我们将auth信息存储到sessionStore中，封装`AuthService`进行访问。

在拦截器内，获取auth信息，如果获取到，则将其token放到header中。


## 全局Http异常处理
拦截器不仅能处理`request`，还能对请求的`response`进行处理。虽然在拦截方法中，并没有`response`参数。但其返回是一个Observable，这样我们就可以对其进行操作了。

```typescript
@Injectable()
export class ErrorHandleInterceptor implements HttpInterceptor {

  constructor(private errorService: ErrorService,
    private loadingService: LoadingService) { }

  intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    return next.handle(request).pipe(
      catchError(err => {
        if (err instanceof HttpErrorResponse) {
          this.errorService.handleHttpError(err);
        }
        return new Observable<HttpEvent<any>>();
      })
    );
  }
}

```

`next.handle(request)`返回一个Observable，我们可以使用RxJs操作符对其进行修改。

使用`catchError`操作符捕捉错误。如果捕捉到的错误是一个`HttpErrorResponse`，则将这个错误交给`ErrorService`处理。

## 全局Loading状态
一般为了更好的用户体验，在进行一些耗时的HTTP请求时，会在页面上加一些loading效果，一来减少用户的焦虑感，二来也避免用户重复提交。
如果在每个页面上都单独写一个loading状态来控制，会显得很繁琐。这里也可以使用拦截器进行全局处理。

比如我们将上面错误处理的拦截器稍微改一下，在其中加上全局loading状态即可。

```typescript

@Injectable()
export class ErrorHandleInterceptor implements HttpInterceptor {

  constructor(private errorService: ErrorService,
    private loadingService: LoadingService) { }

  intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    this.loadingService.setLoading(request.url, true);
    return next.handle(request).pipe(
      catchError(err => {
        this.loadingService.setLoading(request.url, false);
        if (err instanceof HttpErrorResponse) {
          this.errorService.handleHttpError(err);
        }
        return new Observable<HttpEvent<any>>();
      })
    ).pipe(map<HttpEvent<any>, any>((evt: HttpEvent<any>) => {
      if (evt instanceof HttpResponse) {
        this.loadingService.setLoading(request.url, false);
      }
      return evt;
    }));
  }
}
```


在请求开始之前，`this.loadingService.setLoading(request.url, true);` 设置loading状态为true，如果捕捉到错误，或者正常结束请求，`this.loadingService.setLoading(request.url, false);`将loading状态设置为false即可。

其中LoadingService代码如下：

```typescript
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


> 注：HTTP拦截器只能全局处理与HTTP请求相关的loading状态，那如果是其他操作引起的loading状态变化，那该怎么办呢？只能再单独处理么？
> 其实也不是。完全可以复用上面的`LoadingService`，需要将loading设置为true时，调用`loadingService.setLoading(url, true)`，处理完成使用`loadingService.setLoading(url, false)`即可。url这里可以直接设置为当前组件的文件地址，只要能区分不同的组件或页面即可。

## 结果分页处理
有时候后端api的一些额外数据，并不放到body体里，放到了header中，比如分页数据。
这个时候，如果为每个具有分页的接口都写一个从header中获取数据的逻辑，会导致代码重复且耦合高。

这里我们可以使用拦截器对response进行处理，统一将header中的数据拿出来，和body统一起来。
```typescript
@Injectable()
export class PaginationResultInterceptor implements HttpInterceptor {
  constructor() { }
  intercept(request: HttpRequest<unknown>, next: HttpHandler): Observable<HttpEvent<unknown>> {
    return next.handle(request).pipe(
      filter(event => event instanceof HttpResponse),
      map((event: HttpEvent<any>) => {
        const e = event as HttpResponse<any>;
        const recordTotal = e.headers.get('x-content-record-total') as string;
        // 修改分页数据结构
        if (recordTotal) {
          const total = Number.parseInt(recordTotal);
          return e.clone({
            body: { total, data: e.body }
          });
        }
        return e;
      })
    );
  }
}
```

