---
title: 'Modern C++ Concurrency Utilities 04: Mutex and Lock'
comment: true
math: true
date: 2023-06-21 21:59:51
update: 2023-06-21 21:59:51
tags:
    - C++
    - Concurrency
    - mutex
    - lock
categories:
    - 知识类
---

`mutex` 即 mutual exclusion, 表示一种互斥语义. 所谓互斥, 实际上是指所有权的唯一性, 即同一时刻一个互斥量只能由一个所有者持有. 在并发语境中, 互斥量这一原语的所有者一般是「线程」, 它的作用是用于阻止多个线程同时访问共享的资源, 从而避免 data race 并为多个线程的执行提供实现「同步」的机制.

<!--more-->

## `std::mutex`

标准库中的 `std::mutex` 类提供基本的互斥量原语, 表示不可重入 (non-recursive) 的互斥语义. 它提供基本的 `lock`, `try_lock`, `unlock` 等方法:
- `lock`: 一个线程可以通过调用 `lock` 获取 `mutex` 的所有权. 如果 `mutex` 当前正被其他线程占有, 则 `lock` 会阻塞
- `try_lock`: 一个线程可以通过调用 `try_lock` 尝试获取 `mutex` 的所有权. 如果 `mutex` 当前正被其他线程占用, 则 `try_lock` 立刻返回 `false`.
- `unlock`: 一个持有当前 `mutex` 的线程可以通过调用 `unlock` 释放 `mutex`.

需要注意的是:
1. 如果当前线程正持有 `mutex`, 那么再次对该 `mutex` 调用 `lock` 会造成死锁 (调用 `try_lock` 不会死锁, 且返回 `false`, 但最好不要调用).
2. `mutex` 的生命周期一定要比它的持有者更长.
3. `mutex` 的持有者必须要在销毁前释放其对 `mutex` 的所有权.
4. 为了避免以上几点得不到保证时程序出错, 最好不要对 `mutex` 手动调用 `lock`/`try_lock`/`unlock` 等方法, 而是使用标准库提供的 RAII 管理类.

## `std::*_mutex`

除基本的互斥量原语外, C++ 11 中也提供了具备更多功能的 `*_mutex` 类:
- `timed_mutex`: 除 `mutex` 所具备的基本语义和操作外, `timed_mutex` 还提供了一种「半阻塞」获取 `mutex` 所有权的方法:
    - `try_lock_for`: 尝试获取所有权, 在获取到所有权后返回 `true` 或未获取到且阻塞了给定时间后返回 `false`
    - `try_lock_until`: 尝试获取所有权, 在获取到所有权后返回 `true` 或未获取到且阻塞到给定的时间点后返回 `false`
- `recursive_mutex`: 可重入锁/递归锁. 与基本的 `mutex` 提供不可重入的互斥语义不同, 同一个线程可以多次对 `recursive_mutex` 加锁.
    - `lock` 获取 `recursive_mutex` 的所有权, `unlock` 释放所有权. 不同的是同一个线程可以对同一个 `recursive_mutex` 多次 `lock`/`try_lock`, 并且当 `unlock` 调用次数和之前 `lock`/`try_lock` 调用次数相等时该线程才正式释放该 `recursive_mutex`.
    - `recursive_mutex` 的 `lock` 次数可能是有上限的 (在实际中必然如此), 当达到上限后继续 `lock` 会抛出 `std::system_error` 异常.
- `recursive_timed_mutex`: 语义和 `recursive_mutex` 相同, 多提供了 `try_lock_for` 和 `try_lock_until` 方法.

C++ 14 和 C++ 17 中为 `mutex` 拓展了「可共享」的语义:
- `shared_lock`: **可** 共享锁, 或者可以理解为「读写锁」, 它允许两种语义的互斥:
    - `shared`: 多个线程可以共享 `shared_mutex` 的所有权.
    - `exclusive`: 只有一个线程可以独占 `shared_mutex` 的所有权.

    这两种语义看似矛盾, 但其实它们的机制如下:
    - 如果当前 `shared_lock` 已经被通过 `lock`/`try_lock` 获取了 **独占所有权** , 那么任何其他的线程都无法获取该 `shared_lock` 的任何所有权
    - 如果当前 `shared_lock` 已经被通过 `lock_shared`/`try_lock_shared` 获取了 **共享所有权**, 那么其他线程无法获取它的独占所有权, 但是仍可以获取它的共享所有权.

    `shared_lock` 适合于「读共享, 写独占」的场景和类似场景, 即多个线程可以安全地对共享资源同时进行一类操作 (如读取), 但是只能有一个线程进行另一类操作 (如写入).

    `shared_lock` 形式上的差别和语义上的差别对应, 它比 `mutex` 多提供了用于共享所有权的方法:
    - `lock_shared`
    - `try_lock_shared`
    - `unlock_shared`

- `shared_timed_mutex` 相比 `shared_mutex` 多提供了用于 timeout 的方法:
    - `try_lock_for`/`try_lock_until`
    - `try_lock_shared_for`/`try_lock_shared_until`

## Lock

C++ 中 `mutex` 表示一种原子变量, 它的状态改变操作是同步阻塞 (`lock`) 或非阻塞 (`try_lock`) 的. 而「锁」则是 `mutex` 的 RAII 风格管理类.

- `lock_guard`: `lock_guard` 是一个简单轻量的 RAII 风格的泛型 mutex wrapper. 它以一个 `mutex` 风格 (提供了 `lock` 和 `unlock` 方法) 对象的 **引用** 为参数, 并持有这个引用, 且在构造函数中对该引用调用 `lock`, 在析构函数中调用 `unlock`. 仅此而已, 十分简单.
- `scoped_lock`: `scoped_lock` 是 C++ 17 中对 `lock_guard` 的一个拓展, 它支持对多个 `mutex` 对象的 RAII 管理. 它的单个 `mutex` 特化版本和 `lock_guard` 相同, 因此只管理一个 `mutex` 风格对象时 `scoped_lock` 退化为 `lock_guard`, 因此可以说 `scoped_lock` 无额外开销地替代了 `lock_guard`

    在管理多个 mutex 方面, `scoped_lock` 使用 `std::lock` 函数提供避免死锁的管理. `scoped_lock` 的实现也非常简单, 利用了一些 [可变模板参数和折叠表达式](./C-模板022-可变模板) 的特性.
- `unique_lock`: `unique_lock` 与上面两种简单的 RAII 管理类有所不同, 它相当于一个可移动的 mutex wrapper, 并且支持延迟锁定 (即不在构造函数中立刻 `lock`). 它提供对所 wrap 的 mutex 对象的 `lock_*`/`try_lock_*`/`unlock` 操作, 在构造函数中可以设置锁定策略:
    - `defer_lock`: 不在构造函数中调用 `lock`
    - `try_to_lock`: 在构造函数中调用 `try_lock`
    - `adopt_lock`: 假设当前线程已经获取了所传入 mutex 的所有权
    - 不设置锁定策略: 与 `scoped_lock`/`lock_guard` 类似, 在构造函数中调用 `lock`

    `unique_lock` 可以 `defer_lock` 并且可以多次 `lock` 并 `unlock` 的特点还使得它能够配合条件变量 (condition variable) 使用, 细节参考: 
- `shared_lock`: `shared_lock` 与 `unique_lock` 角色类似, 不同之处在于它转为 **共享所有权** 设计, 因此只能用于 `shared_mutex` 风格的 mutex 对象 (实现了 `lock_shared` 等操作). 对于一个 `shared_lock` 对象来说, 可以认为 `unique_lock<shared_lock>` wrap 了它的「独占所有权接口」, 而 `shared_lock<shared_lock>` wrap 了它的「共享所有权」接口.

## others

标准库提供了两个死锁避免的加锁算法, 可以同时获取多把锁:
- `std::try_lock`: 对传入的所有 [Lockable](https://en.cppreference.com/w/cpp/named_req/Lockable) 对象依次调用 `try_lock`, 其中任意一个返回了 `false` 都会终止后续的加锁并 `unlock` 之前被获取的 lockable 对象, 且返回加锁失败的那个锁的下标 (在参数列表中的位置).
- `std::lock`: 使用死锁避免算法对传入的 [Lockable](https://en.cppreference.com/w/cpp/named_req/Lockable) 对象加锁. `scoped_lock` 的多参数版本就使用了 `std::lock`.

对于简单的线程同步, 如控制一个过程在多线程环境中只调用一次, 标准库特地提供了 `call_once` 和 `once_flag`:

```cpp
template<class Callable, class... Args>
void call_once(std::once_flag& flag, Callable&& f, Args&&... args);
```

`call_once` 的语义是对于给定的 `once_flag` 和 `Callable` 对象以及参数, 保证只调用一次 `f`. 其中 `once_flag` 应该是标志着 `f` 和 `args` 是否已被调用的状态标志.

## References

1. [Concurrency support library (since C++11) - cppreference.com](https://en.cppreference.com/w/cpp/thread)
2. [c++ - std::unique_lock<std::mutex> or std::lock_guard<std::mutex>? - Stack Overflow](https://stackoverflow.com/questions/20516773/stdunique-lockstdmutex-or-stdlock-guardstdmutex)
3. [c++ - What's the best way to lock multiple std::mutex'es? - Stack Overflow](https://stackoverflow.com/questions/17113619/whats-the-best-way-to-lock-multiple-stdmutexes/17113678#17113678)
4. [c++ - std::lock_guard or std::scoped_lock? - Stack Overflow](https://stackoverflow.com/questions/43019598/stdlock-guard-or-stdscoped-lock)
