---
title: 'Modern C++ Concurrency Utilities 05: Condition variables and Semaphores'
comment: true
math: true
date: 2023-06-23 23:55:24
update: 2023-06-23 23:55:24
tags:
    - C++
    - Concurrency
    - condition variable
    - semaphore
    - synchronization
categories:
    - 知识类
---

Condition variable 和 Semaphore 是并发编程中用于同步控制和交互的原语. 多个并发线程可以使用 Condition variable 彼此交互, 一些线程可以通过 Condition variable 等待 (wait) 其他线程的通知 (notification). *Condition variable 总是和 mutex 关联使用*. Semaphore 相比 Condition variable 更加轻量, 它用来限制多个线程对共享资源的并发访问.

<!--more-->

## `condition_variable`

`condition_variable` 提供了线程间同步的并发原语, 具体地说, 「线程间同步」指类似下面所述的一种情况:
1. 一些任务分为多个阶段, 可以多线程流水线解决
2. 位于流水线较后位置的线程必须等待前面的阶段完成后才能执行
3. 每个阶段的线程完成任务后都会修改一个「共享的条件」, 从而后面的阶段可以从该条件判断前一阶段是否已完成

上述情况中多个流水线阶段的线程如何在恰当的时机等待并在恰当的时机开始运行即「同步」所需要实现的任务. 注意到要实现这一点那么对共享的状态变量的访问就必须是原子的, 因此 condition variable 通常和 mutex 一起使用.

条件变量提供了一种按特定「变量」等待和唤醒的结构化的接口. 具体地说, 多个线程可以在同一个 condition variable 上 `wait`, 这些线程会被以该 condition variable 为线索关联起来. 当其他线程在该 condition variable 上 `notify` 时, 调度系统就会从以该 condition variable 为线索而关联的那部分线程中取出并唤醒. 这就是 condition variable 所提供的机制.

这一机制如何同步编程中发挥作用呢? 首先共享的状态需要使用 `std::mutex` 保护, 其次需要同步的所有线程之间共享同一个 `condition_variable`.
- 需要修改共享状态的线程必须:
    1. 获取 `mutex` 的所有权
    2. 修改共享的状态变量
    3. 释放 `mutex` 的所有权
    4. 在 `condition_variable` 上调用 `notify_*` 函数. 这一函数会唤醒其他在同一个 `condition_variable` 上 `wait` 并阻塞的线程.
- 需要读取共享状态的线程必须:
    1. 在 `mutex` 上获取一个 `std::unique_lock`
    2. 执行下面的步骤:
        1. 检查共享状态, 判断其是否满足退出等待的条件 (防止条件已经满足且已经 `notify`, 只是当前线程错过了), 如果满足就退出等待.
        3. 在 `condition_variable` 上调用 `wait_*` 函数, 以 上面获取的 `unique_lock` 作为参数. 这一操作会释放 `unique_lock` 并进入阻塞状态, 直到其他线程调用 `notify`, 或者 timeout (对于 `wait_for`/`wait_until`)
        4. 重复上面两步

`condition_variable` 提供的操作:
1. `notify_one`: 唤醒一个正在当前 `condition_variable` 对象上 `wait` 的线程
2. `notify_all`: 唤醒所有正在当前 `condition_variable` 对象上 `wait` 的线程
3. `wait`: 以一个 `unique_lock` 的引用为参数, 原子地先 `unlock`, 然后阻塞该线程 (该线程以某种形式保存在该 `condition_variable` 内部, 以供 `notify` 时唤醒). 当被唤醒时, 会重新获取 `unique_lock` 然后退出.

    `wait` 函数通常与访问共享状态的代码一起使用, 例如:
   ```cpp
  while(!stop_waiting()) wait(lock);  
   ```
  
    有一个重载版本支持传入一个可调用对象, 效果等同于上面的代码.

    需要注意的是, 在进入 `wait` 函数之前, `unique_lock` 必须是 **已 lock** 状态, 并且在 `wait` 函数退出后, `unique_lock` 也会是 **已 lock** 状态.
4. `wait_for`: 相比 `wait` 增加了等待时间长度的参数
5. `wait_until`: 相比 `wait` 增加了等待到时间点的参数

下面是一个经典的利用 `condition_variable` 和 `unique_lock` 实现同步的例子: 用于「生产者-消费者」模型的同步队列:

```cpp
template <class T, class Cont = std::deque<T>>
class SharedQueue {
  std::queue<T, Cont> que{};
  std::mutex mut{};
  std::condition_variable cvProducer{};
  std::condition_variable cvConsumer{};
  std::size_t capacity{};

 private:
  bool full() const { return que.size() == capacity; }
  bool empty() const { return que.size() == 0; }
  
 public:
  SharedQueue() : capacity(100){};
  ~SharedQueue() {
    cvConsumer.notify_all();
    cvProducer.notify_all();
  };
  
  template <class... Args>
  SharedQueue(size_t capacity, Args... args)
      : capacity(capacity), que(std::forward<Args>(args)...) {}
  
  void push(const T& t) {
    std::unique_lock<std::mutex> lock(mut);
    while (full()) {
      cvConsumer.notify_one();
      cvProducer.wait(lock);
    }
    que.push(t);
    cvConsumer.notify_one();
  }

  T pop() {
    std::unique_lock<std::mutex> lock(mut);
    cvConsumer.wait(lock, [this] { return this->empty(); });
    auto v = que.front();
    que.pop();
    cvProducer.notify_one();
    return v;
  }
};
```

### `condition_variable_any`

标准库还提供了 `condition_variable` 的泛化版本即 `condition_variable_any`, 它可以支持任意的 [BasicLockable](https://en.cppreference.com/w/cpp/named_req/BasicLockable) 类型的锁, 而 `condition_variable` 只能和 `unique_lock` 一起使用.

`condition_variable_any` 的一种用法是配合 `shared_lock` 实现对 `shared_mutex` 保护的共享性资源访问的同步.

### `notify_all_at_thread_exit`

```cpp
void notify_all_at_thread_exit(std::condition_variable& cond, std::unique_lock<std::mutex> lk);
```

该函数提供一种机制在线程结束的最后阶段 (所有的 `thread_local` 生命周期变量都已被析构) 对给定的 `condition_variable` 调用 `notify_all`. 它执行下面的操作:
1. 在传递参数时线程内部的 `unique_lock` 的所有权被转移到该函数 (通过 `move`)
2. 修改当前线程的执行环境, 使得当前线程结束时执行:
    1. `lk.unlock();`
    2. `cond.notify_all();`

## `Semaphore`

Semaphore 相当于一种简单的计数器, 它只提供两种操作: 递增和递减. 可以把 semaphore 计数器的值理解为资源的数量, 递增操作增加资源数量, 递减操作减少资源数量. 当资源数量为 0 时, 递减操作会阻塞, 直到有新的递增操作.

### `std::counting_semaphore`

C++ 20 引入的 `counting_semaphore` 即普通意义的信号量, 可以为其设置 `LeastMaxValue` 非类型模板参数, 代表该信号量的「最大值的下限」, 即该信号量的最大值保证大于等于 `LeastMaxValue`.

`counting_semaphore` 提供基础的递增和阻塞以及非阻塞的递减操作:
1. `release`: 原子地递增内部计数器 (可传递递增的数量), 并唤醒因 `acquire` 时内部计数器为 0 而阻塞的线程
2. `acquire`: 原子地如果内部计数器数值大于 0, 递减1. 否则阻塞到其大于 0, 然后再递减 1.
3. `try_acquire`: `acquire` 的非阻塞版本
4. `try_acquire_for`/`try_acquire_until`: `try_acquire` 的 timeout 版本

信号量常用于 signaling/notifying 的语义, 而非互斥语义. 通过初始化信号量为 0, 从而阻止尝试 `acquire()` 的 receiver, 直到 notifier 通过调用 `release(n)` 来 「signaling」. 在这方面, 信号量可以被视为 `std::condition_variable` 的替代, 并且通常具有更好的性能.

通过将 `counting_semaphore` 的非类型模板参数设置为 1, 就得到了标准库内置的 `binary_semaphore`. 实际上, `binary_semaphore` 是一个 alias:

```cpp
using binary_semaphore = std::counting_semaphore<1>;
```

## `std::latch` and `std::barrier`

C++ 20 引入的 `std::latch` 和 `std::barrier` 也是线程间同步机制, 它们内部是一个计数器, 可以阻塞多个线程直到内部计数器达到一个恰当的值.

### `std::latch`

`latch` 内部是一个递减计数器, 计数类型是 `std::ptrdiff_t`. `std::latch` 在构造时初始化内部计数器的数值, 线程可以在 `std::latch` 对象上阻塞, 并一直等待到其内部的计数器递减到 0.

`std::latch` 不提供任何增加或重置计数器的方法, 因此 `std::latch` 是单向不可复用的「屏障」. 多个线程并发调用 `std::latch` 的成员函数不会带来 data race.

`std::latch` 提供以下操作:
1. `count_down`: 非阻塞原子地递减内部计数器. 传入的递减值不能大于内部计数器的数值, 否则是 UB.
2. `wait`: 主动阻塞到内部计数器归零
3. `try_wait`: `wait` 的非阻塞版本, 内部计数器的值为 0 时返回 `true`, 否则返回 `false`
4. `arrive_and_wait`: 原子地递减内部计数器并阻塞到内部计数器为 0.

`latch` 的一个用途是阻塞一系列后续线程直到指定数量的前序线程已经完成工作并递减了 `latch`.

### `std::barrier`

`barrier` 提供一种「屏障」, 使得可以阻塞一组 **确定数量** 的线程, 直到它们都到达该「屏障」. 与 `std::latch` 不同的是, `barrier` 可以复用, 同时可以在唤醒阻塞的线程时执行一个预先设置的可调用对象.

一个 `barrier` 的生命周期由一个或多个「phrase」组成. 每个「phrase」都定义有该「phrase」的 「**同步点 (phrase synchronization point)**」, 线程可以在该「同步点」处阻塞. 一个「phrase」由下面的步骤组成:
1. `barrier` 内部的计数器 (expected count) 在每次调用 `arrive` 或 `arrive_and_drop` 时递减
2. 当内部计数器为 0 时, 运行「phrase completion step」, 即构造时传入的自定义可调用对象 `CompletionFunction` 会被调用, 并且所有在「同步点」处阻塞的线程都被唤醒. 上述过程会在导致内部计数器恰好降为 0 的那个调用 `arrive`/`arrive_and_drop`/`wait` 的线程内执行.
3. 当「phrase completion step」执行完毕后, 内部计数器会被重置到构造时设置的值 **减去构造后调用 `arrive_and_drop` 的次数**, 此时下一个「phrase」就开始了.

`std::barrier` 提供的操作:
1. `arrive`: 原子地构造一个与当前 phrase 同步点关联的 `arrive_token` 对象, 然后递减内部计数器.
2. `wait`: 需要传入一个 `arrive_token` 对象的右值引用作为参数, 代表需要阻塞的同步点. 如果传入的 `arrive_token` 与当前 phrase 的同步点关联, 那么就阻塞到该同步点, 直到该 phrase 的 completion step 的执行. 否则, 立即返回 (传入的 `arrive_token` 与前一个 phrase 的同步点关联) 或 UB (传入的 `arrive_token` 与更早的 phrase 的同步点关联)
3. `arrive_and_wait`: 等价于 `wait(arrive())`
4. `arrive_and_drop`: 原子地先递减初始计数值, 然后递减当前 phrase 的实际计数值

`std::latch` 和 `std::barrier` 以及更新的 `std::flex_barrier` 还是第一次听说, 对它们的用法和用途还需要进一步学习. 一些参考:
1. [c++ - Where can we use std::barrier over std::latch? - Stack Overflow](https://stackoverflow.com/questions/48985967/where-can-we-use-stdbarrier-over-stdlatch)
2. [c++ - Why std::latch if there is std::barrier? - Stack Overflow](https://stackoverflow.com/questions/51065637/why-stdlatch-if-there-is-stdbarrier)
3. [Latches And Barriers - ModernesCpp.com](https://www.modernescpp.com/index.php/latches-and-barriers)

## References

1. [std::latch - cppreference.com](https://en.cppreference.com/w/cpp/thread/latch)
2. [std::barrier - cppreference.com](https://en.cppreference.com/w/cpp/thread/barrier)
3. [std::condition_variable - cppreference.com](https://en.cppreference.com/w/cpp/thread/condition_variable)
4. [std::counting_semaphore, std::binary_semaphore - cppreference.com](https://en.cppreference.com/w/cpp/thread/counting_semaphore)
5. [c++ - Where can we use std::barrier over std::latch? - Stack Overflow](https://stackoverflow.com/questions/48985967/where-can-we-use-stdbarrier-over-stdlatch)
6. [c++ - Why std::latch if there is std::barrier? - Stack Overflow](https://stackoverflow.com/questions/51065637/why-stdlatch-if-there-is-stdbarrier)
7. [Latches And Barriers - ModernesCpp.com](https://www.modernescpp.com/index.php/latches-and-barriers)