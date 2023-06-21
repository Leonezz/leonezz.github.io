---
title: 'Modern C++ Concurrency Utilities 03: atomic'
comment: true
math: true
date: 2023-06-21 14:29:49
update: 2023-06-21 14:29:49
tags:
    - C++
    - Concurrency
    - atomic
categories:
    - 知识类
---

Atomic 提供了开发高性能无锁 (lock free) 结构的工具和接口, 对于内置类型或自定义类型, 也可以通过 `atomic<>` 模板包装成支持原子操作的类型, 以实现并发安全.

<!--more-->

### `atomic`

引用 [cppreference](https://en.cppreference.com/w/cpp/atomic/atomic) 的说法: `std::atomic` 模板的每个实例化和全特化都定义了一个原子类型. 如果一个线程写入一个原子对象的同时另一个线程读取这个原子对象, 其行为是明确定义的. 并且, 对原子对象的访问可能会建立线程间的同步关系, 并将对非原子内存的访问排序为传入的 `std::memory_order` 参数所定义的顺序. (`memory_order` 详见 [内存屏障和-C-内存模型](./内存屏障和-C-内存模型) )

{% cq %}
Each instantiation and full specialization of the `std::atomic` template defines an atomic type. If one thread writes to an atomic object while another thread reads from it, the behavior is well-defined (see memory model for details on data races).

In addition, accesses to atomic objects may establish inter-thread synchronization and order non-atomic memory accesses as specified by `std::memory_order`.
{% endcq %}

按照这个描述, 对于任意的自定义类型 (需要 [CopyConstructible](https://en.cppreference.com/w/cpp/named_req/CopyConstructible) and [CopyAssignable](https://en.cppreference.com/w/cpp/named_req/CopyAssignable)), 只要给它套上 `std::atomic` 这个 wrapper, 就可以定义一个原子的该对象. 但需要注意的是, 原子并不意味着 lock free. C++ 会在模板实例化时决定传入的自定义类型是否可以实现 lock free 的原子操作, 如果不能, 那么其内部仍然是通过 `mutex` 等有锁结构实现原子操作.

C++ 11 中为指针类型提供了偏特化, 为 bool 类型和 **所有** 的整形类型都提供了 `atomic` 模板的全特化, 它们中应该多数都是 lock free 的.

C++ 20 则进一步为浮点类型提供了全特化以及为 `std::shared_ptr<U>` 和 `std::weak_ptr<U>` 提供了偏特化, 而是否 lock free 则取决于 `U` 的类型. 细节参考: [std::atomic - cppreference.com](https://en.cppreference.com/w/cpp/atomic/atomic)

对于任意的 `atomic` 模板实例化或全特化, 其都会提供以下操作:
- `is_lock_free`: 检查 `atomic` 对象是否是 lock free 的, C++ 保证 `std::atomic_flag` 一定是 lock free 的, 对于其他类型在不同平台上可能有不同实现.
- `store`: **原子地** 为 `atomic` 对象赋值, 可以指定 `memory_order`
- `load`: **原子地** 读取 `atomic` 对象的值, 可以指定 `memory_order`
- `exchange`: **原子地** 为 `atomic` 对象赋值并读取它原先的值, 是 RMW (Read Modify Write) 操作.
- `compare_exchange_weak` and `compare_exchange_strong`: **原子地** 加载并 **bit wise** 比较 `atomic` 对象的值和 `expected` 参数值
    - 如果它们不相等, 则把 `atomic` 对象的值赋给 `expected` 参数(`expected` 是个引用), 并返回 `false`.
    - 如果它们相等, 则把 `desired` 参数值赋给 `atomic` 对象.

    值得注意的是, weak 版本和 strong 版本有如下差异:
    1. weak 版本可能会 **伪失败 (fail spuriously)**, 也就是说, 即使 `atomic` 对象的值和 `expected` 相等, 函数也可能产生它们不相等时的行为 (赋值 `expected` 并返回 `false`), 而 strong 版本不会.
    2. 当在循环中使用 `CAS` 时, weak 版本会有更好的性能:
  ```cpp
  while(atomic.compare_and_exchange(expected, desired))
    ;
  ```
  1. 对于一些值相同但位表示可能不同, 即多个位表示对应同一个值的类型 (如, 浮点数的 NaN 有多个位表示, 一些维护了内部状态但这些内部状态不决定对象的值的类), 通常使用 weak 版本更有效, 因为通常对于这些类的 CAS 操作都需要循环以使之收敛到稳定的位表示.
  2. 如果可以使用 strong 版本的 CAS 并且不需要循环时, strong 版本更加高效.

C++ 20 为 `atomic` 对象引入了三个主动式的线程同步操作:
- `wait`: 阻塞当前线程, 直到:
    1. 相同的 `atomic` 对象调用了 `notify` 函数
    2. `atomic` 对象的值与传入的参数值 `old` 不同.

    `wait` 方法用于多线程并发编程时实现线程对原子变量值的变化监测, 需要注意的是其中的比较是 bit wise 比较.
- `notify_one`: 如果存在因在当前 `atomic` 对象上调用 `wait` 函数而阻塞的线程, 那么就唤醒 **至少一个** 这样的线程, 如果不存在这样的线程该函数无效果.
- `notify_all`: 唤醒所有因在当前 `atomic` 对象上调用 `wait` 函数而阻塞的线程.

对于标准库内置的特化版本, `atomic` 还提供了一系列 `read`, `write`, `RMW` 操作, 包括算术和逻辑操作. 另外 `atomic` 所有的成员函数都有对应的非成员函数模板, 可以为非 `atomic` 类型重载出对应的操作. 详见: [cppreference.com](https://en.cppreference.com/w/cpp/atomic).

### `atomic_ref`

C++ 20 中引入了一个新的原子类, 它的语义是对一个非原子对象的「引用」, 特殊之处在于通过该「引用」进行的操作都是原子的!

这意味着 `atomic_ref` 相当于在已有的非原子对象上施加了一个支持原子操作的接口, 而并非一个逻辑上独立的新对象. 当我们在程序中仅在一部分地方需要对变量施加原子操作时, `atomic_ref` 十分有用: 如果使用 `atomic`, 那么对其的访问在整个程序中都必然是原子性的.

换一个角度理解 `atomic_ref`: 它相当于为「原子对象」提供了引用语义. C++ 11 中的 `atomic` 对象是值语义的, 它的构造函数接受一个非原子对象的「值」，并且在内部维护该值. 后续对该原子变量的操作实际上都是在操作原子对象本身. 而 `atomic_ref` 则不然, 它相当于并不拥有背后的非原子对象, 而只是持有对其的一个引用, 特殊之处仅在于这个引用上的操作都是原子的.

以下面的程序为例说明上述概念:

```cpp
// In sequential_fun, there are no concurrent running,
// no data races, therefore no need for atomic operation
void sequential_fun(int& cnt) {
    ++atomic;
}

// concurrent_fun will be running in multiple threads,
// potentially multiple cores.
void concurrent_fun(int& cnt) {
    std::atomic<int> atomic_cnt{cnt}; // an atomic int with the same value as cnt
    ++atomic_cnt; // not as expected: the increament should be performed to the reference parameter
    // unless do this:
    // cnt = atomic_cnt;
    // but it is non-atomic and what if the parameter is a large object
}
```

上面的程序中, 我们仅在函数 `concurrent_fun` 中需要原子操作, 并且原子操作的结果是通过引用传递的. 此时 `atomic` 的值语义就较为麻烦, 无法安全高效地实现对引用的修改. 而 `atomic_ref` 可以胜任这种情况:

```cpp
void concurrent_fun(int& cnt) {
    std::atomic_ref<int> atomic_cnt{cnt};
    ++cnt; // ok, increament is performed atomically to cnt
}
```

除了语义上的区别外, `atomic_ref` 与 `atomic` 在形式上区别不大, 都提供了相似的全特化和偏特化已经相似的成员函数. 详情参考: [std::atomic_ref - cppreference.com](https://en.cppreference.com/w/cpp/atomic/atomic_ref)

### `memory_order`

`memory_order` 是 `atomic` 相关操作中常见的参数, 它用来控制在原子操作中对内存 (非原子多级缓存结构的内存) 的访问进行排序. 包括内存写入操作执行后对于其他线程的可见性顺序. C++ 中抽象了 5 中 `memory_order`, 详见 [std::memory_order - cppreference.com](https://en.cppreference.com/w/cpp/atomic/memory_order), 它们是非常重要且复杂的概念, 已经在文章 [内存屏障和-C-内存模型](./内存屏障和-C-内存模型) 做了初步介绍.

## References

1. [std::atomic - cppreference.com](https://en.cppreference.com/w/cpp/atomic/atomic)
2. [std::atomic_ref - cppreference.com](https://en.cppreference.com/w/cpp/atomic/atomic_ref)
3. [c++ - What exactly is std::atomic? - Stack Overflow](https://stackoverflow.com/questions/31978324/what-exactly-is-stdatomic)
4. [A simple guide to atomics in C++. There’s often confusion around when… | by Josh Weinstein | Dev Genius](https://blog.devgenius.io/a-simple-guide-to-atomics-in-c-670fc4842c8b)
5. [std::atomic_ref - cppreference.com](https://en.cppreference.com/w/cpp/atomic/atomic_ref)
6. [C++20 atomic_ref (mariusbancila.ro)](https://mariusbancila.ro/blog/2020/04/21/cpp20-atomic_ref/)