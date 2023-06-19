---
title: C++ storage duration and specifiers
comment: true
math: true
date: 2023-06-19 16:17:18
update: 2023-06-19 16:17:18
tags:
    - C++
    - storage duration
    - basic concepts
categories:
    - 知识类
---

偶然在 vscode 的代码提示中看到了 `thread_local` 这个关键字, 心想 C++ 中什么时候提供了这种东西. 翻了下 cppreference 才知道早在 C++ 11 中就引入了这个修饰符, 用于声明线程生命周期. 趁此机会系统地看了下 C++ 变量的 4 种生命周期, 特此记录.

<!--more-->

C++ 11 中提供四种变量存储周期, 分别是:
1. automatic: 自动存储周期, 变量的存储空间会在其所处的代码块开始时分配, 并在结束时释放.
2. static: 静态存储周期, 变量的存储空间在程序开始时分配, 并在结束时释放. 在程序运行周期中该变量只存在一个实例.
3. thread: 线程存储周期, 变量的存储空间在其所属的线程启动时分配, 并在线程结束时释放. _每个线程中该变量都存在一个实例_.
4. dynamic: 动态存储周期, 变量存储空间的分配和释放由用户代码控制.

对于上述的存储周期, 在 C++ 中可以通过以下的 specifiers 显式地声明:

1. no specifiers: automatic. 在 C++ 中所有变量默认是自动存储周期的, 除非是全局变量 (file scope or namespace scope), 或显式声明了 `static`/`extern` 的变量.
2. `register`: automatic. `register` 修饰符只允许用在局部变量或函数参数上, 它是一种编译器建议, 语义是建议编译器将该变量存储在寄存器中作为一种优化. 此修饰符在 C++ 17 中已经废弃.
3. `static`: static. `static` 修饰符有多种语义, 用在 _局部_ 变量的声明中表示该变量具有静态存储周期.
4. `extern`: 不影响变量的存储周期, 只表示该变量 (符号) 具有 external linkage. 但是由于 `extern` 声明符只能用在全局变量或函数的 _声明_ 中, 因此凡是被 `extern` 修饰的变量都具有 static 或 thread 存储周期.
5. `thread_local`: thread. `thread_local` 是 C++ 11 中引入的关键字, 用于修饰全局变量, 局部变量或 _静态_ 成员变量. 表示该变量具有 thread 存储周期. 用在局部变量上时, 隐含有 static 语义. 用在全局变量上时, 可以和 `static` 或 `extern` 合用, 表示 internal/external linkage.

## References

1. [Storage class specifiers - cppreference.com](https://en.cppreference.com/w/cpp/language/storage_duration)
