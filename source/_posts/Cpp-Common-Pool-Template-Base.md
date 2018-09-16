title: "编写基于模板的C++通用池"
date: 2018-09-16 22:50:35
tags:
- C++
- cpp
- 通用池
- Common Pool
- 连接池
categories: 
- C++

---


## 概述

在使用 Java 时，[Apache Common Pool](https://commons.apache.org/pool/) 是一个非常常用的通用池解决方案，使用池技术可以缓存较为常用的对象、数据库连接、文件句柄等，避免在每次使用时创建，提高应用程序的响应速度。

对于一个通用池，应当提供以下功能：

* 可配置的初始化池大小、最大池大小、最多空闲资源大小;
* 提供获取资源、归还资源的 API;
* 在资源不足时能够使当前线程阻塞以等待资源被归还.

本文使用 C++ 模板技术编写一个通用的池。

## 设计

首先，应当有三个属性分别保存初始化池大小、最大池大小、最多空闲资源大小。

对于一个通用池，应当存在一个容器存放空闲的资源，一个容器存放正在使用的资源。为使池中的各个资源被平均地被使用，这里使用一个 `std::list` 作为一个队列，每次在获取资源时从队列头获取一个资源。
正在使用的资源不需要保持有序，这里使用 `std::unordered_set` 存放。

对于阻塞线程的需求，使用 `std::metux` 与 `std::condition_variable` 来实现。

另外，为方便使用通用池，使用 RAII 机制，提供一个释放池的 `CommonPoolReleaseGuard`，能够在变量退出作用域时自动归还资源。

## 代码

这里直接贴代码

```cpp
//
// Created by fly2xiang on 2018/7/21.
//

#ifndef DEMO_COMMON_POOL_H
#define DEMO_COMMON_POOL_H

#include <list>
#include <unordered_set>
#include <iostream>
#include <mutex>
#include <condition_variable>
#include <thread>
#include "butil/logging.h"

template <typename T, typename Builder>
class CommonPool {

public:
    CommonPool(int init_size, int max_idle_size, int max_size) {
        _init_size = init_size;
        _max_idle_size = max_idle_size;
        _max_size = max_size;
        _wait_condition = true;
        for (int i = 0; i < _init_size; ++i) {
            T* t = build();
            _idle_queue.push_back(t);
        }
        _idle_queue_check_thread = new std::thread(&CommonPool::_idle_queue_task, this, 1);
    }

    ~CommonPool() {
        LOG(DEBUG) << "_idle_queue.size() = " << _idle_queue.size();
        for (auto i : _idle_queue) {
            delete i;
        }
        _idle_queue.clear();

        LOG(DEBUG) << "_active_set.size() = " << _active_set.size();
        for (auto i : _active_set) {
            delete (i);
        }
        _active_set.clear();
    }

    T* get() {
        bool idle_queue_is_empty = false;
        size_t idle_queue_size = 0;
        size_t active_set_size = 0;
        {
            std::lock_guard<std::mutex> lock_guard(_lock);
            idle_queue_is_empty = _idle_queue.empty();
            idle_queue_size = _idle_queue.size();
            active_set_size = _active_set.size();
            if (idle_queue_is_empty && active_set_size < _max_size) {
                T *t = build();
                _active_set.insert(t);
                return t;
            }
        }

        if (idle_queue_is_empty && active_set_size >= _max_size) {
            std::unique_lock<std::mutex> lock(_wait_mutex);
            _wait_condition = false;
            while (!_wait_condition) {
                _wait_condition_variable.wait(lock);
            }
        }
        std::lock_guard<std::mutex> lock_guard(_lock);
        idle_queue_is_empty = _idle_queue.empty();
        if (idle_queue_is_empty) {
            T *t = build();
            _active_set.insert(t);
            return t;
        }
        T* t = _idle_queue.front();
        _idle_queue.pop_front();
        _active_set.insert(t);
        if (t == nullptr) {
            LOG(ERROR) << "ERROR, Got NULL";
        }
        return t;
    }

    void release(T* t) {
        std::lock_guard<std::mutex> lock_guard(_lock);
        if (_active_set.find(t) != _active_set.end()) {
            _active_set.erase(t);
        }
        _idle_queue.push_back(t);
        _last_idle_queue_push_back_timestamp = time(nullptr);
        _wait_condition = true;
        _wait_condition_variable.notify_one();
    }

    inline T* build() {
        return Builder::build();
    }

    void _idle_queue_task(unsigned int time) {
        while (true) {
            _idle_queue_check();
            sleep(time);
        }
    }

private:
    std::list<T*> _idle_queue;
    std::unordered_set<T*> _active_set;
    int _init_size;
    int _max_idle_size;
    int _max_size;
    std::mutex _wait_mutex;
    std::condition_variable _wait_condition_variable;
    bool _wait_condition;
    std::mutex _lock;
    std::thread* _idle_queue_check_thread;
    time_t _last_idle_queue_push_back_timestamp;

    void _idle_queue_check() {
        LOG(DEBUG) << "_idle_queue_check";
        if (time(nullptr) - _last_idle_queue_push_back_timestamp > 10) {
            std::lock_guard<std::mutex> lock_guard(_lock);
            while (_idle_queue.size() > _max_idle_size) {
                T* t = _idle_queue.front();
                _idle_queue.pop_front();
                LOG(DEBUG) << "idle resource is greater than _max_idle_size, delete it";
                delete t;
            }
        }
    }
};

template <typename Pool, typename T>
class CommonPoolReleaseGuard {
public:
    CommonPoolReleaseGuard(Pool* pool, T* t) {
        _pool = pool;
        _instance = t;
    }

    ~CommonPoolReleaseGuard() {
        if (_instance != nullptr) {
            LOG(DEBUG) << "_pool release";
            _pool->release(_instance);
        }
    }

private:
    Pool* _pool;
    T* _instance;
};


#endif //DEMO_COMMON_POOL_H
```

## 如何使用

首先对于资源的生成，通用池提供了一个模板参数 `Builder`，在使用时需要编写一个 `Builder` 类，这里以 MySQL 连接 mysql-connector-cpp 为例。

```cpp
//
// Created by fly2xiang on 2018/7/21.
//

#ifndef DEMO_MYSQL_POOL_H
#define DEMO_MYSQL_POOL_H

#include "common_pool.h"
#include "mysql_driver.h"
#include "mysql_connection.h"
#include "cppconn/prepared_statement.h"
#include "cppconn/statement.h"
#include "cppconn/resultset.h"

class MysqlConnectionBuilder {
public:
    static sql::Connection* build() {
        sql::ConnectOptionsMap option;
        option["hostName"] = "127.0.0.1";
        option["port"] = 3306;
        option["userName"] = "root";
        option["password"] = "";
        option["schema"] = "test";
        option["OPT_RECONNECT"] = true;
        sql::Connection* connection = sql::mysql::get_driver_instance()->connect(option);
        return connection;
    }
};

class MysqlPool : public CommonPool<sql::Connection, MysqlConnectionBuilder> {
public:
    MysqlPool(int init_size, int max_idle_size, int max_size) : CommonPool(init_size, max_idle_size, max_size) {

    }

    static MysqlPool* get_instance() {
        return _instance;
    }

    static MysqlPool* _instance;
};


#endif //DEMO_MYSQL_POOL_H
```

此处，建立了 `MysqlPool` 类，继承了 `CommonPool`，这里管理的资源是 `sql::Connection`。

实际使用时，利用 `CommonPoolReleaseGuard` 较为方便的释放资源 ：

```cpp
std::string get_name_by_id(int id) {
    sql::Connection* conn = MysqlPool::get_instance()->get();
    CommonPoolReleaseGuard<MysqlPool, sql::Connection> release_guard(MysqlPool::get_instance(), conn);
    bool success;
    std::unique_ptr<::sql::PreparedStatement> preparedStatement(conn->prepareStatement("SELECT nickname FROM user WHERE id = ?"));
    preparedStatement->setString(1, id);
    success = preparedStatement->execute();
    if (! success) {
        response->set_msg("execute fail");
        return;
    }
    std::unique_ptr<::sql::ResultSet> resultSet(preparedStatement->getResultSet());
    long int rowsCount = resultSet->rowsCount();
    if (rowsCount <= 0) {
        response->set_msg("result is 0 row");
        return;
    }

    resultSet->next();
    std::string name = resultSet->getString("nickname");
    return name;
}
```

## 参考链接

[Apache Common Pool](https://commons.apache.org/pool/)
