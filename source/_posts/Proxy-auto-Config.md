title: "代理自动配置 Proxy auto-Config"
date: 2014-12-02 11:28:23
tags:
- 代理
categories: 
---

## 简介

代理自动配置（PAC）是一种浏览器技术，它用来告诉浏览器如何选择适当的代理服务器。

## 简单使用

在IE浏览器中可以在“Internet 选项” -> “连接”选项卡 -> “局域网设置”中看到：

![](https://github.com/fly2xiang/fly2xiang_blog/raw/master/images/pac.png)
    

PAC一般使用一个proxy.pac文件作为配置，若使用http服务器提供pac文件，建议使用的MIME是`application/x-ns-proxy-autoconfig`。

pac文件中其实是javascript代码，其中包含一个重要的函数：

```javascript
function FindProxyForURL(url, host);
```

浏览器会将请求的URL与主机名传入到这个函数进行查询，函数FindProxyForURL返回一个包含代理服务器信息的字符串，浏览器根据这个字符串使用对应的代理服务器链接网络。

一个简单的pac文件如下：

```javascript
function FindProxyForURL(url, host) {
    return "PROXY proxy.example.com:8080; DIRECT";
}
```
在这个文件中，所有的网络访问都会使用`proxy.example.com:8080`代理，若这个代理不可用，则会直接连接（DIRECT）。

## 函数列表

在pac文件中可以使用的其他javascript函数如下：

`dnsDomainIs` 若host匹配google.com例如map.google.com等，则直接连接：

```javascript
if (dnsDomainIs(host, ".google.com"))
    return "DIRECT";
```

`shExpMatch` 若url以.local结尾或在domain.com/folder/目录下则直接连接：

```javascript
if (shExpMatch(url, "*.local") || shExpMatch(url, "http://domain.com/folder/*"))
    return "DIRECT";
```

`dnsResolve` DNS反查IP：

`isInNet` 若IP在127.16.0.0/12子网内则直接访问：

```javascript
if (isInNet(dnsResolve(host), "172.16.0.0", "255.240.0.0"))
    return "DIRECT";
```

`myIpAddress` 返回我当前的IP

```javascript
if (isInNet(myIpAddress(), "10.10.1.0", "255.255.255.0"))
    return "PROXY 10.10.5.1:8080";
```

`isPlainHostName` 若host中不包含“.”则直接访问：

```javascript
if (isPlainHostName(host))
    return "DIRECT";
```

`localHostOrDomainIs`

```javascript
if (localHostOrDomainIs(host, "www.google.com"))
    return "DIRECT";
```

`isResolvable` 若DNS可以被反查则使用代理：

```javascript
if (isResolvable(host))
    return "PROXY proxy1.example.com:8080";
```

`dnsDomainLevels` host中“.”的个数大于0则使用代理：

```javascript
if (dnsDomainLevels(host) > 0)
    return "PROXY proxy1.example.com:8080";
else
    return "DIRECT";
```

`weekdayRange` 周一到周五使用代理：

if (weekdayRange("MON", "FRI"))
    return "PROXY proxy1.example.com:8080";
else
    return "DIRECT";
```

`dateRange` 一月到三月使用代理：

```javascript
if (dateRange("JAN", "MAR"))
    return "PROXY proxy1.example.com:8080";
else
    return "DIRECT";
```

`timeRange` 8:00到18:00使用代理：

```javascript
if (timeRange(8, 18))
    return "PROXY proxy1.example.com:8080";
else
    return "DIRECT";
```

`alert` 函数并没有在PAC规范中指定，但IE与FireFox是支持的，用于调试：

```javascript
resolved_host = dnsResolve(host);
alert(resolved_host);
```

## 高级应用

一个复杂的pac文件示例：

```javascript
function FindProxyForURL(url, host) {

    // 一些不使用代理的域名
    if (dnsDomainIs(host, ".intranet.domain.com") || shExpMatch(host, "(*.abcdomain.com|abcdomain.com)"))
        return "DIRECT";

    // 对于FTP和abcdomain.com/folder/下的请求不使用代理

    if (url.substring(0, 4)=="ftp:" || shExpMatch(url, "http://abcdomain.com/folder/*"))
        return "DIRECT";

    // 局域网中的访问不使用代理
    if (isPlainHostName(host) ||
        shExpMatch(host, "*.local") ||
        isInNet(dnsResolve(host), "10.0.0.0", "255.0.0.0") ||
        isInNet(dnsResolve(host), "172.16.0.0", "255.240.0.0") ||
        isInNet(dnsResolve(host), "192.168.0.0", "255.255.0.0") ||
        isInNet(dnsResolve(host), "127.0.0.0", "255.255.255.0"))
        return "DIRECT";

    // 如果我当前的IP地址在10.10.5.0/24子网内则使用代理
    if (isInNet(myIpAddress(), "10.10.5.0", "255.255.255.0"))
        return "PROXY 1.2.3.4:8080";

    // 默认的，使用下面的两个代理做负载均衡
    return "PROXY 4.5.6.7:8080; PROXY 7.8.9.10:8080";
}
```

## 注意事项

有些浏览器，例如Firefox和Internet Explorer只支持系统缺省编码的PAC文件，不支持Unicode编码的PAC文件，例如UTF-8编码的PAC文件。

**函数dnsResolv(及其他类似函数)在执行DNS查询时，如果DNS服务器没有回应，这个会导致你的浏览器被阻塞很长时间。**