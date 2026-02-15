---
title: 异步编程之旅（03）同步与异步，Promise与Future
published: 2026-02-15
description: '同步与异步，从回调函数到 Promise/Future'
image: ''
tags: [promise, future, async, C++]
category: '异步编程之旅'
draft: false 
lang: ''
---
上一个文章通过线程、IO 模型、轮询三个角度解释了阻塞与非阻塞，但是：**阻塞/非阻塞关注的是“过程”（线程状态），而同步/异步关注的是“消息通知机制”（结果交付）。**

很多时候我们实现了“非阻塞”，却依然在“同步”地轮询。为了实现真正高效的系统，我们需要引入 Promise/Future 模型，让异步任务拥有同步的语义。

这里需要再强调一下：

- **同步 ≠ 阻塞**
- **异步 ≠ 非阻塞**

---



## 一、何为同步与异步



### 同步

调用者发起请求后，在没有得到结果之前，该调用就不返回。

**调用必须等待结果**

```cpp
#include <iostream>
#include <thread>
#include <chrono>

int slow_square(int x) {
    std::this_thread::sleep_for(std::chrono::seconds(2));
    return x * x;
}

int main() {
    std::cout << "开始计算\n";

    // 同步调用
    int result = slow_square(5);

    std::cout << "结果: " << result << "\n";
}

```



### 异步

调用者发起请求后，调用直接返回，不需要立刻得到结果。结果通常通过**回调（Callback）**、**事件**或**信号**来通知调用者。

**调用后立即返回，主线程不等待，结果通过回调通知**

```cpp
#include <iostream>
#include <thread>

void slow_square_async(int x, void(*callback)(int)) {
    std::thread([x, callback]() {
        std::this_thread::sleep_for(std::chrono::seconds(2));
        callback(x * x);
    }).detach();
}

void on_done(int result) {
    std::cout << "结果: " << result << "\n";
}

int main() {
    std::cout << "开始计算\n";

    slow_square_async(5, on_done);

    std::cout << "主线程继续运行\n";

    std::this_thread::sleep_for(std::chrono::seconds(3));
}

```

---



## 二、由回调函数到 Promise / Future



### 1. 基于回调函数的异步代码

上面的异步实现是通过回调完成的，但回调有几个问题：

- 控制流分散
- 错误处理麻烦
- 多个异步组合困难

例如，在一个后端服务中，我们需要：

- 异步读取用户信息

- 再根据用户信息读取订单

- 最后打印结果

```cpp
#include <iostream>
#include <thread>
#include <chrono>

void get_user_async(void(*callback)(int)) {
    std::thread([callback]() {
        std::this_thread::sleep_for(std::chrono::seconds(1));
        callback(42);  // user_id
    }).detach();
}

void get_order_async(int user_id, void(*callback)(int)) {
    std::thread([user_id, callback]() {
        std::this_thread::sleep_for(std::chrono::seconds(1));
        callback(user_id * 10);  // order_id
    }).detach();
}

void on_order(int order_id) {
    std::cout << "订单ID: " << order_id << "\n";
}

void on_user(int user_id) {
    get_order_async(user_id, on_order);
}

int main() {
    get_user_async(on_user);
    std::this_thread::sleep_for(std::chrono::seconds(3));
}

```

在真实服务中，“读取用户信息”和“读取订单”通常意味着：

- 网络请求
- 数据库查询
- RPC 调用

这些操作大部分时间都在等待 IO。如果采用同步阻塞方式处理，高频请求场景下，线程将大量时间浪费在等待上，系统吞吐量会严重下降。因此，异步执行往往是必须的。

然而，上述回调写法带来了明显问题：

- 业务逻辑被拆散到多个函数中
- 控制流不再是线性结构
- 错误必须逐层传递
- 一旦再增加一个步骤，就会继续嵌套

更严重的是：

> 代码的执行顺序已经不等于代码的书写顺序。

开发者阅读代码时，必须在多个回调之间来回跳转，才能还原真实的执行路径。这种现象被称为“回调地狱（Callback Hell）”。

------

#### 核心问题

回调的本质问题不在于“异步”，而在于：

> 异步逻辑无法被表达为一个“值”。

我们无法写出类似这样的代码：

```cpp
auto user = get_user_async();
auto order = get_order_async(user);
std::cout << order;
```

因为异步调用不会返回结果，而是通过回调“推送”结果。

那么，是否存在一种方式：**既保持异步执行，又能像同步代码一样获取结果，同时避免把逻辑写进回调函数中？**

为了解决这个问题，C++11 引入了 Promise/Future 模型。

---



### 2. Promise/Future 模型

**Future 是什么？**

> Future = “未来会得到的结果”

它本质上是一个：现在没有值，但将来一定会有值的对象

------

**Promise 是什么？**

> Promise = “承诺未来会给你一个值”

- Promise 负责“生产结果”
- Future 负责“消费结果”

它们是配对的。

Promise / Future 基本用法

```cpp
#include <iostream>
#include <future>
#include <thread>

int main() {
    std::promise<int> prom;
    std::future<int> fut = prom.get_future(); // 配对

  	// 在另一个线程中设置 Promise
    std::thread t([&prom]() {
        std::this_thread::sleep_for(std::chrono::seconds(2));
        prom.set_value(25);
    });

    std::cout << "等待结果...\n";

  	// 在主线程中，从 Promise 获取 Future
    int result = fut.get();  // 会阻塞

    std::cout << "结果: " << result << "\n";

    t.join();
}

```

- 线程是异步执行的
- 但 `future.get()` 是同步等待
- Future 让“异步执行”看起来像“同步取值”

---



**Future 让异步任务拥有了“同步语义”**

上述用户和订单的例子，可以改成这样：

```cpp
#include <iostream>
#include <future>
#include <thread>
#include <chrono>

int main() {
    using namespace std::chrono_literals;

    std::cout << "开始\n";

    // 异步获取用户
  	// 这里的auto，返回的实际上就是 std::future<int> 类型
    auto user_future = std::async(std::launch::async, [] {
        std::this_thread::sleep_for(1s);
        std::cout << "获取用户完成\n";
        return 42;
    });

    // 像同步一样“取值”
    int user_id = user_future.get();

    // 异步获取订单
    auto order_future = std::async(std::launch::async, [user_id] {
        std::this_thread::sleep_for(1s);
        std::cout << "获取订单完成\n";
        return user_id * 10;
    });

    int order_id = order_future.get();

    std::cout << "订单ID: " << order_id << "\n";
}

```

虽然获取用户和获取订单都是异步操作，但是语义上是同步的。

---



`std::async`是 C++ 标准库引入的异步处理函数，在 GCC 和 Clang 的实现中，`std::async` 会明确创建一个新的线程用于执行异步操作，在 MSVC 的实现中，它则会创建一个新的线程，或者利用 Windows 系统维护的线程池。

`std::async`实际上是对`std::promise`的一种包装，底层上，它们都共享一个**共享状态（Shared State）**。`std::promise` 需要你手动去 `set_value`，如：

```cpp
std::thread t([&prom]() {
    std::this_thread::sleep_for(std::chrono::seconds(2));
    prom.set_value(25);
});
```

而 `std::async` 则自动帮完成了“启动线程 -> 执行函数 -> 将返回值填入共享状态”的整个流水线。

很明显，当前的`std::async`并不适合高性能、高并发的操作，更适合作为简单的异步工具。在高并发场景中，开发者通常更倾向于使用自定义线程池，结合 `std::promise` 或 `std::packaged_task` 来实现可控的并发管理。

### 3. 异步不等于非阻塞

在上面的例子中

```cpp
int user_id = user_future.get();
int order_id = order_future.get();
```

这两个操作是同步地获取结果，如果异步任务尚未完成，`.get()` 会阻塞当前线程直到结果就绪。这其实就是异步阻塞的一种（异步执行 + 同步等待）。

如果刚创建 future 就执行`.get()`，则会让异步代码退化为同步代码。

当然，对于 C++ 的 Promise/Future 模型，还有很多细节，比如 `std::async` 的执行策略、线程的生命周期管理、获取 future 的时机等等，但是这里不再深入，仅仅是引入这个模型来讨论同步异步。

---



真正理想的方案是**既能用“同步风格”表达异步逻辑，又不会阻塞线程资源**，很遗憾 C++ 标准库的 Promise/Future 模型不提供这个功能，真正实现异步非阻塞，还需要引入更高层级的抽象，使用第三方库，例如 boost::future，以及，最重要的协程。

举这个例子是为了表明：Future 并没有消灭阻塞，它只是把“等待”从回调结构，变成了“显式取值”。最大的意义在于：让异步任务有了同步语义。至于是否阻塞，则取决于调用方是否选择等待。