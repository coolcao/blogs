---
title: Angular 中 HttpParameterCodec 对 + 编码的 bug
date: 2022-12-02 01:59:00
tags: [Angular, urlencode]
categories:
  - 技术博客
  - 原创
---

在一个前端使用 Angular， 后端使用 SpringBoot 的项目中，有一个对象的某个属性值中带有 `+` 字符，前端发到后端后，发现 `+` 被解析成了 空格 ` ` 。

<!--more-->

## 目录

- [Angular 中 HttpParameterCodec 对 + 编码的 bug](#angular中httpparametercodec对--编码的bug)
  - [现象](#现象)
  - [解决方案](#解决方案)
  - [原因解析](#原因解析)
    - [源码分析](#源码分析)
    - [问题追踪](#问题追踪)
    - [问题分析](#问题分析)
  - [意外收获](#意外收获)
    - [空格的转换](#空格的转换)
    - [encodeURI 和 encodeURIComponent](#encodeuri和encodeuricomponent)

## 现象

前端代码大致如下：

```typescript
const url = "url";

let queryParams = new HttpParams();

if (params.name) {
  queryParams = queryParams.set("name", params.name);
}
this.http.get(url, { params: queryParams });
```

比如有一个 name 参数值是 `edqd+0I5FKI` ，后端接收到的参数中， `+` 被转换成了空格，成了 `edqd 0I5FKI` ，经过排查发现，前端发送给后端的数据中，确实带了 `+` 号，完成的数据就是 `edqd+0I5FKI` ，而后端在解析时，要经过 URLDecode ，加号 `+` 经过 URLDecode 后就是空格。

所以，这里的根本问题在于，前端在发送数据时，未对 `+` 进行 URLEncode。也就是 Angular 的 HTTP Client 未对参数中的 `+` 进行 URL 转码。

## 解决方案

先说解决方案，如果有兴趣可以再看后面的原因解析。

解决方案其实也很简单，Angular 的 Http 客户端未对 `+` 做转码，那么我们自己做一下转码即可。

⓵ 创建一个拦截器，拦截器内指定 HttpParams 的编码器

```typescript
import { Injectable } from "@angular/core";
import {
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpInterceptor,
  HttpParameterCodec,
  HttpParams,
} from "@angular/common/http";
import { Observable } from "rxjs";

/**
 * 自定义Angular的HTTP参数Encoder
 * 因为Angular内置的编码方法将一些特殊字符未做转换，包括 @ : $ , ; + ; ? /
 * 所以这里重写其Encoder，直接使用标准的encodeURIComponent()方法进行转码
 */
class CustomEncoder implements HttpParameterCodec {
  encodeKey(key: string): string {
    return encodeURIComponent(key);
  }
  encodeValue(value: string): string {
    return encodeURIComponent(value);
  }
  decodeKey(key: string): string {
    return decodeURIComponent(key);
  }
  decodeValue(value: string): string {
    return decodeURIComponent(value);
  }
}

@Injectable()
export class QueryParamEncodeInterceptor implements HttpInterceptor {
  constructor() {}

  intercept(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    const params = new HttpParams({
      encoder: new CustomEncoder(),
      fromString: request.params.toString(),
    });
    return next.handle(request.clone({ params }));
  }
}
```

⓶ 在 `app.module.ts` 中将该拦截器提供给 `providers`

```typescript
providers: [
    {
      provide: HTTP_INTERCEPTORS,
      useClass: QueryParamEncodeInterceptor,
      multi: true,
    }
  ],
```

## 原因解析

### 源码分析

Angular 中负责对 URL 参数进行编码的类是 `HttpUrlEncodingCodec` ，该类实现了 `HttpParameterCodec` 接口。

```typescript
/**
 * Provides encoding and decoding of URL parameter and query-string values.
 *
 * Serializes and parses URL parameter keys and values to encode and decode them.
 * If you pass URL query parameters without encoding,
 * the query parameters can be misinterpreted at the receiving end.
 *
 *
 * @publicApi
 */
export declare class HttpUrlEncodingCodec implements HttpParameterCodec {
  /**
   * Encodes a key name for a URL parameter or query-string.
   * @param key The key name.
   * @returns The encoded key name.
   */
  encodeKey(key: string): string;
  /**
   * Encodes the value of a URL parameter or query-string.
   * @param value The value.
   * @returns The encoded value.
   */
  encodeValue(value: string): string;
  /**
   * Decodes an encoded URL parameter or query-string key.
   * @param key The encoded key name.
   * @returns The decoded key name.
   */
  decodeKey(key: string): string;
  /**
   * Decodes an encoded URL parameter or query-string value.
   * @param value The encoded value.
   * @returns The decoded value.
   */
  decodeValue(value: string): string;
}
```

我们看一下 `HttpUrlEncodingCodec` 类的具体实现方法，在 `@angular/common/esm2015/http/src/params.js` 中可以看到：

```typescript
/**
 * @license
 * Copyright Google LLC All Rights Reserved.
 *
 * Use of this source code is governed by an MIT-style license that can be
 * found in the LICENSE file at https://angular.io/license
 */
/**
 * Provides encoding and decoding of URL parameter and query-string values.
 *
 * Serializes and parses URL parameter keys and values to encode and decode them.
 * If you pass URL query parameters without encoding,
 * the query parameters can be misinterpreted at the receiving end.
 *
 *
 * @publicApi
 */
export class HttpUrlEncodingCodec {
    /**
     * Encodes a key name for a URL parameter or query-string.
     * @param key The key name.
     * @returns The encoded key name.
     */
    encodeKey(key) {
        return standardEncoding(key);
    }
    /**
     * Encodes the value of a URL parameter or query-string.
     * @param value The value.
     * @returns The encoded value.
     */
    encodeValue(value) {
        return standardEncoding(value);
    }
    /**
     * Decodes an encoded URL parameter or query-string key.
     * @param key The encoded key name.
     * @returns The decoded key name.
     */
    decodeKey(key) {
        return decodeURIComponent(key);
    }
    /**
     * Decodes an encoded URL parameter or query-string value.
     * @param value The encoded value.
     * @returns The decoded value.
     */
    decodeValue(value) {
        return `(value);
    }
}
```

我们可以看到，两个 decode 方法使用了 `decodeURIComponent()` 函数，而 encode 方法并没有使用标准的 `encodeURIComponent()` 函数，而是使用了 Angular 自己写的 `standardEncoding()` 函数：

```typescript
/**
 * Encode input string with standard encodeURIComponent and then un-encode specific characters.
 */
const STANDARD_ENCODING_REGEX = /%(\d[a-f0-9])/gi;
const STANDARD_ENCODING_REPLACEMENTS = {
  "40": "@",
  "3A": ":",
  "24": "$",
  "2C": ",",
  "3B": ";",
  "2B": "+",
  "3D": "=",
  "3F": "?",
  "2F": "/",
};
function standardEncoding(v) {
  return encodeURIComponent(v).replace(STANDARD_ENCODING_REGEX, (s, t) => {
    var _a;
    return (_a = STANDARD_ENCODING_REPLACEMENTS[t]) !== null && _a !== void 0
      ? _a
      : s;
  });
}
```

这个 `standardEncoding()` 函数其实就是调用了标准的 `encodeURIComponent()` 函数，只不过对几个特殊的字符（`@:$;,+=?/`）跳过，不做转码。

所以，解决方案就如同上面所说，我们自己实现一个 `HttpParameterCodec` ，将其上面几个所谓的特殊字符也一同转码了即可。

### 问题追踪

那么，问题来了，Angular 为什么会这么做？按照规范，`+` 字符被 URLEncode 后，就应该被转换为 `%2B` 呀，Angular 为什么会跳过这些特殊字符的转码呢？

> 上面的其他字符跳过转码，其实也还好，但是 `+` 很特殊，如果前端发给后端的数据中带有 `+` 字符，按照规范，`+` 会被解码成空格 ` ` ，这也就是最开始我们遇到的问题。
> Angular 未对 `+` 做转码，到后端进行 URL Decode 解码时，`+` 会被解析成空格，导致数据发生变化。

网上查了一下，在 Angular 的官方 github issues 中发现有这么一个 issues:

[Breaking change with RC2: Sending Urls with search params because of encoding search values · Issue #9348 · angular/angular · GitHub](https://github.com/angular/angular/issues/9348 "Breaking change with RC2: Sending Urls with search params because of encoding search values · Issue #9348 · angular/angular · GitHub")

提出这个问题的哥们大致是说，Angular2 升级到 RC2(相当早相当早了)时，对 query 参数的转码发生了破坏性的变化：

原来是 `q=repo:janbaer/howcani+type:issue&sort=created&order=desc&page` ，升级后成了 `q=repo%3Ajanbaer%2Fhowcani%2Btype%3Aissue&sort=created&order=desc&page=1`

然后 Angular 官方就根据这个问题，提了一个修复，对 query 参数中的几个特殊字符(`@ : $ , ; + ; ? /`)不进行转码，这其中就包括 `+` 字符，2016 年 6 月份修复的，到目前已经 6 年半了。

[fix(http): don't encode values that are allowed in query by jeffbcross · Pull Request #9651 · angular/angular · GitHub](https://github.com/angular/angular/pull/9651 "fix(http): don't encode values that are allowed in query by jeffbcross · Pull Request #9651 · angular/angular · GitHub")

[https://github.com/hbkrunal/angular/commit/47d2f1f1b82d85da2033ee6daa78b769e4a387d2](https://github.com/hbkrunal/angular/commit/47d2f1f1b82d85da2033ee6daa78b769e4a387d2 "https://github.com/hbkrunal/angular/commit/47d2f1f1b82d85da2033ee6daa78b769e4a387d2")

### 问题分析

那么，这算是一个 bug 么？

这整个过程很有意思，先是 Angular 对参数所有字符都做了转码，后来有人提了 bug，然后就放开了上面几个特殊字符的转码，为什么要放开这几个字符的转码，放开后，却有产生了因为解码导致 `+` 被“误解”的 bug？那么这几个字符，到底应不应该进行转码？

很有意思的是，经过上面一个“修复”后，又有很多人提出了由于 `+` 字符不做转码导致 bug 的 issue，里面也有很多人在讨论：

[common/http: HttpParams encoding of form data · Issue #18261 · angular/angular · GitHub](https://github.com/angular/angular/issues/18261 "common/http: HttpParams encoding of form data · Issue #18261 · angular/angular · GitHub")

[HttpParameterCodec improperly encodes special characters like '+' and '=' · Issue #11058 · angular/angular · GitHub](https://github.com/angular/angular/issues/11058 "HttpParameterCodec improperly encodes special characters like '+' and '=' · Issue #11058 · angular/angular · GitHub")

[HttpParameterCodec improperly encodes special characters like '+' and '=' · Issue #11058 · angular/angular · GitHub](https://github.com/angular/angular/issues/11058 "HttpParameterCodec improperly encodes special characters like '+' and '=' · Issue #11058 · angular/angular · GitHub")

尤其最后一个 issue 中，有很多人不理解，为什么 Angular 默认的转码器不对 `+` 和 `=` 进行转码，这样会有很大的机率导致 bug。因为很多后端服务器，包括 php，python，以及 java 的 tomcat 等等，都会将 `+` 解码为空格，导致 bug 的出现。（按照标准，`+` 字符在解码时就是会被解码为空格，目测应该后端服务都会这样，不仅仅前面提到的 php，python，tomcat 等等）

在讨论中有人提出，Angular 应该默认就是将参数所有字符都进行转码，而不应该留下上面几个所谓的特殊字符，至少 `+` 和 `=` 会出现明显的 bug。

Angular 官方的维护者，也出来回复了，大致的意思是，不确定是否所有的后端服务器都能正确处理百分号形式的编码（即`+` 被转码为`%2B` 这种格式的编码），所以不敢贸然都改了。至于为什么会有在转码时，留下几个特殊字符不进行转码，那个 Angular 维护者给出的回复是，这段代码是从 AngularJS1.0 就这么些的，至今已运行了 10 年之久，根本原因，其实他也不知道为啥这么写，反正就是从 AngularJS1.0 开始就这么写的，贸然改动不确定会不会出现更大的问题。

> 这个回复，我真的也是醉了。就因为这段代码从 AngularJS1.0 就存在了，即使出现 bug，有很多人中招了，官方也不进行更改。无语。。。

## 意外收获

在翻阅整个 issue 讨论的过程中，我发现 Angular 官方在合并“修复”代码时用的标题是 “don't encode values that are allowed in query”，意思是说，这几个特殊字符在 querystring 里是合法的字符，被允许的字符，就不被转码了。

这是什么意思，为什么说 `+` 是 querystring 允许的合法字符，原来 `+` 会被解码成空格。

### 空格的转换

空格在进行 URLEncode 时有两种规范：

⓵\[RFC 3986]：

```text
A percent-encoding mechanism is used to represent a data octet in a
component when that octet's corresponding character is outside the
allowed set or is being used as a delimiter of, or within, the
component.  A percent-encoded octet is encoded as a character
triplet, consisting of the percent character "%" followed by the two
hexadecimal digits representing that octet's numeric value.  For
example, "%20" is the percent-encoding for the binary octet
"00100000" (ABNF: %x20), which in US-ASCII corresponds to the space
character (SP).  Section 2.4 describes when percent-encoding and
decoding is applied.
```

**\[RFC 3986]** 明确规定了**空格 会被百分号编码为`%20`**

⓶\[RFC 1866]:

```纯文本
The default encoding for all forms is `application/x-www-form-
   urlencoded'. A form data set is represented in this media type as
   follows:

        1. The form field names and values are escaped: space
        characters are replaced by `+', and then reserved characters
        are escaped as per [URL]
```

这里要求`application/x-www-form-urlencoded`类型的消息中，空格要被替换为`+`,其他字符按照\[URL]中的定义来转义，其中的\[URL]指向的是[RFC 1738](https://www.rfc-editor.org/rfc/rfc1738 "RFC 1738") 而它的修订版中和 URL 有关的最新文档恰恰就是 **\[RFC 3986]**。

这也是为什么在很多文档中的描述中，空格的百分号编码结果是 `+` 或 `%20` 。

[w3schools](https://www.w3schools.com/tags/ref_urlencode.ASP "w3schools")： URLs cannot contain spaces. URL encoding normally replaces a space with a plus (+) sign or with %20. 【URL 不能包含空格。URL 编码通常使用加号（+）或 %20 替代空格。】

[MDN](https://developer.mozilla.org/en-US/docs/Glossary/percent-encoding "MDN")：Depending on the context, the character `' '` is translated to a `'+'` (like in the percent-encoding version used in an `application/x-www-form-urlencoded` message), or in `'%20'` like on URLs. 【根据上下文，空白符 `' '` 将会转换为 `'+'` （必须在 HTTP 的 POST 方法中使定义 `application/x-www-form-urlencoded` 传输方式），或者将会转换为 `'%20'` 的 URL。】

**注意**：MDN 中已说明，空格被转换为 `+` 必须是在 HTTP 的 POST 方法中使用 `application/x-www-form-urlencoded` 进行传输，其他情况下，均转换为 `%20` 。

> 在 StackOverflow 上也有人问，[空格](https://stackoverflow.com/questions/2678551/when-should-space-be-encoded-to-plus-or-20 "空格")[ ](https://stackoverflow.com/questions/2678551/when-should-space-be-encoded-to-plus-or-20 " ")[ 到底在什么时候会被转码为 ](https://stackoverflow.com/questions/2678551/when-should-space-be-encoded-to-plus-or-20 " 到底在什么时候会被转码为 ")[+](https://stackoverflow.com/questions/2678551/when-should-space-be-encoded-to-plus-or-20 "+")[ ，什么时候会被转码为 ](https://stackoverflow.com/questions/2678551/when-should-space-be-encoded-to-plus-or-20 " ，什么时候会被转码为 ")[%20](https://stackoverflow.com/questions/2678551/when-should-space-be-encoded-to-plus-or-20 "%20")[ ](https://stackoverflow.com/questions/2678551/when-should-space-be-encoded-to-plus-or-20 " ")
> 点赞较多的回答如 MDN 中所说， 空格只有在 `application/x-www-form-urlencoded` 才会被转码为 `+` ，如果在 url 中表示 query 参数，则应被转换为 `%20` 。有兴趣的，可以去看一下大家的讨论。

> 🔔Angular 的转码器，默认不对 `+` 等字符进行转码，估计就是因为上面这个规范，因为 `+` 也是一个合法的转码后的字符。但实际上正如规范所说，`+` 作为转码字符，只有在 HTTP 的 POST 方法中使用 `application/x-www-form-urlencoded` 进行传输时，其作为空格字符的转码。
> 所以，可以明确，Angular 中默认未对 `+` 进行转码是错误的。
> 从 github 上的 issue 的讨论也可以看出，他们对为什么会有这段代码十分含糊。

### encodeURI 和 encodeURIComponent

在整个过程中，我发现 js 的两个转码函数： `encodeURI()` 和 `encodeURIComponent()` 在进行转码时结果是不同的，当时在猜想，Angular 的这种默认方式，会不会和 `encodeURI()` 有关，因为 `encodeURI()`并不会对 `+` 进行转码，研究了一下发现并不是，因为如果是的话，源码里就直接用 `encodeURI()` 函数了，而不是用 `encodeURIComponent()` 函数，然后再将几个特殊字符替换回去。

而且这几个字符，只是保留字符，像下面的非转义字符和数字符号，都没替换回去。

---

- encodeURI
  - `encodeURI()`函数无需对那些保留的并且在 URI 中有特殊意思的字符进行编码。
  - `encodeURI()` 会替换所有的字符，但不包括以下字符，即使它们具有适当的 UTF-8 转义序列：
    | 类型 | 包含 |
    | ------------ | --------------------------------------------- |
    | 保留字符 | `;` `,` `/` `?` `:` `@` `&` `=` `+` `$` |
    | 非转义的字符 | 字母 数字 `-` `_` `.` `!` `~` `*` `'` `(` `)` |
    | 数字符号 | `#` |
  - 请注意，`encodeURI` 自身*无法*产生能适用于 HTTP GET 或 POST 请求的 URI，例如对于 XMLHTTPRequests，因为 "&", "+", 和 "=" 不会被编码，然而在 GET 和 POST 请求中它们是特殊字符。

- encodeURIComponent
  - `encodeURIComponent()` 函数转义除了如下所示外的所有字符：
    不转义的字符：
    `A-Z a-z 0-9 - _ . ! ~ * ' ( )`

`encodeURIComponent()` 和 **`encodeURI()`** 有以下几个不同点：

```javascript
var set1 = ";,/?:@&=+$"; // 保留字符
var set2 = "-_.!~*'()"; // 不转义字符
var set3 = "#"; // 数字标志
var set4 = "ABC abc 123"; // 字母数字字符和空格

console.log(encodeURI(set1)); // ;,/?:@&=+$
console.log(encodeURI(set2)); // -_.!~*'()
console.log(encodeURI(set3)); // #
console.log(encodeURI(set4)); // ABC%20abc%20123 (空格被编码为 %20)

console.log(encodeURIComponent(set1)); // %3B%2C%2F%3F%3A%40%26%3D%2B%24
console.log(encodeURIComponent(set2)); // -_.!~*'()
console.log(encodeURIComponent(set3)); // %23
console.log(encodeURIComponent(set4)); // ABC%20abc%20123 (空格被编码为 %20)
```

为了避免服务器收到不可预知的请求，对任何用户输入的作为 URI 部分的内容你都需要用 encodeURIComponent 进行转义。比如，一个用户可能会输入"`Thyme &time=again`"作为`comment`变量的一部分。如果不使用 encodeURIComponent 对此内容进行转义，服务器得到的将是`comment=Thyme%20&time=again`。请注意，"&"符号和"="符号产生了一个新的键值对，所以服务器得到两个键值对（一个键值对是`comment=Thyme`，另一个则是`time=again`），而不是一个键值对。

对于 [application/x-www-form-urlencoded](https://www.whatwg.org/specs/web-apps/current-work/multipage/association-of-controls-and-forms.html#application/x-www-form-urlencoded-encoding-algorithm "application/x-www-form-urlencoded") (POST) 这种数据方式，空格需要被替换成 '+'，所以通常使用 `encodeURIComponent` 的时候还会把 "%20" 替换为 "+"。

为了更严格的遵循 [RFC 3986](https://datatracker.ietf.org/doc/html/rfc3986 "RFC 3986")（它保留 !, ', (, ), 和 \*），即使这些字符并没有正式划定 URI 的用途，下面这种方式是比较安全的：

```javascript
function fixedEncodeURIComponent(str) {
  return encodeURIComponent(str).replace(/[!'()*]/g, function (c) {
    return "%" + c.charCodeAt(0).toString(16).toUpperCase();
  });
}
```

以上部分摘自[MDN 文档-encodeURIComponent()](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/encodeURIComponent "MDN文档-encodeURIComponent()")，有兴趣的可以去看看文档。

---

> 🔔 如上 MDN 文档说的，对于任何用户数组的，作为 URI 部分的内容，都应该需要使用 `encodeURIComponent()` 进行转码，这样服务器端才能正确处理数据。
> 这也同样印证了上面 Angular 默认转码器未对 `+` 等字符转码的做法是错误的。
