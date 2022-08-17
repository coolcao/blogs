## 目录

*   [优势](#优势)

*   [劣势](#劣势)

*   [Keep-Alive模式，客户端如何判断请求所得到的响应数据已经接收完成](#keep-alive模式客户端如何判断请求所得到的响应数据已经接收完成)

*   [分块编码(Transfer-Encoding: chunked)](#分块编码transfer-encoding-chunked)

# HTTP keep-alive

**HTTP持久连接**是使用同一个TCP连接来发送和接收多个HTTP请求/应答，而不是为每一个新的请求和应答打开新的连接的方法。

在HTTP 1.0中，没有官方的keepalive操作。通常是在现有协议上添加一个指数。如果浏览器支持keep-alive，它会在请求的包头中添加：

```bash
Connection: Keep-Alive
```

然后当服务器收到请求，作出回应的时候，它也添加一个头在响应中：

```bash
Connection: Keep-Alive
```

这样做，连接就不会中断，而是保持连接。当客户端发送另一个请求时，它会使用同一个连接。这一直继续到客户端或服务器端认为会话已经结束，其中一方中断连接。

**在HTTP 1.1中所有的连接默认都是持续连接，除非特殊声明不支持。** HTTP持久连接不使用独立的keepalive信息，而是仅仅允许多个请求使用单个连接。然而，Apache 2.0 httpd的默认连接过期时间仅有15秒，Apache 2.2仅有5秒。短的过期时间的优点是能够快速的传输多个web页组件，而不会绑定多个服务器进程或线程太长时间。

## 优势

*   较少的[CPU](https://zh.wikipedia.org/wiki/CPU "CPU")和内存的使用（由于同时打开的连接的减少了）

*   允许请求和应答的[HTTP流水线](https://zh.wikipedia.org/wiki/HTTP管線化 "HTTP流水线")

*   降低[拥塞控制](https://zh.wikipedia.org/wiki/拥塞控制 "拥塞控制") （[TCP连接](https://zh.wikipedia.org/wiki/传输控制协议 "TCP连接")减少了）

*   减少了后续请求的[延迟](https://zh.wikipedia.org/wiki/延遲_\(電腦\) "延迟")（无需再进行[握手](https://zh.wikipedia.org/wiki/握手_\(技术\) "握手")）

*   报告错误无需关闭TCP连接

根据RFC 2616 （47页），用户客户端与任何服务器和代理服务器之间不应该维持超过2个链接。[代理服务器](https://zh.wikipedia.org/wiki/代理服务器 "代理服务器")应该最多使用2×N个持久连接到其他服务器或代理服务器，其中N是同时活跃的用户数。这个指引旨在提高HTTP响应时间并避免阻塞。

## 劣势

对于现在的广泛普及的宽带连接来说，Keep-Alive也许并不像以前一样有用。web服务器会保持连接若干秒（Apache中默认15秒），这与提高的性能相比也许会影响性能。

对于单个文件被不断请求的服务（例如图片存放网站），Keep-Alive可能会极大的影响性能，因为它在文件被请求之后还保持了不必要的连接很长时间。

## Keep-Alive模式，客户端如何判断请求所得到的响应数据已经接收完成

对于非持续性连接，客户端可以通过连接是否关闭来界定请求或响应的边界。而对于持续性连接，这种方法是不生效的。

1.  使用消息首部字段 `Content-Length`

    `Content-Length` 表示实体内容长度，当客户端向服务器端发送一个静态页面或一张图片时，服务器可以很清楚的知道内容大小，然后通过 `Content-Length` 消息首部字段高速客户端响应的大小。

2.  使用消息首部字段 `Transfer-Encoding`

    如果是动态页面大小，服务器不可能预先知道响应的大小。如果要准确获取响应的大小，只能开一个足够大的buffer，等内容全部生成好再进行计算，这样会影响性能，也会影响客户端等待时间。

    这时可以使用 `Transfer-Encoding：chunked` 模式，服务器就需要使用 `Transfer-Encoding：chunked` 这样的方式来一边生产数据，一边发送给客户端。

## 分块编码(Transfer-Encoding: chunked)

1.  Transfer-Encoding，是一个 HTTP 头部字段（响应头域），字面意思是「传输编码」。最新的 HTTP 规范里，只定义了一种编码传输：分块编码(chunked)。

2.  分块传输编码（Chunked transfer encoding）是超文本传输协议（HTTP）中的一种数据传输机制，允许HTTP由网页服务器发送给客户端的数据可以分成多个部分。分块传输编码只在HTTP协议1.1版本（HTTP/1.1）中提供。

3.  数据分解成一系列数据块，并以一个或多个块发送，这样服务器可以发送数据而不需要预先知道发送内容的总大小。

4.  具体方法

    1.  在头部加入 `Transfer-Encoding: chunked` 之后，就代表这个报文采用了分块编码。这时，报文中的实体需要改为用一系列分块来传输。

    2.  每个分块包含十六进制的长度值和数据，长度值独占一行，长度不包括它结尾的 CRLF(\r\n)，也不包括分块数据结尾的 CRLF。

    3.  **最后一个分块长度值必须为 0，对应的分块数据没有内容，表示实体结束。**

5.  `Content-Encoding` 和 `Transfer-Encoding` 二者经常会结合来用，其实就是针对 `Transfer-Encoding` 的分块再进行 `Content-Encoding`压缩。

```bash
HTTP/1.1 200 OK
Content-Type: text/plain
Transfer-Encoding: chunked

25\r\n
This is the data in the first chunk\r\n

1C\r\n
and this is the second one\r\n

3\r\n
con\r\n

8\r\n
sequence\r\n

0\r\n
\r\n
```
