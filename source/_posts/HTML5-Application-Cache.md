title: "HTML5 应用程序缓存"
date: 2014-12-01 12:21:35
tags:
- HTML5
- Cache
categories: 
- 前端

---

## 简介

HTML5提供一种缓存机制，可以使Web应用程序可以离线运行。在处于离线状态时，即使用户点击刷新按钮，应用也能正常工作。此前，Google曾通过Google Gears来做同样的事，但由于HTML5的到来，Google已经放弃为Gears增加新功能。并且于2011年底前撤出Google产品线而不再发行。

使用应用缓存可以得到以下益处：

- 离线浏览: 用户可以在离线状态下浏览网站内容。
- 更快的速度: 因为数据被存储在本地，所以速度会更快.
- 减轻服务器的负载: 浏览器只会下载在服务器上发生改变的资源。

## 如何使用

### 创建一个清单文件

一个示例的清单文件如下：

```
CACHE MANIFEST
CACHE:
# Defines resources to be cached.
script/library.js
css/stylesheet.css
images/figure1.png

FALLBACK:
# Defines resources to be used if non-cached
# resources cannot be downloaded, for example
# when the browser is offline..
photos/ figure2.png

NETWORK:
# Defines resources that will not be cached.
api/
```

清单文件分为三节：

- CACHE：指定被缓存的资源。
- FALLBACK：指定备用资源，每一行有两个URL，若前面的URL资源无法访问，则后面的资源会被使用。作为备用资源，后面的URL资源会被缓存。可以使用通配符*。
- NERWORK：不缓存的资源，可以使用通配符*。
清单文件可以包含任意个以上的节，但新的节必须以节标题开头后加冒号。如果没有使用节标题，则默认为CACHE节。

**清单文件必须使用UTF-8编码。**

**服务器在发送清单文件时必须使用 text/cache-manifest 的MIME Type。**

如PHP代码：

```php
header('Content-Type: text/cache-manifest');
```

或在Apache服务器中配置：

```apache
AddType text/cache-manifest .appcache
```

**清单文件必须以CACHE MANIFEST开头。**

**清单文件的注释以#开头。**

**清单文件中的URL若使用相对路径，意思是资源相对于清单文件的路径。**

### 在要使用缓存的每个网页中引用该清单文件

一个引用appcache.manifest.php清单文件的网页示例：

```html
<!DOCTYPE html>
<html lang="en" manifest="appcache.manifest.php">
    <head>
        <meta charset="UTF-8">
        <title>Document</title>
    </head>
    <body>
    </body>
</html>
```

appcache.manifest.php

```php
<?php
    header('Content-Type: text/cache-manifest');
?>CACHE MANIFEST
# v1 - 2014-12-1
# This is a comment.
appcache.php

# Use from network if available
NETWORK:
network.html

# Fallback content
FALLBACK:
api/* fallback.php
```

**没有必要在清单文件中声明引用该清单文件的网页，它会默认被缓存。**

## 三、技巧

在 Chrome 中，你可以在设置中选择 「清除浏览器数据...」 或访问 [chrome://appcache-internals/](chrome://appcache-internals/) 来清除缓存。