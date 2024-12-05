---
title: "Modern C++ 中的多线程"
summary: "Modern C++ 多线程原理和线程池实现"
author: ["wivl"]
date: 2024-12-03
tags: ["cpp"]
weight: 1
ShowToc: true
draft: false
---

> 未完成

## 前言

自从 C++11 开始，C++ 标准库提供了跨平台的多线程设施。这意味着 C++ 开发者在编写多线程应用的时候不再需要为每一个操作系统使用特定的库，编写特定的代码。

我将在本篇文章探讨 C++17 多线程标准库的应用，并且在最后实现一个线程池。

## 多线程原理

> TODO: 

## 头文件

C++17 有以下几个与多线程相关的头文件：

|头文件|功能|C++ 标准|
|-----|---|--------|
|\<thread\>|提供 `std::thread` 及线程管理相关功能|C++11|
|\<mutex\>|提供互斥锁 `std::mutex`，`std::lock_guard` 等|C++11|
|\<condition_variable\>|提供条件变量，用于线程同步|C++11|
|\<future\>|提供异步任务、`std::future`，`std::promise` 等|C++11|
|\<atomic\>|提供原子操作支持，`std::atomic` 和 `std::atomic_flag`|C++11|
|\<shared_mutex\>|提供读写锁 `std::shared_mutex`，`std::shared_lock`|C++17|


## 线程

C++ 线程相关的设施定义在 `<thread>` 头文件中。`<thread>` 头文件主要包含两部分内容：线程对象 `std::thread` 和 `std::this_thread` 命名空间。

### 创建一个线程

`<thread>` 头文件里包含 `std::thread` 和相关方法。`std::thread` 即 C++ 中的线程对象。我们用以下方式创建一个新线程：

```cpp
// 1
#include <thread>
#include <iostream>

// 2
void task(int parameter) {
    std::cout << "thread with parameter " << parameter << std::endl;
}

int main() {
    std::cout << "Main thread" << std::endl;
    // 3
    std::thread a_thread(task, 42);
    // 4
    a_thread.join();
    // 5
    std::cout << "End main thread" << std::endl;
    return 0;
}
```

解释一下以上代码：

1. 引入了头文件 `thread`，包含线程相关的必要内容；

2. 定义了一个函数 `void task(int parameter)`，这是子线程的入口。函数包含了一个 int 参数 `parameter`，参数以**拷贝**形式传递；

3. 创建了一个新线程 `a_thread`，在这里构造函数接受了两个参数 `task` 和 `42`。第一个参数是我们需要线程执行的函数即线程的入口函数，第二个参数是入口函数的参数。在这个例子中相当于子线程调用了 `task(42)`。入口函数可以有很多参数，也可以没有参数；

4. 该语句声明添加线程的方式，`join()` 方法代表主线程需要阻塞等待子线程执行。除了使用 `join()`，还可以使用 `detach()` 方法，作用不同：

|方法|作用|
|---|---|
|join|主线程阻塞等待子线程执行|
|detach|将子线程分离于主线程独立运行，分离后的子线程称为守护线程（daemon threads）|

如果在 `std::thread` 对象销毁的时候没有声明添加线程的方式，将引发异常；

5. 由于 `a_thread` 调用 `join()` 方法，主线程阻塞，该语句将在子线程执行完后执行。

### std::this_thread

`std::this_thread` 是 `thread` 头文件定义的一个命名空间，一般在线程执行的函数中调用，例如上一个例子中的 `void task(int)` 函数。`this_thread` 命名空间中的函数与该线程的行为绑定。包含以下函数：

|函数|作用|C++ 标准|
|---|---|-------|
|yield()|手动让出 CPU 资源，线程暂停执行，转为就绪状态|C++11|
|get_id()|获得调用线程的线程 ID|C++11|
|sleep_for()|暂停线程一段时间|C++11|
|sleep_until()|暂停线程直到某一个时间点|C++11|

#### sleep_for & sleep_until

```cpp
#include <iostream>
#include <thread>
// 1
#include <chrono>

int main() {
    std::cout << "Start sleeping..." << std::endl;
    // 2
    std::this_thread::sleep_for(std::chrono::seconds(2));  // 休眠 2 秒
    std::cout << "Wake up after 2 seconds!" << std::endl;

    // 3
    auto wake_up_time = std::chrono::steady_clock::now() + std::chrono::seconds(3);
    std::cout << "Sleep until 3 seconds later..." << std::endl;
    // 4
    std::this_thread::sleep_until(wake_up_time);  // 直到指定时间点休眠
    std::cout << "Woke up after 3 seconds!" << std::endl;

    return 0;
}
```

1. `sleep_for`、`sleep_until` 使用 `chrono` API；

2. 休眠两秒；

3. 定义一个时间点；

4. 休眠直到时间点；

#### get_id

```cpp
#include <iostream>
#include <thread>

void print_thread_id() {
    // 1
    std::cout << "Thread ID: " << std::this_thread::get_id() << std::endl;
}

int main() {
    // 2
    print_thread_id();  // 主线程的 ID
    // 3
    std::thread t(print_thread_id);  // 创建一个子线程
    t.join();
    return 0;
}
```

1. 声明任务函数，调用 `get_id` 获取执行该函数的线程的 ID；

2. 主线程调用函数，输出主线程 ID；

3. 新建子线程调用函数，输出子线程 ID，两次输出的 ID 不同。

#### yield

```cpp
#include <iostream>
#include <thread>

void thread_function() {
    for (int i = 0; i < 5; ++i) {
        std::cout << "Thread working..." << std::endl;
        // 1
        std::this_thread::yield();  // 让出 CPU 给其他线程
    }
}

int main() {
    std::thread t(thread_function);
    t.join();
    return 0;
}
```

1. 在任务函数中调用 yield 函数，主动让出 CPU。


## 互斥锁

C++ 互斥锁相关的内容包含在 `<mutex>` 和 `<shared_mutex>`（C++14/17）头文件中。`<mutex>` 头文件主要包含互斥锁（mutual exclusion）相关的类和函数。当多个线程都需要访问同一资源的时候，互斥锁可以保证同一时间仅有一个线程访问该资源，从而防止数据竞争（data racing）。

C++17 标准支持以下几种不同类型的互斥锁：

|类型|描述|头文件|C++ 标准|
|---|---|------|--------|
|mutex|标准互斥锁，用于基本的互斥操作|`<mutex>`|C++11|
|timed_mutex|支持定时的互斥锁，提供超时功能（try_lock_for、try_lock_until）|`<mutex>`|C++11|
|recursive_mutex|递归互斥锁，同一线程可以多次获取锁，释放相同次数|`<mutex>`|C++11|
|recursive_timed_mutex|支持定时的递归互斥锁，结合递归锁和定时锁的功能|`<mutex>`|C++11|
|shared_timed_mutex|可共享互斥锁，带有超时功能|`<shared_mutex>`|C++14|
|shared_mutex|可共享互斥锁|`<shared_mutex>`|C++17|

### std::mutex

`std::mutex` 是最基础的互斥锁，用于保护共享资源，仅支持独占加锁，一个线程加锁后其他线程必须等待解锁。

```cpp
#include <iostream>
#include <thread>
// 1
#include <mutex>

// 2
std::mutex mtx;
// 3
int counter = 0;

void increment() {
    // 4
    mtx.lock();  // 独占加锁
    // 5
    ++counter;
    std::cout << "Counter: " << counter << std::endl;
    // 6
    mtx.unlock();  // 解锁
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);

    t1.join();
    t2.join();

    return 0;
}
```

1. 使用 `std::mutex` 需要引入头文件 `<mutex>`；

2. 定义全局互斥锁变量 `mtx`；

3. 定义共享资源 `int counter`，在这个例子中将有两个线程访问该变量；

4. 访问变量 counter 前加锁，使得其他线程无法同时访问，避免数据竞争；

5. 访问变量，修改值，并打印；

6. 访问变量完毕，解锁，允许其他线程访问变量；

### std::timed_mutex

`std::timed_mutex` 是 `std::mutex` 的增强版本，支持超时尝试加锁。提供 `try_lock_for` 和 `try_lock_until` 两种超时锁功能。

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <chrono>

// 1
std::timed_mutex tmtx;

void task(int id) {
    // 2
    if (tmtx.try_lock_for(std::chrono::milliseconds(100))) {  // 尝试在 100ms 内获取锁
        std::cout << "Thread " << id << " acquired the lock.\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(200));  // 模拟工作
        // 3
        tmtx.unlock();
    // 4
    } else {
        std::cout << "Thread " << id << " failed to acquire the lock.\n";
    }
}

int main() {
    std::thread t1(task, 1);
    std::thread t2(task, 2);

    t1.join();
    t2.join();

    return 0;
}

```

1. 定义全局 `timed_mutex` 变量；
2. 使用 `try_lock_for()` 方法，尝试在 100ms 内获取锁，如果获得锁返回 true，`try_lock_until()` 方法同理；
3. 成功获得锁，访问完资源后需手动解锁；
4. 处理未获得锁的情形。

### std::recursive_mutex

`std::recursive_mutex` 支持递归加锁，同一线程可以重复锁定，同时也需要释放相同的次数，可以用于需要递归调用任务函数的场景。

```cpp
#include <iostream>
#include <thread>
#include <mutex>

// 1
std::recursive_mutex rmtx;

// 2
void recursive_function(int count) {
    if (count > 0) {
        // 3
        rmtx.lock();
        std::cout << "Lock acquired for count: " << count << std::endl;
        // 4
        recursive_function(count - 1);
        // 5
        rmtx.unlock();
        std::cout << "Lock released for count: " << count << std::endl;
    }
}

int main() {
    std::thread t1(recursive_function, 3);
    t1.join();

    return 0;
}
```

1. 定义全局 `std::recursive_mutex` 变量；

2. 定义需要递归调用的任务函数；

3. 加锁；

4. 任务函数中递归调用任务函数，所以符合同一线程的条件；

5. 需要手动解锁。


### std::recursive_timed_mutex

`std::recursive_timed_mutex` 是 `std::recursive_mutex` 的增强版本，支持超时尝试加锁，适用于递归调用场景，同时需要加锁的超时机制。

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <chrono>

// 1
std::recursive_timed_mutex rtm;

void recursive_function(int count) {
    // 2
    if (rtm.try_lock_for(std::chrono::milliseconds(100))) {  // 尝试加锁
        std::cout << "Lock acquired for count: " << count << std::endl;
        if (count > 0) {
            // 3
            recursive_function(count - 1);
        }
        // 4
        rtm.unlock();
        std::cout << "Lock released for count: " << count << std::endl;
    // 5
    } else {
        std::cout << "Failed to acquire lock for count: " << count << std::endl;
    }
}

int main() {
    std::thread t1(recursive_function, 3);
    t1.join();

    return 0;
}
```

## atomic

`<atomic>` 头文件包含原子类型。原子类型用于在多线程环境中执行线程安全的操作，避免了使用互斥锁。它保证了对共享变量的操作不会被中断或干扰。原子操作时不可分割的操作，要么完全执行，要么完全不执行，不会被线程切断或中断。

## condition_variable

`<condition_variable>` 头文件主要包含条件变量（condition variable）相关的类和函数。条件变量是一种同步机制，用于在线程之间进行信号传递。它允许一个线程在某些条件下等待，而另一个线程在条件满足时通知等待的线程继续执行。

## future

`<future>` 头文件提供了异步机制。

## 线程池

### 线程池如何工作

### 线程池实现

## References

- [C++多线程详解](https://github.com/0voice/cpp_backend_awsome_blog/blob/main/%E3%80%90NO.610%E3%80%91C%2B%2B%E5%A4%9A%E7%BA%BF%E7%A8%8B%E8%AF%A6%E8%A7%A3%EF%BC%88%E5%85%A8%E7%BD%91%E6%9C%80%E5%85%A8%EF%BC%89.md)

- [C++ 并发编程](https://paul.pub/cpp-concurrency/)