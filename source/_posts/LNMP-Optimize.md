title: "LNMP 性能优化"
date: 2015-11-23 16:02:32
tags:
- PHP
- Nginx
- 性能
- 优化
categories: 
- PHP

---

## 结论

### 先说优化要点：

* Nginx `work_processes` 设置为CPU核心数
* Nginx 设置 `worker_cpu_affinity` CPU亲和性绑定 Nginx 进程
* Nginx `worker_rlimit_nofile` 设置为较大的数，可应对较大的并发量，这里设置为 102400
* Nginx `worker_connections` 设置为较大的数，以应对高并发，这里设置为 102400
* Nginx 使用 `epoll` 提高性能
* Nginx 连接后端 `php-fpm` 使用 `unixsocket` 方式，比使用 `tcp` 方式性能好太多（ `tcp` 方式在高并发下稳定较 `unixsocket` 方式好，但在高并发下性能太差）
* Nginx 使用 `upstream` 连接到后端的多个 `php-fpm` 实例中，每个 `php-fpm` 实例监听一个 `unixsocket`，可提高稳定性，建议 `php-fpm` 实例数与CPU核心数相同（使用 Apache ab 进行压测，在高并发下错误为0）
* php-fpm 监听 `unixsocket` 文件放置在 `/dev/shm/` 下，使用内存文件系统，提高 io 性能
* php-fpm `pm.max_childred` 不宜设置的过大，CPU核心数的 1~2 倍即可，`pm` 设置为 `pm = dynamic`，`pm.start_servers` 设置为 `pm.max_childred` 的一般, `pm.min_spare_servers` 设置为 `pm.max_childred` 的 1/4, `pm.max_spare_servers` 设置与 `pm.max_childred` 相同
* php-fpm `pm.max_requests` 设置为一个较大的值，这里设置为 200000，可根据 `Apache ab` 测试中查看内存的占用率查看是否有内存泄漏情况，如果存在内存泄漏可调小该值
* php-fpm 打开 `slowlog`，将 `request_slowlog_timeout` 设置为 1，即在响应时间超过 1s 时就记入 `slowlog`，一个响应迅速的网站应当在毫秒级别响应用户请求
* php-fpm 设置 `rlimit_files` 为一个较大值，这里设置为 102400
* php 打开 `opcache` 可显著提高性能，PHP 文件包含会导致性能降低，打开 `opcache` 能够显著提升性能
* php 使用 `redis` 作为 session 存储，PHP 默认使用文件作为 session 存储，性能较差（此处可在打开了 php-fpm `slowlog` 时进行高并发压测，可以看到 `session_start` 函数调用影响了性能）
* redis 使用多实例，redis 为单线程，不能利用多CPU核心，使用多个实例能够提高性能（此处实例数为 10 个）

### 配置

配置文件（服务器配置为 16核32G内存）：

nginx.conf:

```
worker_processes  16;
worker_cpu_affinity 0000000000000001 0000000000000010 0000000000000100 0000000000001000 0000000000010000 0000000000100000 0000000001000000 0000000010000000 0000000100000000 0000001000000000 0000010000000000 0000100000000000 0001000000000000 0010000000000000 0100000000000000 1000000000000000;
worker_rlimit_nofile 102400;

error_log  logs/error.log  notice;

events {
    use epoll;
    worker_connections  102400;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    client_max_body_size 6m;

    sendfile        on;
    keepalive_timeout  65;

    fastcgi_connect_timeout 300;
    fastcgi_send_timeout 300;
    fastcgi_read_timeout 300;
    fastcgi_buffer_size 256k;
    fastcgi_busy_buffers_size 256k;
    fastcgi_temp_file_write_size 1024k;
    fastcgi_intercept_errors on;
    fastcgi_buffers 8 256k;

    upstream php-fpm {
        server unix:/dev/shm/php-fpm1.sock;
        server unix:/dev/shm/php-fpm2.sock;
        server unix:/dev/shm/php-fpm3.sock;
        server unix:/dev/shm/php-fpm4.sock;
        server unix:/dev/shm/php-fpm5.sock;
        server unix:/dev/shm/php-fpm6.sock;
        server unix:/dev/shm/php-fpm7.sock;
        server unix:/dev/shm/php-fpm8.sock;
        server unix:/dev/shm/php-fpm9.sock;
        server unix:/dev/shm/php-fpm10.sock;
        server unix:/dev/shm/php-fpm11.sock;
        server unix:/dev/shm/php-fpm12.sock;
        server unix:/dev/shm/php-fpm13.sock;
        server unix:/dev/shm/php-fpm14.sock;
        server unix:/dev/shm/php-fpm15.sock;
        server unix:/dev/shm/php-fpm16.sock;
    }

    include vhosts/*;
}
```

vhosts/default.conf

```
server {
    listen       80;
    server_name  localhost;
    charset      utf-8;
    root         /data/www/default/;

    location / {
        index  index.html index.htm index.php;
    }

    location ~ \.php$ {
        fastcgi_pass   php-fpm;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        include        fastcgi_params;
    }
}
```

php-fpm.conf 

配置多个 php-fpm 实例，配置文件中的 pid，error_log，listen，slowlog 都要相应修改

```
[global]
pid = run/php-fpm1.pid
error_log = log/php-fpm1.log
log_level = notice

[www]
listen = /dev/shm/php-fpm1.sock
pm = dynamic
pm.max_children = 16
pm.start_servers = 8
pm.min_spare_servers = 4
pm.max_spare_servers = 16
pm.process_idle_timeout = 10s;
pm.max_requests = 20000

slowlog = log/$pool.1.log.slow
request_slowlog_timeout = 1 

rlimit_files = 102400
```

php-fpm 关键配置是启动多个实例，监听多个 `unixsocket` 文件。

php.ini（关键配置）

```
session.save_handler = redis
session.save_path = "tcp://127.0.0.1:6371, tcp://127.0.0.1:6372, tcp://127.0.0.1:6373, tcp://127.0.0.1:6374, tcp://127.0.0.1:6375, tcp://127.0.0.1:6376, tcp://127.0.0.1:6377, tcp://127.0.0.1:6378, tcp://127.0.0.1:6379"

[opcache]
zend_extension=opcache.so
opcache.enable=1
opcache.revalidate_freq=10

extension=redis.so
```

相应的，redis-server 应启动多个实例

## 调优经过

场景是一个推广的H5小游戏，游戏结束后可根据名次获得奖品。业务逻辑较为简单，但因需要在各大社交平台上推广，对性能要求较高，要求

* 可承受并发在 1500 左右
* 每秒请求数 1500 以上
* 错误数极少（个位数）
* 90% 请求在 200ms 内完成
* 平均响应时间 < 200 ms。

以上也是压力测试需要关注的几个点，压测时要同时关注，缺一不可。

首先根据网上搜索到的一些文章对 nginx 进行了配置：工作进程数、CPU 亲和性、连接数、打开文件限制等。对 php-fpm 使用了 /dev/shm 下的 unixsocket 进行监听，打开了两个 php-fpm 实例以防止高并发下的不稳定。此时使用 Apache ab 进行压测，发现并发能够完成测试，但错误请求 `Failed requests` 过多，达到 40% 左右。

想到 php-fpm 在高并发下不稳定，将 php-fpm 切换到监听本机 127.0.0.1 下的 9000 和 9001 端口进行测试，发现错误为 0，但并发数达不到要求。以上两种方式测试同时监控服务器 CPU、内存使用情况，发现在使用 unixsocket 进行通信时 CPU 占用在 30%，而使用 tcp 进行通信 CPU 会卡在 100% 一段时间。

最终开启 16 个 php-fpm 实例，能够在错误数为 0 的情况下完成测试，此时每秒处理请求数达不到要求。突然想到 php 在使用文件作为 session 存储时会在请求开始时锁定文件，请求结束后再释放锁。于是将 php 改为 session handler 改为 redis，并开启了 php-fpm 的 slowlog 设置为 1s。

再次压测发现性能出现在 session_start() 函数调用，于是将 redis-server 也开启了多个实例，压测发现 slowlog 不再出现 session_start() 调用。

最后将 php 的 opcache 打开，又提升了不少性能。

## 结果

PHP 程序逻辑为：

打开 session，包含文件，连接数据库，输出内容。

```
session_start();
require 'api.php';
new PDO('mysql:...', '...', '...');
echo 'something';
```

Apache ab 测试 1500 并发 15000 个请求，90% 的请求在 200ms 内完成，平均响应时间 190ms，每秒钟处理请求数在 7500，错误请求为0.

```
ab -c 1500 -n 15000 http://www.example.com/test.php
```







