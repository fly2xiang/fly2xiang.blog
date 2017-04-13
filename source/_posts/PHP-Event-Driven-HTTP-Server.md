title: "使用PHP编写基于事件驱动的HTTP Server"
date: 2017-04-13 16:20:17
tags:
- PHP
- 事件驱动
- 多进程
- Event-Driven
- non-blocking I/O
categories: 
- PHP

---

## PHP-FPM 的进程模型

PHP-FPM 使用的是多进程模型，每个进程处理一个请求。也就是说，工作进程数量有多少，可以并发处理的请求就有多少。而操作系统可以创建的进程数是有限的。这就带来了 [C10K问题](http://rango.swoole.com/archives/381)。

<!-- more -->

在 php-fpm 的配置文件中有以下配置:

```
pm.max_children #子进程最大数
pm.start_servers #启动时的子进程数
pm.min_spare_servers #最小空闲进程数，空闲进程不够时自动补充
pm.max_spare_servers #最大空闲进程数，空闲进程超过时自动清理
```

这些配置依据当前服务器的硬件配置来限制工作进程的数量，同样也限制了并发量。当并发量超过 php-fpm 限制的最大子进程数时，我们就会看到 502 Bad Gateway 错误。

在 php-fpm 环境下，php 脚本应当尽快处理请求，从而释放工作进程来处理下一个请求。

启动大量的进程会带来更多的进程调度，上下文切换的消耗最终会比完成实际工作的消耗还要多。

## 解决方案

为了解决这个问题，操作系统推出了一系列的解决方案，FreeBSD、DarWin 下的 kqueue，Linux 下的 epoll、poll、select，Windows 下的 IOCP等。都是为了解决这个问题而产生的。

它们都提供了事件驱动、异步非阻塞IO、事件循环。NodeJS 也使用了这些技术。

由于在各个不同的操作系统平台上的接口不同，也产生了很多对它们的封装。例如 libevent、libev、libuv （对比可见[Libevent Libev Libuv](http://zheolong.github.io/blog/libevent-libev-libuv/)）。

## 在 PHP 中使用

pecl 中提供一些 PHP 扩展，如 [event](https://pecl.php.net/package/event) 是 libevent 的扩展，[ev](https://pecl.php.net/package/ev) 是 libev 的扩展。

简单的使用 ev 扩展单进程的 HTTP Server 代码如下 `ev_http_server.php` :

```php
$connections = array();
$events = array();

$socket = stream_socket_server('tcp://0.0.0.0:9800');
stream_set_blocking($socket, 0);
$event = new EvIo($socket, Ev::READ, function($watcher, $events) use ($socket){
    global $connections, $events;
    $client_socket = stream_socket_accept($socket, 0, $remote_address);
    stream_set_blocking($client_socket, 0);
    $client_event = new EvIo($client_socket, Ev::READ, function($watcher, $events) use ($client_socket){
        global $connections, $events;
        fread($client_socket, 65535);
        $content = '<h1>It Works!</h1>';
        $content_len = strlen($content);
        $response = <<<EOL
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Connection: keep-alive
Content-Length: $content_len

EOL;
        $response .= "\r\n".$content;

        fwrite($client_socket, $response);
        fclose($client_socket);
        unset($connections[(int)$client_socket]);
        unset($events[(int)$client_socket]);
    });
    $events[(int)$client_socket] = $client_event;
    $connections[(int)$client_socket] = $client_socket;
});

Ev::run();
```

代码中，使用 `stream` 创建了一个 tcp 服务器，监听在 9800 端口。`stream` 是 PHP 中对数据流的一层封装。然后创建了 `EvIo`，监听 `$socket` 的可读事件，并设置了回调函数。最终执行 `Ev::run()` 开启事件循环。

当 `$socket` 可读时，也就是有新的连接时，回调函数被调用。此时 `accept` 这个连接，并在这个连接上设置可读回调函数。并在回调函数中写业务逻辑。

执行 `php ev_http_server.php`，PHP 脚本会阻塞，等待网络请求。(需要安装 ev 扩展 `pecl install ev` )

此时使用浏览器访问，可以看到输出。

使用 `ab -c 100 -n 10000 http://127.0.0.1:9800/` 压测 qps 可以达到 3W+。请求时间均在 10ms 以内，无错误。

* 测试主机使用 i7-4720MQ 4C8T @ 2.2GHz 处理器，512M 内存 VMWare 虚拟机进行

边缘触发（Edge triggered）和水平触发（也称条件触发 Level triggered）：

边缘触发会在文件描述符有可读写事件时触发用户的回调函数，此时回调函数必须将要读写的数据读写完，若没有读写完就返回，不会再次调用回调函数。

水平触发不同，若此次未读写完就返回，则回调函数会被再次调用。

在多核服务器下，可以使用 `fork` 方式创建多个进程，达到提高硬件利用率的效果，多进程版的代码如下 `ev_http_server_multi_process.php` ：

```php
$connections = array();
$client_events = array();

cli_set_process_title('Ev Http Server Main Process');
$socket = stream_socket_server('tcp://0.0.0.0:9800');
stream_set_blocking($socket, 0);

function init() {
    global $socket;
    $event = new EvIo($socket, Ev::READ, function($watcher, $events) use ($socket){
        global $connections, $client_events;
        $client_socket = @stream_socket_accept($socket, 0, $remote_address);
        if(! $client_socket) {
            // 惊群现象
            return;
        }
        stream_set_blocking($client_socket, 0);
        $client_event = new EvIo($client_socket, Ev::READ, function($watcher, $events) use ($client_socket){
            global $connections, $client_events;
            fread($client_socket, 65535);
            $content = '<h1>It Works!</h1>';
            $content .= '<p>current pid: '.posix_getpid().'</p>';
            $content_len = strlen($content);
            $response = <<<EOL
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Connection: keep-alive
Content-Length: $content_len

EOL;
            $response .= "\r\n".$content;

            fwrite($client_socket, $response);
            fclose($client_socket);
            unset($connections[(int)$client_socket]);
            unset($client_events[(int)$client_socket]);
        });
        $client_events[(int)$client_socket] = $client_event;
        $connections[(int)$client_socket] = $client_socket;
    });

    Ev::run();
}

for ($i = 0; $i < 5; $i ++) {
    $pid = pcntl_fork();
    if($pid > 0) {

    } elseif ($pid === 0) {
        cli_set_process_title('Ev Http Server Worker Process');
        init();
        exit;
    }
}

pcntl_wait($status, WUNTRACED);
```

此种方式的问题是会有 [惊群现象](http://pureage.info/2015/12/22/thundering-herd.html)，造成性能的浪费。

优化后的代码如下 `ev_http_server_multi_process.php` :

```php
$connections = array();
$client_events = array();

cli_set_process_title('Ev Http Server Main Process');

function init() {
    $context = stream_context_create();
    stream_context_set_option($context, 'socket', 'so_reuseport', 1);
    //使用 re use port, 有操作系统调度，避免惊群现象
    $socket = stream_socket_server('tcp://0.0.0.0:9800', $errno, $errstr, STREAM_SERVER_BIND | STREAM_SERVER_LISTEN, $context);
    stream_set_blocking($socket, 0);
    $event = new EvIo($socket, Ev::READ, function($watcher, $events) use ($socket){
        global $connections, $client_events;
        $client_socket = @stream_socket_accept($socket, 0, $remote_address);
        if(! $client_socket) {
            // 无惊群现象
            return;
        }
        stream_set_blocking($client_socket, 0);
        $client_event = new EvIo($client_socket, Ev::READ, function($watcher, $events) use ($client_socket){
            global $connections, $client_events;
            fread($client_socket, 65535);
            $content = '<h1>It Works!</h1>';
            $content .= '<p>current pid: '.posix_getpid().'</p>';
            $content_len = strlen($content);
            $response = <<<EOL
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Connection: keep-alive
Content-Length: $content_len

EOL;
            $response .= "\r\n".$content;

            fwrite($client_socket, $response);
            fclose($client_socket);
            unset($connections[(int)$client_socket]);
            unset($client_events[(int)$client_socket]);
        });
        $client_events[(int)$client_socket] = $client_event;
        $connections[(int)$client_socket] = $client_socket;
    });

    Ev::run();
}

for ($i = 0; $i < 5; $i ++) {
    $pid = pcntl_fork();
    if($pid > 0) {

    } elseif ($pid === 0) {
        cli_set_process_title('Ev Http Server Worker Process');
        init();
        exit;
    }
}

pcntl_wait($status, WUNTRACED);
```

优化的要点是将监听端口的动作放到 `fork` 之后，由各个子进程单独监听，并且使用操作系统提供的 `reuseport` 配置监听同一个端口。这样在 TCP 链接到来时操作系统会选择一个进程处理，不会唤起其他进程。

## 注意事项

使用这种方式虽然创建了异步多进程的 http 服务器，但在编程时不能在业务逻辑中编写同步代码（例如 PHP 中的 `file_get_contents`、`PDO` 等 IO 操作）。如果在业务中写同步代码，服务器就会退化为同步模式。可以使用第三方实现的异步组件来替代，例如：

[异步MySQL react/mysql](https://github.com/bixuehujin/reactphp-mysql)

[异步Redis clue/redis-react](https://github.com/clue/php-redis-react)

[异步HTTP-Client react/http-client](https://github.com/reactphp/http-client)

## event 扩展的其他应用

event 扩展编写的定时器 `event_timer.php` ，可以执行定时任务调度等。不能在其中写同步业务，会影响下次定时器的触发:

```php
$base = new EventBase();
$n = 0.05;
$last_time = microtime(true);
$e = new Event($base, -1, Event::TIMEOUT|Event::PERSIST, function($fd, $what, $n) use (&$e, &$last_time) {
    $current_time = microtime(true);
    $elapsed_time = $current_time - $last_time;
    echo "$elapsed_time seconds elapsed\n";
    $last_time = $current_time;
//    $e->delTimer();   // Trigger Once
}, $n);
$e->add($n);
$base->loop();
```

event 扩展编写的文件监控 `event_file_watcher.php` ，类似与 `tail -f /tmp/append_file.log` :

```php
<?php

$cfg = new EventConfig();
$cfg->avoidMethod('epoll'); // epoll doesn't support regular Unix files

$base = new EventBase($cfg);

$file = '/tmp/append_file.log';

$fd = fopen($file, 'r');

fseek($fd, 0, SEEK_END);

$event = new Event($base, $fd, Event::READ|Event::PERSIST, function($fd, $what) {
    echo fread($fd, 4096);
});

$event->add();

var_dump($base->getMethod()); // current backend method

$base->loop();

// after run this script, execute `echo  something >> /tmp/append_file.log`
```

运行 `php event_file_watcher.php`，另起一个终端，执行 `echo  something >> /tmp/append_file.log` 可以看到效果。

## 总结

IO异步化是提升服务器性能，提高服务并发能力的重要解决方案。使用异步方式编写 HTTP Server 解决了 PHP 在长连接应用上的不足。

## 参考链接

[关于C10K、异步回调、协程、同步阻塞](http://rango.swoole.com/archives/381)

[Libevent Libev Libuv](http://zheolong.github.io/blog/libevent-libev-libuv/)

[accept与epoll惊群](http://pureage.info/2015/12/22/thundering-herd.html)

