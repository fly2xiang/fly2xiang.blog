title: "PHP spl_object_hash 多进程程序的问题"
date: 2017-04-03 13:38:32
tags:
- PHP源码
- spl_object_hash
- 多进程
- WorkerMan
- pcntl_fork
categories: 
- PHP源码

---

## 前提

Workerman是一款纯PHP开发的开源高性能的PHP socket 服务器框架。本篇文章也要从使用Workerman开始。

在Workerman中, 每个Worker启动时, 预先fork多个进程（$worker->count）作为这个Worker的进程池。在Worker启动时, 回调函数`onWorkerStart`会被调用。

<!-- more -->

## 问题

先看如下代码 `simple_worker.php`：

```php
class Foo {
    public $bar = 0;
}

$global_var = new Foo();

$worker = new \Workerman\Worker('http://0.0.0.0:3000');
$worker->count = 5;
$worker->name = 'simple_worker';

$worker->onWorkerStart = function(\Workerman\Worker $worker) {
    global $global_var;
    $global_var->bar ++;
    echo 'current pid: '.posix_getpid().' current worker id: '.$worker->id.' $global_var->bar = '.$global_var->bar.' spl_object_hash($global_var) = '.spl_object_hash($global_var).PHP_EOL;
};

$worker->onMessage = function(Workerman\Connection\TcpConnection $connection, $data) {
    $connection->send('Hello');
};

\Workerman\Worker::runAll();
```

执行 `php simple_worker.php start`, 输出如下内容：

```
Workerman[simple_worker.php] start in DEBUG mode
----------------------- WORKERMAN -----------------------------
Workerman version:3.3.9          PHP version:5.6.30
------------------------ WORKERS -------------------------------
user          worker         listen               processes status
fly2xiang     simple_worker  http://0.0.0.0:3000   5         [OK]
----------------------------------------------------------------
Press Ctrl-C to quit. Start success.
current pid: 28523 current worker id: 1 $global_var->bar = 1 spl_object_hash($global_var) = 0000000047ec4f7a00000001103ecbd2
current pid: 28524 current worker id: 2 $global_var->bar = 1 spl_object_hash($global_var) = 0000000047ec4f7a00000001103ecbd2
current pid: 28522 current worker id: 0 $global_var->bar = 1 spl_object_hash($global_var) = 0000000047ec4f7a00000001103ecbd2
current pid: 28525 current worker id: 3 $global_var->bar = 1 spl_object_hash($global_var) = 0000000047ec4f7a00000001103ecbd2
current pid: 28526 current worker id: 4 $global_var->bar = 1 spl_object_hash($global_var) = 0000000047ec4f7a00000001103ecbd2
```

可以看出, `onWorkerStart`回调是在fork出的子进程中执行的。 fork出的子进程相当于复制了父进程的存储数据和代码空间, 现在输出的内容却有一些不符合我们的常识。

fork之后, 每个子进程应当有他们自己的`$global_var`, 对于`$global_var->bar++`, 每个进程对自己的`$global_var`自增, 所以各个子进程之间互不影响, 所以输出的`$global_var->bar = 1`。问题是为什么输出的`spl_object_hash($global_var)`是相同的呢？

## 验证

为了理解这是如何造成的, 查看Workerman源码`Worker.php`, 可以看到, 在`Worker::runAll()`静态方法中执行了一系列的操作, 其中`self::forkWorkers()`预先fork了Worker进程, 实际的fork操作在`self::forkOneWorker($worker)`方法中。

由于Workerman的代码有些复杂, 提取大概逻辑如下 `fork.php`：

```php
class Worker {
    public $count = 5;
    public static $_workers = array();

    function __construct() {
        self::$_workers[] = $this;
    }

    public static function forkOne($worker) {
        global $global_var;
        $pid = pcntl_fork();
        if($pid > 0){

        } elseif($pid === 0) {
            $global_var->bar ++;
            echo 'current pid: '.posix_getpid().' $global_var->bar = '.$global_var->bar.' spl_object_hash($global_var) = '.spl_object_hash($global_var).PHP_EOL;
            exit;
        }
    }

    public static function forkWorker() {
        foreach (self::$_workers as $worker) {
            for ($i = 0; $i < $worker->count; $i ++) {
                self::forkOne($worker);
            }
        }
    }

    public static function runAll() {
        self::forkWorker();
    }

}


class Foo {
    public $bar = 0;
}

$global_var = new Foo();

$worker = new Worker();

Worker::runAll();
```
执行该文件`php fork.php`
```
current pid: 28585 $global_var->bar = 1 spl_object_hash($global_var) = 0000000030f0896c0000000129a25381
current pid: 28586 $global_var->bar = 1 spl_object_hash($global_var) = 000000004bcf0705000000012d786d7f
current pid: 28587 $global_var->bar = 1 spl_object_hash($global_var) = 00000000298b5a46000000013e32daf0
current pid: 28588 $global_var->bar = 1 spl_object_hash($global_var) = 0000000029ab8bfc0000000120eed3d2
current pid: 28589 $global_var->bar = 1 spl_object_hash($global_var) = 0000000079cacb160000000175e3fd80
```
可以看出这里的`spl_object_hash($global_var)`是不同的, 与上面的Workerman例子不同。这又是为什么呢。

## spl_object_hash 的实现

为了找到这是为什么, 我从 [php.net](http://php.net) 下载了 PHP 的源代码, 在 `ext/spl/php_spl.c` 中找到了如下代码：

```c
/* {{{ proto string spl_object_hash(object obj)
 Return hash id for given object */
PHP_FUNCTION(spl_object_hash)
{
	zval *obj;

	if (zend_parse_parameters(ZEND_NUM_ARGS(), "o", &obj) == FAILURE) {
		return;
	}

	RETURN_NEW_STR(php_spl_object_hash(obj));
}
/* }}} */

PHPAPI zend_string *php_spl_object_hash(zval *obj) /* {{{*/
{
	intptr_t hash_handle, hash_handlers;

	if (!SPL_G(hash_mask_init)) {
		SPL_G(hash_mask_handle)   = (intptr_t)(php_mt_rand() >> 1);
		SPL_G(hash_mask_handlers) = (intptr_t)(php_mt_rand() >> 1);
		SPL_G(hash_mask_init) = 1;
	}

	hash_handle   = SPL_G(hash_mask_handle)^(intptr_t)Z_OBJ_HANDLE_P(obj);
	hash_handlers = SPL_G(hash_mask_handlers);

	return strpprintf(32, "%016zx%016zx", hash_handle, hash_handlers);
}
/* }}} */
```

从这里可以看出, `spl_object_hash`的逻辑是先查看全局变量`hash_mask_init`是否为`true`, 如果不为`true`, 就用随机数填充变量`hash_mask_handle`和`hash_mask_handles`, 最后返回值由`hash_mask_handle`与参数`obj.value.obj.handle`按位与或, 最后与`hash_mask_handles`拼接起来。

`obj.value.obj.handle`是宏`Z_OBJ_HANDLE_P(obj)`的展开, PHP7中, 变量在源码中的定义如下:

```php
typedef union _zend_value {
	zend_long         lval;				/* long value */
	double            dval;				/* double value */
	zend_refcounted  *counted;
	zend_string      *str;
	zend_array       *arr;
	zend_object      *obj;
	zend_resource    *res;
	zend_reference   *ref;
	zend_ast_ref     *ast;
	zval             *zv;
	void             *ptr;
	zend_class_entry *ce;
	zend_function    *func;
	struct {
		uint32_t w1;
		uint32_t w2;
	} ww;
} zend_value;

struct _zval_struct {
	zend_value        value;			/* value */
	union {
		struct {
			ZEND_ENDIAN_LOHI_4(
				zend_uchar    type,			/* active type */
				zend_uchar    type_flags,
				zend_uchar    const_flags,
				zend_uchar    reserved)	    /* call info for EX(This) */
		} v;
		uint32_t type_info;
	} u1;
	union {
		uint32_t     next;                 /* hash collision chain */
		uint32_t     cache_slot;           /* literal cache slot */
		uint32_t     lineno;               /* line number (for ast nodes) */
		uint32_t     num_args;             /* arguments number for EX(This) */
		uint32_t     fe_pos;               /* foreach position */
		uint32_t     fe_iter_idx;          /* foreach iterator index */
		uint32_t     access_flags;         /* class constant access flags */
		uint32_t     property_guard;       /* single property guard */
		uint32_t     extra;                /* not further specified */
	} u2;
};

struct _zend_object {
	zend_refcounted_h gc;
	uint32_t          handle; // TODO: may be removed ???
	zend_class_entry *ce;
	const zend_object_handlers *handlers;
	HashTable        *properties;
	zval              properties_table[1];
};
```
`obj.value.obj.handle`也就是`_zend_object`的`handle`。

在我们自己写的测试程序`fork.php`中, 在fork之前, `hash_mask_init`未被初始化, 而变量`$global_var`是在fork前全局定义的, `$global_var.val.obj.handle`在fork后随着`$global_var`被复制到各个子进程中。之所以每个子进程输出的`spl_object_hash($global_var)`是不同的, 是因为在子进程中每次都要初始化`hash_mask_handle`和`hash_mask_handles`。

那为什么在Workerman中`spl_object_hash($global_var)`是相同的呢? 可以猜测Workerman在fork之前执行过`spl_object_hash`函数, 此时预先初始化了`hash_mask_handle`和`hash_mask_handles`。带着这个猜测在Workerman源码中搜索`spl_object_hash`, 可以找到在Worker的构造函数中执行了`$this->workerId = spl_object_hash($this);`, 将此行去掉之后再次运行, 每个子进程输出的`spl_object_hash($global_var)`也是不同的。这也验证了之前在PHP源码中得到的结论。

## 意外发现

在寻找原因的过程中, 意外发现了另外一个问题, 先看代码:
```php

class Worker {
    public $count = 5;
    public static $_workers = array();

    function __construct() {
//        mt_rand();
        self::$_workers[] = $this;
    }

    public static function forkOne($worker) {
        $pid = pcntl_fork();
        if($pid > 0){

        } elseif($pid === 0) {
            echo mt_rand().PHP_EOL;
            exit;
        }
    }

    public static function forkWorker() {
        foreach (self::$_workers as $worker) {
            for ($i = 0; $i < $worker->count; $i ++) {
                self::forkOne($worker);
            }
        }
    }

    public static function runAll() {
        self::forkWorker();
    }

}

$worker = new Worker();

Worker::runAll();
```

在这段代码中, 如果与前面的代码结构相同, 不同的是将`spl_object_hash($global_var)`去掉了, 改为随机函数`mt_rand()`。运行这段代码会得到5个不同的随机数。

但如果将构造函数中`mt_rand()`的注释去掉, 再次运行, 你将得到5个相同的随机数。

我们知道通过`mt_rand()`生成的是伪随机数, 伪随机数生成需要一个种子, 伪随机数的种子一般以当前时间、进程ID或者计算机硬件代码来得出。在生成了随机数种子之后调用随机数函数得到的随机数序列已经确定了。

回到上面的代码, 在构造函数中执行`mt_rand()`之后, 随机数种子已经生成, 在fork时被复制到子进程中, 所以子进程在执行随机数函数时生成的随机数是相同的。

解决办法是可以在子进程中重新播下随机数种子。

下面是PHP源码中`mt_rand()`的实现：

```php
#define GENERATE_SEED() (((zend_long) (time(0) * getpid())) ^ ((zend_long) (1000000.0 * php_combined_lcg())))

/* {{{ php_mt_rand
 */
PHPAPI uint32_t php_mt_rand(void)
{
	/* Pull a 32-bit integer from the generator state
	   Every other access function simply transforms the numbers extracted here */

	register uint32_t s1;

	if (UNEXPECTED(!BG(mt_rand_is_seeded))) {
		php_mt_srand(GENERATE_SEED());
	}

	if (BG(left) == 0) {
		php_mt_reload();
	}
	--BG(left);

	s1 = *BG(next)++;
	s1 ^= (s1 >> 11);
	s1 ^= (s1 <<  7) & 0x9d2c5680U;
	s1 ^= (s1 << 15) & 0xefc60000U;
	return ( s1 ^ (s1 >> 18) );
}
/* }}} */

/* {{{ php_mt_rand_common
 * rand() allows min > max, mt_rand does not */
PHPAPI zend_long php_mt_rand_common(zend_long min, zend_long max)
{
	zend_long n;

	if (BG(mt_rand_mode) == MT_RAND_MT19937) {
		return php_mt_rand_range(min, max);
	}

	/* Legacy mode deliberately not inside php_mt_rand_range()
	 * to prevent other functions being affected */
	n = (zend_long)php_mt_rand() >> 1;
	RAND_RANGE_BADSCALING(n, min, max, PHP_MT_RAND_MAX);

	return n;
}
/* }}} */

/* {{{ proto int mt_rand([int min, int max])
   Returns a random number from Mersenne Twister */
PHP_FUNCTION(mt_rand)
{
	zend_long min;
	zend_long max;
	int argc = ZEND_NUM_ARGS();

	if (argc == 0) {
		// genrand_int31 in mt19937ar.c performs a right shift
		RETURN_LONG(php_mt_rand() >> 1);
	}

	if (zend_parse_parameters(argc, "ll", &min, &max) == FAILURE) {
		return;
	}

	if (UNEXPECTED(max < min)) {
		php_error_docref(NULL, E_WARNING, "max(" ZEND_LONG_FMT ") is smaller than min(" ZEND_LONG_FMT ")", max, min);
		RETURN_FALSE;
	}

	RETURN_LONG(php_mt_rand_common(min, max));
}
```

可以看到`mt_rand()`最终也调用了`php_mt_rand()`, 而`spl_object_hash`也调用了`php_mt_rand()`。

再回到PHP源码中`spl_obejct_hash`的实现, 它会在`hash_mask_init`为`false`时调用`php_mt_rand()`, 这样会带来的问题是：如果在父进程中没有调用`spl_object_hash`, 而调用了`mt_rand()`也会造成fork后的子进程中`spl_object_hash($global_var)`的结果相同, 读者可以自行在`fork.php`中Worker的构造函数加上`mt_rand()`进行测试。

## 总结

这应该不能算是PHP的一个Bug, 对于`spl_object_hash`, 本身就是为了区分不同的对象, 而在两个进程中相同的object_hash必然也不是相同的对象。在编写多进程程序中, 这些都是应当考虑的问题。

## 参考链接

[PHP:spl_object_hash - Manual](http://php.net/spl_object_hash)

[PHP:pcntl_fork - Manual](http://php.net/pcntl_fork)

[PHP:mt_rand - Manual](http://php.net/mt_rand)

[原理 | WorkerMan 3.x 手册](http://doc3.workerman.net/principle/README.html)

[fork(2) - Linux manual page](http://man7.org/linux/man-pages/man2/fork.2.html)
