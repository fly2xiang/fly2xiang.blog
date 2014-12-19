title: "HTTP 缓存实战"
date: 2014-11-30 15:41:55
tags: 
- HTTP Header
- Cache
- 缓存
categories: 
---

## 使用缓存的好处

- 降低服务器负载，减少带宽费用
- 加快页面加载速度，提升用户体验

## 相关的HTTP Header

### 在HTTP 1.0 中使用的：

```
Request：If-Modified-Since: Tue, 02 Sep 2014 09:12:36 GMT
Response：Last-Modified: Sun, 30 Nov 2014 05:14:45 GMT
```

这对HTTP头协作来控制客户端缓存是否是最新的。服务器根据资源的Last-Modified是否小于或等于请求头中的If-Modified-Since匹配来决定使用缓存还是返回新的内容。

```
Response：Expires: Sun, 30 Nov 2014 05:15:45 GMT
```

在直接输入地址时，若资源未到Expires指定的过期时间，则直接使用缓存，不去服务器验证。但在刷新（F5）时还是要去服务器端验证是否过期。

```
Pragma: no-cache
```

告诉浏览器不使用缓存

**这种缓存控制的缺点是需要时间同步，若时间不同步则缓存可能不像预期的那样工作。**

### 在HTTP 1.1中使用的：

`Response：Cache-Control: `取值有以下几种

public

private

public与private的区别在于代理服务器缓存的处理。public意味着每个使用代理服务器的用户看到的是相同的内容，private意味着每个使用代理服务器的用户看到的内容是不同的。所以public是代理服务器会缓存的，而private的内容代理服务器不会缓存，只有浏览器会缓存。

no-cache

指示浏览器不要缓存内容

no-store

指示浏览器既不要缓存，也不要存储，用于机密资料，防止泄漏

no-transform

非透明代理会对内容进行转码（如Opera Mini， UC，百度Wap搜索会对页面的内容及图像进行转码方便移动浏览器浏览，对图片进行压缩节省流量），no-transform告诉这些非透明代理不要对内容进行转码（对于医疗成像、科学数据分析、端到端认证等特别有用）。

must-revalidate

指示所有的缓存都必须重新验证，在这个过程中，浏览器会发送一个If-Modified-Since头。如果服务器程序验证得出当前的响应数据为最新的数 据，那么服务器应当返回一个304  Not Modified响应给客户端，否则响应数据将再次被发送到客户端。

proxy-revalidate

与6相同，只不过用于代理服务器缓存

max-age=时间

数据经过max-age设置的秒数后就会失效，相当于HTTP/1.0中的Expires头。如果在一次响应中同时设置了max-age和 Expires，那么max-age将具有较高的优先级。

s-maxage=时间

与8相同，只不过用户代理服务器缓存

###ETag验证

```
Request ：If-None-Match: "ddb2-5021185338900"
```

服务器根据资源的ETag是否与请求头中的If-None-Match匹配来决定使用缓存还是返回新的内容

```
Response：ETag:  "ddb2-5021185338900"
```

资源的ETag

**如果同时存在Cache-Control和Expires，浏览器总是优先使用Cache-Control，如果没有Cache-Control才考虑Expires。**

ETag解决了使用过期时间控制缓存的一些问题：

- 服务器时间准确度的问题
- 使用过期时间控制缓存只能控制在秒级别
- 修改时间改变但是内容没有改变的情况

## 三、技巧

使用Ctrl + F5刷新可以强制不使用缓存。

在浏览器中直接输入地址会直接使用缓存，F5刷新会去服务器验证文件是否是最新的。

 

参考资料：

> HTTP/1.1: Header Field Definitions ：<http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html>