---
title: 'Modern C++ Concurrency Utilities 06: Promise, Future and Asynchronous'
comment: true
math: true
date: 2023-06-24 21:44:54
update: 2023-06-24 21:44:54
tags:
    - C++
    - Concurrency
    - Promise
    - Future
    - Asynchronous
categories:
    - 知识类
---

「期值」是标准库为异步任务中返回值以及异常的传递提供的基础设施. 具体的做法是将这些值关联到共享的「future」对象上. 异步任务可以向这些期值对象中写入返回值或异常值, 异步任务的发起线程则可以通过这些期值对象等待, 读取, 检查其所需要的异步结果. 标准库中为上述功能提供的设施即 `std::promise` 和 `std::future`.

## `std::promise`

`promise` 为异步任务提供了存储返回值或异常的机制, 可以说, 在「期值」构建的异步交互机制中, `promise` 对象及其接口负责 **写入**. 异步任务所存储的值可以被其他线程异步地通过该 `promise` 对象构建出的 `std::future` 对象获取.

从语义上看, 每个 `std::promise` 对象都表示一个「共享的状态」, 并且这个共享的状态可能还没有被计算和设置, 或者在计算时发生了异常. 异步程序可以对 `std::promise` 对象执行三种操作以设置其所代表的共享状态的状态:
1. make ready: 将该 `promise` 对象所代表的共享状态设置为有效. 唤醒其他正在等待该共享状态的线程以读取该状态.
2. release: 将该 `promise` 对象和其关联的共享状态断开关联.
3. abandon: 设置该 `promise` 对象所代表的共享状态为异常状态, 然后 make ready 并 release

`std::promise` 提供的以下方法对应于上述三种抽象操作:
- `set_value`: 相当于 make ready, 向 `promise` 对象所代表的共享状态设置值并标记为 ready.
- `set_exception`: 相当于 abandon, 将 `promise` 对象所代表的共享状态设置为异常状态并 make ready
 
    需要注意一个 `promise` 对象只能 `set_*` 一次, 即如果已经调用了 `set_value` 或 `set_exception`, 那么新的 `set_*` 操作会抛出异常.
- `get_future`: 获取一个与当前 `promise` 对象关联的 `future` 对象, 相当于获取当前 `promise` 对象关联的共享状态的异步读取接口.

    需要注意的是一个 `promise` 只能 `get_future` 一次, 如果多次 `get_future` 会抛出异常.
- `set_value_at_thread_exit`: 与 `set_value` 不同之处在于该函数会设置 `promise` 所代表的共享状态的值, **但是不会立即 make ready, 而是在当前线程退出时才 make ready**
- `set_exception_at_thread_exit`: 与 `set_value_at_thread_exit` 同理

## `std::future`

`future` 提供了「期值」体系中的异步读取接口. 对于通过 `std::async`, `std::packaged_task` 等工具创建的异步任务, 或者手动使用 `std::promise` 的异步任务, 都提供了获取 `future` 对象的接口, 异步任务的创建者可以通过该接口获取 `future` 对象并读取结果, 或者在结果尚未计算完成并设置时等待结果, 或者在异步任务异常时捕获异常.

`future` 提供以下方法实现对异步结果的等待/验证/读取:
1. `get`: 阻塞地等待 `future` 对象所关联的共享状态被设置有效的值, 并且获取该值.
2. `valid`: 检查 `future` 对象所关联的共享状态是否有效. 所谓「有效」指 `future` 对象与一个共享状态相关联. 如果直接默认构造一个 `future` 对象而不是从 `async`/`packaged_task`/`promise` 获取 `future` 对象, 则 `valid()` 返回 `false`. 当对一个有效的 `future` 对象调用 `get` 后, `valid` 也返回 `false`.
3. `wait`: 阻塞等待到 `future` 对象所关联的共享状态可被读取.
4. `wait_for`/`wait_until`: `wait` 函数的 timeout 版本
5. `share`: 将当前 `future` 对象关联的共享状态 **转移** 到一个 `std::shared_future` 对象, 并返回该 `std::shared_future` 对象.

## `std::shared_future`

`std::future` 独占地指涉到共享状态 (不可拷贝), 不允许多个线程持有对一个共享状态的指涉, 因而无法做到多个线程访问该共享状态. `std::shared_future` 则可以做到多个 `std::shared_future` 对象指涉同一个共享状态 (可以拷贝). 同时多个线程通过不同的 `std::shared_future` 对象并发访问同一个共享状态是安全的.

`std::shared_future` 提供的方法在语义上只和 `std::future` 存在一点差别: 调用 `get` 后不再导致 `valid` 返回 `false`.

## `std::async`

`std::async` 函数模板接受可调用对象及其参数作为参数, 它会根据传入的执行策略参数 **异步** 地执行传入的可调用对象. `std::async` 会立即返回一个 `std::future` 对象, 作为读取传入的异步任务返回值的接口.

其中「执行策略」是一个枚举, 即 `std::launch`. 标准定义了两种执行策略:
- `std::launch::async`: 异步执行传入的可调用对象, *as if* 在一个新的 `std::thread` 对象中执行
- `std::launch::deferred`: 将传入的可调用对象和其参数存储在返回的 `future` 对象所指涉的共享状态中. 将其执行推迟到第一次对返回的 `future` 对象调用 **非 timeout** 版本的 `wait` 函数时. 后续对该 `future` 对象的访问 (如 `get`) 将立即返回结果.

## `std::packaged_task`

`std::package_task` 类模板提供一个 wrapper, 可以将任意的可调用对象 wrap 成异步任务 (可异步调用的可调用对象). 被 wrap 的异步任务的返回值或异常通过 `future` 对象传递. `std::package_task` 可以看作是 `std::function` 的异步版本.

- `valid`: 检查当前 `packaged_task` 是否持有一个共享状态.
- `get_future`: 返回一个指涉到当前 `packaged_task` 所持有的共享状态的 `future` 对象.
- `operator()`: 调用其 wrap 的可调用对象, 返回值或异常将被储存到其持有的共享状态中.
- `std::make_ready_at_thread_exit`: 执行其 wrap 的可调用对象, 并将返回值或异常储存到共享状态中, 但是并不立即将共享状态 make ready, 而是在当前线程退出时才 make ready.
- `reset`: 重置内部状态, 抛弃之前的执行结果并在内部重建新的共享状态.

## References

1. [std::promise - cppreference.com](https://en.cppreference.com/w/cpp/thread/promise)
2. [std::future - cppreference.com](https://en.cppreference.com/w/cpp/thread/future)
3. [std::shared_future - cppreference.com](https://en.cppreference.com/w/cpp/thread/shared_future)
4. [std::async - cppreference.com](https://en.cppreference.com/w/cpp/thread/async)
5. [std::packaged_task - cppreference.com](https://en.cppreference.com/w/cpp/thread/packaged_task)
