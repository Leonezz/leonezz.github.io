---
title: 'C++ black magic: 利用友元函数和显式实例化突破访问权限'
comment: true
math: true
date: 2023-06-03 22:34:16
update: 2023-06-03 22:34:16
tags:
    - C++ Template
    - 黑魔法
categories: 知识类
---

C++ 中的非继承类有两种访问权限, `private` 和 `public`. 一般来说, `private` 只允许类型内部使用, 不允许外部访问. 然而, 由于 C++ 语言巨大的复杂性, 这一约束存在若干漏洞. 一种简单的做法就是假设 C++ 编译器不会对类对象的内存布局做奇怪的扰动, 这一假设通常是正确的, 因此可以通过猜测私有成员相对对象整体的地址偏移量访问私有成员. 或者更加方便地通过定义一个成员完全相同但访问权限相反的类, 并将原类对象的指针转换成新类的指针的方式访问原类对象的私有指针.

这两种方法本质上都基于「假设」, 即假设 C++ 编译器不会对类型的成员做特殊的处理. 本文介绍一种更加优雅的方法, 可以稳定地实现对私有成员的访问.

<!--more-->

基本的思路是利用友元函数, 由于友元函数这一特殊函数不属于成员, 而它又具有将类型的成员带到全局命名空间的能力. 具体的代码如下:

对于一个含有私有成员的类:

```cpp
class Private {
private:
    int a = 1;
};
```

我们想通过一个「小偷」类偷取它的私有成员:

```cpp
template <auto Member>
class theft {};

template <typename T, typename U, T U::*Member>
class theft<Member> {
    friend T& steal(U& u) {
        return u.*Member;
    }
};

int& steal(Private&);
```

可以看到上面定义的「小偷」类是一个模板, 它有一个万能的主模板, 可以匹配任意类型.
同时有一个特化模板, 特化模板的模板参数 `Member` 被声明为类型为指向类型为 `T` 的对象的指针, 同时这个 `Member` 指针指向的是类型 `U` 的成员: `T U::*Member`.

同时注意到, 特化的「小偷」模板中有一个友元函数 `steal` 的定义, 在这个定义中, 它直接返回了对未知类型 `U` 的未知成员指针 `Member` 解引用的结果. 同时需要注意, `steal` 有一个全局作用域声明.

显然地, 如果 `Member` 指向的是类型 `U` 的私有成员, 那么「小偷」类型将无法构造出具体的对象, 编译器会在检查到对私有成员访问的时候发出抱怨.

但 C++ 中可以对模板类 *显示实例化*, 而且这个显示实例化过程是不会检查成员访问权限的. 因此如果我们写下下面的代码:

```cpp
template class theft<&Private::a>;
```

「小偷」模板就会实例化成功, 模板参数 `T`, `U`, `Member` 分别会被替换为 `int`, `Private`, `&Private::a`. 此时我们仍然构造这个类, 但是仔细分析, 在实例化成功时, 函数 `int steal(Private&)` 有了一个定义, 即友元定义. 那么我们在调用这个函数时, 编译器可以找到合适的定义, 自然不会发出任何抱怨 !

```cpp
#include <iostream>
using std::cout;
int main() {
    Private p;
    endl(cout << steal(p)); // 1
    steal(p) = 100;
    endl(cout << steal(p)); // 100
}
```

整个过程具有一种声东击西的感觉, 不够聪明的编译器只能缴械投降.

代码: https://godbolt.org/z/brs6MPcfc

## Reference

1. http://eel.is/c++draft/temp.spec#6
2. https://www.zhihu.com/question/521898260