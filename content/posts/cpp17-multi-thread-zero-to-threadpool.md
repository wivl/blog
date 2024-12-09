---
title: "C++17 多线程入门到线程池实现"
summary: "介绍基于 C++17 的多线程，并且最后实现一个简单的线程池"
author: ["wivl"]
date: 2024-12-03
tags: ["cpp"]
weight: 1
ShowToc: true
draft: false
---

## 前言

自从 C++11 开始，C++ 标准库提供了跨平台的多线程设施。这意味着 C++ 开发者在编写多线程应用的时候不再需要为每一个操作系统使用特定的库，编写特定的代码。

我将在本篇文章探讨 C++17 多线程标准库的应用，并且在最后实现一个线程池。本文面向那些对多线程原理有一定的了解，但是从来没有使用任何语言上手实现过相关内容，或者单纯想了解 C++17 标准如何编写多线程代码的读者。


## 头文件

C++17 有以下几个与多线程相关的头文件：

|头文件|功能|C++ 标准|
|-----|---|--------|
|\<thread\>|提供 `std::thread` 及线程管理相关功能|C++11|
|\<mutex\>|提供互斥锁 `std::mutex`，`std::lock_guard` 等|C++11|
|\<condition_variable\>|提供条件变量，用于线程同步|C++11|
|\<future\>|提供异步任务、`std::future`，`std::promise` 等|C++11|
|\<atomic\>|提供原子操作支持，`std::atomic` 和 `std::atomic_flag`|C++11|
|\<shared_mutex\>|提供读写锁 `std::shared_mutex`，`std::shared_lock`|C++14|


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

1. 定义全局 `std::recursive_timed_mutex`；

2. 尝试在 100ms 内获得锁，如果成功返回 `true`，否则返回 `false`；

3. 递归调用任务函数，需要重入锁；

4. 需要手动解锁；

5. 100ms 内未获得锁。

### std::shared_mutex

`std::shared_mutex` 在 C++17 标准中被引用，定义在头文件 `<shared_mutex>` 中。`std::shared_mutex` 支持共享锁（shared_mutex）和独占锁（unique_mutex）。共享锁允许多个线程同时获取，而独占锁同时只可以有一个线程获取。

考虑`reader-writer`问题：对于一个共享资源，可能有多个 reader 线程和多个 writer 线程。reader 线程可以同时读取资源，writer 线程只有在没有 reader 线程读取，并且没有其他 writer 线程写入的时候写入。在下面这段代码的例子中，reader 线程获取一个全局变量的值并输出，writer 线程让这个全局变量递增。

```cpp
#include <iostream>
#include <thread>
// 1
#include <shared_mutex>

// 2
std::shared_mutex smtx;
// 3
int shared_data = 0;

void reader(int id) {
    // 4
    std::shared_lock<std::shared_mutex> lock(smtx);  // 共享锁
    std::cout << "Reader " << id << " reading data: " << shared_data << std::endl;
}

void writer(int id) {
    // 5
    std::unique_lock<std::shared_mutex> lock(smtx);  // 独占锁
    ++shared_data;
    std::cout << "Writer " << id << " updated data to: " << shared_data << std::endl;
}

int main() {
    std::thread t1(reader, 1);
    std::thread t2(writer, 1);
    std::thread t3(reader, 2);

    t1.join();
    t2.join();
    t3.join();

    return 0;
}
```

1. `std::shared_mutex` 包含在头文件 `<shared_mutex>` 中；

2. 定义全局 `std::shared_mutex`；

3. 定义共享全局 int 变量；

4. reader 线程获取共享锁，这里直接使用 C++17 标准建议的 `std::shared_lock` 方式上锁，定义局部变量 `lock`，在作用域内无需手动上锁和解锁。这种方式将在稍后提及；

5. write 线程获取独占锁。

### std::shared_timed_mutex

`std::shared_timed_mutex` 相比 `std::shared_mutex` 支持超时尝试加锁，适用于对锁获取时间有要求的场景。

```cpp
#include <iostream>
#include <thread>
#include <shared_mutex>
#include <chrono>

std::shared_timed_mutex stm;
int shared_data = 0;

void reader(int id) {
    if (stm.try_lock_shared_for(std::chrono::milliseconds(100))) {  // 尝试共享锁
        std::cout << "Reader " << id << " reading data: " << shared_data << std::endl;
        stm.unlock_shared();
    } else {
        std::cout << "Reader " << id << " failed to acquire shared lock.\n";
    }
}

void writer(int id) {
    if (stm.try_lock_for(std::chrono::milliseconds(100))) {  // 尝试独占锁
        ++shared_data;
        std::cout << "Writer " << id << " updated data to: " << shared_data << std::endl;
        stm.unlock();
    } else {
        std::cout << "Writer " << id << " failed to acquire lock.\n";
    }
}

int main() {
    std::thread t1(reader, 1);
    std::thread t2(writer, 1);
    std::thread t3(reader, 2);

    t1.join();
    t2.join();
    t3.join();

    return 0;
}
```

这个例子与之前的例子类似，无需过多解释。

### RAII 管理互斥锁

熟悉 C++ 的读者应该也对 RAII 很熟悉。RAII 是 **Resource Acquisition Is Initialization** 的缩写，直译为**资源获取即初始化**，是贯彻现代 C++ 资源管理思想。RAII 要求，资源的有效期与持有资源的对象的生命周期绑定，即*由构造函数完成资源的分配，由析构函数完成资源的释放*。

在互斥锁的应用场景中，RAII 的做法实际上是利用 C++ 对象的生命周期。我们使用一个容器将锁包裹，当需要获取锁时创建对象，调用构造函数，自动上锁；当该容器生命周期结束时调用析构函数，自动解锁。在介绍 `std::shared_mutex` 的例子中的 `std::shared_lock` 和 `std::unique_lock` 就使用了这个原理。

C++ 标准建议使用 RAII 管理互斥锁。`<mutex>` 和 `<shared_mutex>` 库中提供了多种 RAII 工具用来管理互斥锁。

|设施|功能|头文件|C++ 标准|
|----|---|-----|-------|
|std::lock_guard|最基础的 RAII 管理类，自动加锁和解锁，不能手动解锁|\<mutex\>|C++11|
|std::unique_lock|std::lock_guard 的升级版，支持延迟加锁（即创建变量时不锁定），随时手动加锁解锁，以及超时锁功能|\<mutex\>|C++11|
|std::scoped_lock|同时管理多个互斥锁，防止死锁，构造时自动加锁所有互斥锁，析构时解锁|\<mutex\>|C++11|
|std::shared_lock|管理共享锁，用于 std::shared_mutex 和 std::shared_timed_mutex，支持延迟加锁等功能，支持多个线程同时加共享锁，但只有一个线程可以加独占锁|\<shared_mutex\>|C++14|

下面这段代码演示了如何使用这几个类。

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <chrono>

std::mutex mtx;

// 示例 1
// std::lock_guard，模板类，接受 std::mutex 为锁类型，使用全局变量 mtx 作为构造函数参数
void task1(int id) {
    std::lock_guard<std::mutex> lock(mtx);  // 创建对象自动加锁
    std::cout << "Thread " << id << " is working.\n";
    // 作用域结束，自动解锁
}


// 示例 2
// std::unique_lock，在这个例子中使用 std::timed_mutex 延迟加锁
std::timed_mutex tmtx;

void task2(int id) {
    std::unique_lock<std::timed_mutex> lock(tmtx, std::defer_lock);  // 延迟加锁，允许后面尝试获得锁，也可以不使用 std::defer 参数，直接加锁
    if (lock.try_lock_for(std::chrono::milliseconds(100))) {  // 尝试在100ms内加锁
        std::cout << "Thread " << id << " acquired the lock.\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(200));  // 模拟工作
    } else {
        std::cout << "Thread " << id << " could not acquire the lock.\n";
    }
}


// 示例 3
// std::scoped_lock，自动同时加多个互斥锁
std::mutex mtx1, mtx2;

void task3() {
    std::scoped_lock lock(mtx1, mtx2);  // 自动同时加锁多个互斥锁
    std::cout << "Task 3 is working.\n";
}

// 示例 4
// std::shared_mutex，我们已经在上面 shared_mutex 的 reader-writer 例子见过了
```

通过选择适合的 RAII 工具，可以简化锁的管理，避免手动加锁和解锁的错误，提高代码的安全性和可维护性。

值得注意的是，上面举得例子知识冰山一角。关于这几个锁其他用法请参考文档，本文末尾也列出了我在编写这篇文章时参考的其他内容。


## 条件变量

`<condition_variable>` 头文件主要包含条件变量（condition variable）相关的类和函数。条件变量是一种同步机制，用于在线程之间进行信号传递。它允许一个线程在某些条件下等待，而另一个线程在条件满足时通知等待的线程继续执行。我将着重介绍 `std::condition_variable`，头文件里还包含 `std::condition_variable_any` 类，相比前者更加灵活，其用法还请读者自行探索。

### std::condition_variable

`std::condition_variable` 的主要功能是让一个或多个线程等待某个条件成立，通过 `notify_one` 或 `notify_all` 唤醒等待线程。适用场景为：当线程需要等待某种条件，例如数据准备好、某个任务完成等。使用条件变量需要配合互斥锁使用，确保等待条件时的线程安全。

`std::condition_variable` 有两类常用成员函数：**等待函数**和**通知函数**。

|等待函数|用法|
|------|----|
|wait(std::unique_lock\<std::mutex\>& lock)|释放锁并阻塞线程，直到收到通知或条件满足时被唤醒；唤醒后，线程会重新获取并继续运行|
|wait(std::unique_lock\<std::mutex\>& lock, Predicate pred)|和上面的函数类似，但提供一个“谓词” pred，只有当 pred 返回 true 时，线程才会继续执行|
|wait_for()、wait_until()|除了等待通知外，还可以设置超时时间|


|通知函数|用法|
|------|----|
|`notify_one()`|唤醒一个正在等待的线程，如果没有线程在等待，则无效果|
|`notify_all()`|唤醒所有正在等待的线程，确保公平竞争|


用一个生产者-消费者的例子来说明它的使用方法。

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>
#include <chrono>

std::mutex mtx;
std::condition_variable cv;
std::queue<int> buffer;
const int maxSize = 10;

// 生产者函数：如果缓冲区已满（buffer.size() >= maxSize），调用 cv.wait 阻塞，知道缓冲区有空间
// 生产一个数据后，通过 cv.notify_all 通知消费者数据可用
void producer(int id) {
    int item = 0;
    while (item < 20) {
        // 使用 unique_lock 保护 buffer，避免竞争条件
        std::unique_lock<std::mutex> lock(mtx);
        // 调用 wait 方法，先释放锁，使得其他线程可以访问共享资源
        // 当被唤醒后，wait 会自动重新获取锁，确保被唤醒的线程能安全地检查条件
        cv.wait(lock, [] { return buffer.size() < maxSize; }); // 等待空间可用
        
        buffer.push(item);
        std::cout << "Producer " << id << " produced: " << item << std::endl;
        item++;
        
        cv.notify_all(); // 通知消费者
        lock.unlock();
        std::this_thread::sleep_for(std::chrono::milliseconds(100)); // 模拟生产耗时
    }
}

// 消费者函数：如果缓冲区为空（buffer.empty()），调用 cv.wait 阻塞，直到缓冲区有数据
// 消费一个数据后，通过 cv.notify_all 通知生产者可以继续生产
void consumer(int id) {
    while (true) {
        // 使用 unique_lock 保护 buffer，避免竞争条件
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, [] { return !buffer.empty(); }); // 等待数据可用
        
        int item = buffer.front();
        buffer.pop();
        std::cout << "Consumer " << id << " consumed: " << item << std::endl;
        
        cv.notify_all(); // 通知生产者
        lock.unlock();
        std::this_thread::sleep_for(std::chrono::milliseconds(150)); // 模拟消费耗时
    }
}

int main() {
    std::thread p1(producer, 1);
    std::thread p2(producer, 2);
    std::thread c1(consumer, 1);
    std::thread c2(consumer, 2);
    
    p1.join();
    p2.join();
    c1.detach(); // 消费者在生产结束后继续消费剩余的数据
    c2.detach();

    return 0;
}
```

`wait` 方法带谓词的版本被更广泛地使用，因为如果使用第一种不带谓词的 wait 方法而不反复检查条件的话，可能会造成*虚假唤醒（spurious wakeup）*。

考虑一种可能发生的情况：假设存在多个消费者线程阻塞等待，生产者调用 `notify_*` 方法通知阻塞的消费者线程。即使某个消费者线程的条件在调用 `notify_*` 时是满足的，另一个线程可能在当前线程被唤醒并重新获取锁之前改变了共享资源的状态，导致条件不再满足。因此，在多线程环境中，条件变量等待的核心规则是等待线程自己检查条件是否满足，而不能完全依赖 `notify_*` 的调用。也因此，`wait` 方法通常需要一个谓词函数来重复检查条件。


## future

`<future>` 头文件用于支持异步操作和线程间通信。它提供了一系列工具类和函数，允许线程安全地获取异步计算的结果。头文件组要提供了以下设施：

|主要类|用途|C++ 标准|
|-----|---|--------|
|std::future|用于获取异步任务的结果|C++11|
|std::promise|用于设置共享状态，配合 std::future 使用|C++11|
|std::shared_future|允许多个线程共享一个异步结果|C++11|
|std::packaged_task|将一个可调用对象与共享状态绑定|C++11|


|主要函数|用途|C++ 标准|
|-----|---|--------|
|std::async|启动一个异步任务并返回 std::future|C++11|


我将通过一些例子展示上述设施的用法。

### std::future & std::async


```cpp
#include <iostream>
#include <future>
#include <thread>

// 需要异步执行的任务
int compute() {
    std::this_thread::sleep_for(std::chrono::seconds(2));
    return 42;
}

int main() {
    // std::async 返回 std::future<int>，模板类型取决于异步任务的返回类型
    std::future<int> result = std::async(std::launch::async, compute);
    
    std::cout << "Waiting for result...\n";
    int value = result.get(); // 阻塞直到任务完成
    std::cout << "Result: " << value << "\n";

    return 0;
}
```

`std::future` 可以简单地理解为一个“占位符变量”，当某个函数返回一个 `std::future` 时，并不意味着该变量已经得到了一个值。只有调用 `future` 的 `get()` 方法时才尝试获得这个值。std::async 的作用是启动一个异步任务，之后主线程继续执行而**不阻塞等待返回结果**。

在这个例子中，`compute()` 任务在调用 `std::async()` 后执行，并需要等待两秒才会返回值，而主线程两秒内大概率已经执行到包含 `get()` 的语句了，所以主线程在 `get()` 阻塞直到获得返回值。

`std::async` 可选择两种启动模式：

|模式|作用|
|---|----|
|std::launch::async|在新线程中执行任务（默认）|
|std::launch::deferred|在调用 get() 时延迟执行任务|



### std::promise

```cpp
#include <iostream>
#include <future>
#include <thread>

void producer(std::promise<int> p) {
    std::this_thread::sleep_for(std::chrono::seconds(1));
    p.set_value(42); // 设置值
    // 可以继续做其他事情
}

void consumer(std::future<int> f) {
    // 可以先做其他事情
    int value = f.get(); // 获取值
    std::cout << "Received: " << value << "\n";
}

int main() {
    std::promise<int> p;
    std::future<int> f = p.get_future();

    std::thread t1(producer, std::move(p));
    std::thread t2(consumer, std::move(f));

    t1.join();
    t2.join();

    return 0;
}
```

`std::promise` 往往需要配合 `std::future` 使用。之前使用 `std::async` 的做法是将任务函数返回作为 std::future 获得值的时间点。配对使用 std::future 和 std::promise 使得线程间可以通过赋值进行值的获取。

在这个例子中，主线程分别定义 `std::promise` 和 `std::future` 对象并绑定。producer 线程通过调用 `std::promise` 的 `set_value()` 方法设置值。consumer 线程通过调用 `std::future` 的 `get()` 方法获得值。

### std::shared_future

相比 `std::future`，`std::shared_future` 允许多个线程共享异步结果，`std::shared_future` 的 `get()` 方法可以多次调用。

```cpp
#include <iostream>
#include <future>
#include <thread>

int compute() {
    return 42;
}

int main() {
    std::future<int> result = std::async(std::launch::async, compute);
    std::shared_future<int> shared_result = result.share(); // 转换为 shared_future

    auto worker = [shared_result]() {
        std::cout << "Shared result: " << shared_result.get() << "\n";
    };

    std::thread t1(worker);
    std::thread t2(worker);

    t1.join();
    t2.join();

    return 0;
}
```

### std::packaged_task

`std::packaged_task` 将可调用对象包装为任务，并将其与共享状态绑定。读者如果理解了上面的例子应该很容易就能看懂下面的例子。

```cpp
#include <iostream>
#include <future>
#include <thread>

int compute(int x) {
    return x * 2;
}

int main() {
    std::packaged_task<int(int)> task(compute);
    std::future<int> result = task.get_future();

    std::thread t(std::move(task), 21); // 启动任务
    t.join();

    std::cout << "Result: " << result.get() << "\n";

    return 0;
}
```


## 线程池

什么是线程池？为什么我们需要线程池？

如果读者恰好熟悉游戏开发，应该也听说过对象池这个概念。在游戏开发中，我们需要创建对象和消除对象。例如敌人生成的时候需要创建对象，敌人死亡的时候需要销毁对象。有时候这样的生成和销毁会很频繁，比如弹幕游戏里的子弹，或者是某种粒子效果的播放。频繁地生成和销毁对象的开销可能造成一些性能问题。对象池解决了这个问题。对象池事先创建若干个对象，在需要生成对象的时候将对象池里的对象移动到对应的位置重置并设置为可见即可；当需要销毁对象的时候只需要设置其可见性或者将对象移动到摄像机看不到的地方，而不用真的销毁它。

线程池的概念也类似，通过提前创建若干个线程，在需要执行任务的时候直接从池子里拿来用，用完的线程回收进池子，减少创建和销毁线程的开销。

### 线程池如何工作

线程池一般包含以下几个部分：

1. 线程容器：用于存放提前创建好的工作线程
2. 任务：工作线程需要执行的具体函数
3. 任务队列：用于存放工作线程需要执行的任务，作为线程池的缓冲机制

线程池的工作原理是：

- 创建线程池的同时事先创建指定数量的工作线程，工作线程在没有任务时阻塞，而不占用 CPU 资源；
- 当用户向线程池提交任务时，阻塞的工作线程被唤醒，从任务队列里取出一个任务执行；
- 工作线程执行完手头的任务时，如果队列里还有任务，便直接再取出一个任务执行而不阻塞。

### 线程池实现

#### 线程池类

我的目标是实现一个可以接受多种类型的任务函数的线程池，如果任务函数包含返回值，通过 std::promise 异步获得其值。

为了简化，整个线程池的实现都包含在 `Threadpool` 类内。有很多线程池的实现喜欢把任务实现为一个伪函数，包装成一个 `Task` 类，*在这里我不这样做*。

```cpp
#pragma once

#include <condition_variable>
#include <functional>
#include <future>
#include <memory>
#include <mutex>
#include <queue>
#include <thread>
#include <vector>

class Threadpool {
public:
    // 创建线程池并启动
    Threadpool(size_t thread_count = 0);
    ~Threadpool();

    template <typename F, typename... Args>
    auto submit(F &&f, Args &&...args) -> std::future<decltype(f(args...))> { ... }

private:
    std::vector<std::thread> threads;
    void worker_thread();

    std::queue<std::function<void()>> tasks;

    std::mutex mtx;
    std::condition_variable cv;

    bool exit_flag;
};
```

所有线程被存放在一个 `std::vector` 里，任务存放在 `std::queue` 里，并且这里使用了 `std::function`。定义了互斥锁、条件变量，和一个表示是否应该退出的 flag。工作线程运行的函数 `work_thread` 定义为类的普通成员函数。除了构造和析构函数以外，`Threadpool` 类只暴露了 `submit` 方法，用来接收新任务。`submit` 被定义为模板函数，其实现暂时先省略。

#### 构造函数和析构函数

```cpp
// 构造函数，接受参数表示线程池内工作线程的数量，默认值为 0
Threadpool::Threadpool(size_t thread_count) {
    exit_flag = false;
    // 如果构造函数传值为 0 或者省略使用默认值，将工作线程数量设置为 CPU 支持的线程数
    if (thread_count == 0) {
        thread_count = std::thread::hardware_concurrency();
    }
    // 启动指定数量的线程，并保存在 vector 里
    for (int i = 0; i < thread_count; i++) {
        // 工作线程运行的函数，注意和任务函数做区分
        auto thread = std::thread(&Threadpool::worker_thread, this);
        threads.push_back(std::move(thread));
    }
}

// 析构函数
Threadpool::~Threadpool() {
    {
        std::lock_guard<std::mutex> lock(mtx);
        // 设置 flag 为 true
        exit_flag = true;
    }
    // 唤醒线程，检查退出条件
    cv.notify_all();

    // 等待所有线程结束
    for (auto &thread : threads) {
        if (thread.joinable()) {
            thread.join();
        }
    }
    threads.clear();
}
```

#### 工作线程函数

工作线程函数是线程池里工作线程直接运行的函数。注意要将“工作线程函数”和“任务函数”做区分。任务函数是用户希望线程池执行的实际内容，工作线程函数是创建工作线程时直接运行的函数。这俩的关系有点像“应用程序”和“操作系统”的关系。

工作线程函数要做的事情就是：如果任务队列非空，从任务队列里取出一个任务执行，没有任务时阻塞，在线程池修改 flag 为 `true` 时退出执行。

```cpp
void Threadpool::worker_thread() {
    std::unique_lock<std::mutex> lock(mtx);
    // 退出条件：被通知退出，并且任务队列为空
    while (!exit_flag || (exit_flag && !tasks.empty())) {
        cv.wait(lock, [&] {
            // 1. 有新任务 2. 线程池需要退出   被唤醒
            return exit_flag || !tasks.empty();
        });
        // 如果是有新任务
        if (!tasks.empty()) {
            auto task = tasks.front();
            tasks.pop();

            lock.unlock();
            // 运行任务
            task();
            lock.lock();
        }
        // 否则是线程池需要退出
    }
}
```

函数里使用的时带谓词的 `wait` 方法，当线程执行完一个任务时，如果此时任务队列非空并且线程池不退出的话，`wait` 方法不阻塞而直接继续执行，从而不需要被唤醒。

#### 提交任务

线程池实现最复杂的是 `submit` 方法。之前说过我的线程池的目标是运行多种类型的任务函数，所以我们要使用函数模板。函数的返回值类型和参数类型、参数数量都是未知的。

```cpp
template <typename F, typename... Args>
auto submit(F &&f, Args &&...args) -> std::future<decltype(f(args...))> {
    std::function<decltype(f(args...))()> func =
        std::bind(std::forward<F>(f), std::forward<Args>(args)...);

    auto task_ptr =
        std::make_shared<std::packaged_task<decltype(f(args...))()>>(func);

    std::function<void()> warpper_func = [task_ptr]() { (*task_ptr)(); };

    {
        std::lock_guard<std::mutex> lock(mtx);
        tasks.push(warpper_func);
    }

    cv.notify_one();

    return task_ptr->get_future();
}
```

这个方法的实现运用了 C++ 的很多“魔法”。

1. 模板与返回类型

```cpp
template <typename F, typename... Args>
auto submit(F &&f, Args &&...args) -> std::future<decltype(f(args...))>
```

- `F` 是函数对象类型，可以是普通函数、lambda 表达式、成员函数指针等
- `Args...` 是可变参数模板，表示函数 `f` 的参数类型，可以是任意数量的参数
- `decltype(f(args...))` 的作用是推导出函数 `f` 的返回类型，并且在这里使用了返回值后置，该方法返回一个 `std::future`，允许异步获得任务返回结果


2. 函数绑定

```cpp
std::function<decltype(f(args...))()> func =
    std::bind(std::forward<F>(f), std::forward<Args>(args)...);
```

- `std::bind` 函数将函数和参数提前绑定，稍后可以直接调用

3. 包装任务函数

```cpp
auto task_ptr = std::make_shared<std::packaged_task<decltype(f(args...))()>>(func);
```

- 使用 `std::packaged_task` 将任务函数进一步包装，可以稍后直接获得 `std::future` 类型包裹的返回值

4. 进一步封装任务

```cpp
std::function<void()> warpper_func = [task_ptr]() { (*task_ptr)(); };
``

- 将任务包装进 `std::function`，便于同一存储到任务队列中

5. 将任务入队

```cpp
{
    std::lock_guard<std::mutex> lock(mtx);
    tasks.push(warpper_func);
}
```

6. 条件变量通知阻塞线程

```cpp
cv.notify_one();
```

- 唤醒可能的阻塞线程，使其从任务队列里取出任务执行

7. 返回 std::future

```cpp
return task_ptr->get_future();
```

至此，一个简单的，支持多种任务类型的线程池就完成了。

## References

- [C++多线程详解](https://github.com/0voice/cpp_backend_awsome_blog/blob/main/%E3%80%90NO.610%E3%80%91C%2B%2B%E5%A4%9A%E7%BA%BF%E7%A8%8B%E8%AF%A6%E8%A7%A3%EF%BC%88%E5%85%A8%E7%BD%91%E6%9C%80%E5%85%A8%EF%BC%89.md)

- [C++ 并发编程](https://paul.pub/cpp-concurrency/)

- [RAII, 维基百科词条](https://zh.wikipedia.org/wiki/RAII)

- [基于C++11实现线程池](https://zhuanlan.zhihu.com/p/367309864)