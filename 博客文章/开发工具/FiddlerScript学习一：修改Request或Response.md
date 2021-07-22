---
title: FiddlerScript学习一：修改Request或Response
date: 2014-09-26 16:04:26
tags: [fiddler,调试,web开发]
categories:
- 技术博客
- 原创
---

前两天因项目需要，简单看了一下FiddlerScript，功能挺强的，今天有时间仔细看一下，做个笔记。
修改Request或Response
修改Request和Response要在FiddlerScript中的OnBeforeRequest和OnBeforeResponse函数中添加规则即可。OnBeforeRequest函数是在每次请求之前调用，OnBeforeResponse函数是在每次响应之前调用。

<!-- more -->

#### 添加请求头Header
```javascript
oSession.oRequest["NewHeaderName"] = "New header value";
```

#### 删除Response的Header
```javascript
oSession.oResponse.headers.Remove("Set-Cookie");
```

#### 将请求从一个页面转发到同一Server上的另一页面
```javascript
if (oSession.PathAndQuery=="/hello/hello.html") {
      oSession.PathAndQuery="/hello/index.html";
    }
```

注意：oSession.PathAndQuery的值为fiddler中session列表中的Url：
![](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/20140929154209755.jpeg)

即图中红色标注出来的部分。图中黄色标注出来的部分有点特殊，host为Tunnel to ,url为另一host。查看该请求的Header为：
![](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/20140929154512744.jpeg)

这种特殊情况会在下面还有例子。
上面的例子，拦截请求地址为/hello/hello.html的请求，并将其转发到相同Server的/hello/index.html
#### 将请求转发到相同端口号的不同服务器（修改请求的Host）
```javascript
if(oSession.HostnameIs("www.baidu.com")){
      oSession.hostname = "www.sina.com.cn";
}
```
这个例子是将发送到百度的请求转发到新浪，则会提示页面不存在。这里只是改变了host，并不改变后面的地址，因此，如果在新浪上不存在相应的页面。如下面图片所示：
![1](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/20140929152403525.jpeg)

如果我访问的是如下地址：http://www.baidu.com/link?url=CQuVpjo9u9UQADcstwECPEmrziPMk5u5H9PlRN2TbWLkKZaxafVER2X8OEYzovr-yasX2Fwcgj0NANBtKVj0gN78jNJ3bXTmIsTeBk7hXem
则结果如下：（该页面实际是存在的，是百度搜索出来的结果页面，被fiddler转发到新浪，但是新浪上不存的此页面）
![](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/20140929152504443.jpeg)

#### 将请求转发到不同端口号，不同Server
```javascript
if (oSession.host=="192.168.0.70:8080") {
  oSession.host="192.168.0.69:8020";
}
```
这个例子是将发送到192.168.0.70:8080的请求转发到192.168.0.69:8020，这里只是改变host，并不改变后面的请求地址。例如，做以上的规则后，我请求的是：
http://192.168.0.70:8080/hello/hello.html
![](https://img001-10042971.cos.ap-shanghai.myqcloud.com/blog/20140929153525496.jpeg)

而实际我项目部署到的是：192.168.0.69:8020

#### 将所有请求从一个服务器转发到另一个服务器，包括Https
```javascript
// Redirect traffic, including HTTPS tunnels
if (oSession.HTTPMethodIs("CONNECT") && (oSession.PathAndQuery == "www.example.com:443")) {
    oSession.PathAndQuery = "beta.example.com:443";
}

if (oSession.HostnameIs("www.example.com")) oSession.hostname = "beta.example.com";
```
#### Simulate the Windows HOSTS file, by pointing one Hostname to a different IP address. (Retargets without changing the request's Host header)

```javascript
// All requests for subdomain.example.com should be directed to the development server at 128.123.133.123
if (oSession.HostnameIs("subdomain.example.com")){
oSession.bypassGateway = true;                   // Prevent this request from going through an upstream proxy
oSession["x-overrideHost"] = "128.123.133.123";  // DNS name or IP address of target server
}
```
#### Retarget requests for a single page to a different page, potentially on a different server. (Retargets by changing the request's Host header)
```javascript
if (oSession.url=="www.example.com/live.js") {
  oSession.url = "dev.example.com/workinprogress.js";
}
```

#### Prevent upload of HTTP Cookies
```javascript
oSession.oRequest.headers.Remove("Cookie");
```

#### Decompress and unchunk a HTTP response, updating headers if needed
```javascript
// Remove any compression or chunking from the response in order to make it easier to manipulate
oSession.utilDecodeResponse();
```

#### Search and replace in HTML.
```javascript
if (oSession.HostnameIs("www.bayden.com") && oSession.oResponse.headers.ExistsAndContains("Content-Type","text/html")){
  oSession.utilDecodeResponse();
  oSession.utilReplaceInResponse('<b>','<u>');
}
```

#### Case insensitive Search of response HTML.
```javascript
if (oSession.oResponse.headers.ExistsAndContains("Content-Type", "text/html") && oSession.utilFindInResponse("searchfor", false)>-1){
  oSession["ui-color"] = "red";
}
```

#### Remove all DIV tags (and content inside the DIV tag)
```javascript
// If content-type is HTML, then remove all DIV tags
if (oSession.oResponse.headers.ExistsAndContains("Content-Type", "html")){
  // Remove any compression or chunking
  oSession.utilDecodeResponse();
  var oBody = System.Text.Encoding.UTF8.GetString(oSession.responseBodyBytes);

  // Replace all instances of the DIV tag with an empty string
  var oRegEx = /<div[^>]*>(.*?)<\/div>/gi;
  oBody = oBody.replace(oRegEx, "");

  // Set the response body to the div-less string
  oSession.utilSetResponseBody(oBody);
}
```

#### Pretend your browser is the GoogleBot webcrawler
```javascript
oSession.oRequest["User-Agent"]="Googlebot/2.X (+http://www.googlebot.com/bot.html)";
```

#### Request Hebrew content
```javascript
oSession.oRequest["Accept-Language"]="he";
```

#### Deny .CSS requests
```javascript
if (oSession.uriContains(".css")){
 oSession["ui-color"]="orange";
 oSession["ui-bold"]="true";
 oSession.oRequest.FailSession(404, "Blocked", "Fiddler blocked CSS file");
}
```

#### Simulate HTTP Basic authentication (Requires user to enter a password before displaying web content.)
```javascript
if ((oSession.HostnameIs("www.example.com")) &&
 !oSession.oRequest.headers.Exists("Authorization"))
{
// Prevent IE's "Friendly Errors Messages" from hiding the error message by making response body longer than 512 chars.
var oBody = "<html><body>[Fiddler] Authentication Required.<BR>".PadRight(512, ' ') + "</body></html>";
oSession.utilSetResponseBody(oBody);
// Build up the headers
oSession.oResponse.headers.HTTPResponseCode = 401;
oSession.oResponse.headers.HTTPResponseStatus = "401 Auth Required";
oSession.oResponse["WWW-Authenticate"] = "Basic realm=\"Fiddler (just hit Ok)\"";
oResponse.headers.Add("Content-Type", "text/html");
}
```

#### Respond to a request with a file loaded from the \Captures\Responses folder (Can be placed in OnBeforeRequest or OnBeforeResponse function)
```javascript
if (oSession.PathAndQuery=="/version1.css") {
  oSession["x-replywithfile"] ="version2.css";

```

以上例子我并没有都实践，只实践了中间几个地址转发的，因为现在需要用。剩下的请大家有需要的自己实践吧。
