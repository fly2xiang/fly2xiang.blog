title: "Nginx 多规则替换模块 ngx_http_substitutions_filter_module"
date: 2014-12-19 21:27:57
tags:
- Nginx
- 替换
categories: 
---

## 简介

Nginx作为反向代理时有一个`ngx_http_sub_module`模块可以修改响应的内容: <http://nginx.org/en/docs/http/ngx_http_sub_module.html>，但缺点是只能使用一条规则。而`ngx_http_substitutions_filter_module`这个第三方的模块可以做到多条规则。

## 安装

### 下载ngx_http_substitutions_filter_module

需要使用git下载安装包，可以使用`sudo yum install git安装`或直接到<https://github.com/yaoweibin/ngx_http_substitutions_filter_module>下载最新的源码包。

```
git clone git://github.com/yaoweibin/ngx_http_substitutions_filter_module.git

```

### 安装Nginx

安装需要依赖pcre(PERL 5 regular expression pattern matching): <http://sourceforge.net/projects/pcre/files/pcre/>和zlib: <http://sourceforge.net/projects/libpng/files/zlib/>，这里事先下载好并解压。

```
wget http://nginx.org/download/nginx-1.7.8.tar.gz
tar zxvf nginx-1.7.8.tar.gz
cd nginx-1.7.8
./configure --with-pcre=../pcre-8.36 --with-zlib=../zlib-1.2.8 --add-module=../ngx_http_substitutions_filter_module/
make
sudo make install
```
上面configure命令中 ../pcre-8.36 ../zlib-1.2.8 ../ngx_http_substitutions_filter_module/分别是pcre、zlib与ngx_http_substitutions_filter_module的源码路径。

Nginx被默认安装在/usr/local/nginx/目录。

## 使用

模块只有两条指令：

**subs_filter_types**

语法: subs_filter_types mime-type [mime-types]

默认: subs_filter_types text/html

适用范围: http, server, location

subs_filter_types指令指定需要替换内容的MIME类型，默认是text/html类型。

**需要注意的是**：如果服务器会对内容进行了gzip压缩，则要加入以下指令使服务器返回未经压缩的内容才能起作用：

```
proxy_set_header Accept-Encoding "";
```

**subs_filter**

语法: subs_filter 源字符串 替换为的字符串 [gior]

默认: none

使用范围: http, server, location

subs_filter使用直接替换或者正则匹配替换的方式替换。

第三个参数的含义是：

g(默认): 替换所有匹配的字符串

i：区分大小写

o：仅替换首个匹配的字符串

r：使用正则匹配

## 例子

```
location / {

    proxy_pass http://example.com/; #反向代理
    proxy_set_header Accept-Encoding ""; #防止后端服务器在返回gzip后的内容时模块不起作用
    subs_filter_types text/html text/css text/xml; #替换html、css、xml内容
    subs_filter st(\d*).example.com $1.example.com ir; #使用正则替换
    subs_filter a.example.com s.example.com; #使用直接匹配替换

}
```

## 相似的模块

Nginx Module for Google: <https://github.com/cuber/ngx_http_google_filter_module>

参考链接：

> nginx_substitutions_filter: <https://code.google.com/p/substitutions4nginx/>

> ngx_http_substitutions_filter_module: <https://github.com/yaoweibin/ngx_http_substitutions_filter_module>

