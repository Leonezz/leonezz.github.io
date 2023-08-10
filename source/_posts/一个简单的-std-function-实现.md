---
title: 一个简单的 std::function 实现
comment: true
math: true
date: 2023-08-10 13:25:09
update: 2023-08-10 13:25:09
tags:
    - C++
    - C++ Template
    - Type Erasure
categories:
    - 知识类
---

一个简单的 `std::function` 实现, 能够完成基本功能, 即可以代理:
1. 普通函数
2. `bind` 表达式
3. `lambda` 表达式以及普通 `functor`
4. 成员函数
5. 成员变量

`functor` 存储部分使用继承体系擦除 `functor` 类型, 保留返回值以及参数列表类型. 核心的调用部分则是直接使用 `std::invoke` 实现, 同时也没有做 `functor` 大小的区分, 统一存在堆上.

<!--more-->

```cpp
#include <functional>
#include <iostream>
#include <new>
#include <utility>

namespace trivial {
template <typename ReturnType, typename... Args>
struct FunctorHolderBase {
 virtual ~FunctorHolderBase() {}
 virtual ReturnType operator()(Args&&...) = 0;
 virtual FunctorHolderBase<ReturnType, Args...>* clone() const = 0;
};

template <typename Functor, typename ReturnType, typename... Args>
struct FunctorHolder final : FunctorHolderBase<ReturnType, Args...> {
 FunctorHolder(Functor func) : f(func) {}
 ReturnType operator()(Args&&... args) override {
  return std::invoke(f, std::forward<Args>(args)...);
 }

 FunctorHolderBase<ReturnType, Args...>* clone() const override {
  return new FunctorHolder(f);
 }
 Functor f;
};

template <class>
class function;

template <class ReturnType, class... Args>
class function<ReturnType(Args...)> {
 FunctorHolderBase<ReturnType, Args...>* functionHolderPtr{};

public:
 constexpr function() noexcept = default;
 template <class Functor>
 function(Functor f) {
  functionHolderPtr = new FunctorHolder<Functor, ReturnType, Args...>(f);
 }
 ~function() {
  if (functionHolderPtr) delete functionHolderPtr;
 }

 function(const function& other)
   : functionHolderPtr(other.functionHolderPtr->clone()) {}
 function(function&& other) noexcept : function() { other.swap(*this); }

 function& operator=(function other) = delete;
  
 void swap(function& other) noexcept {
  using std::swap;
  std::swap(functionHolderPtr, other.functionHolderPtr);
 }

 ReturnType operator()(Args&&... args) {
  return (*functionHolderPtr)(std::forward<Args>(args)...);
 }
};
} // namespace trivial

void normal_fun(int i) { std::cout << i << "\n"; }
struct Foo {
 Foo(int number) : num(number) {}
 void print_add(int i) const { std::cout << num + i << '\n'; }
 int num;
};

int main() {
 trivial::function<void()> f1([]() { return; });
 f1();
 trivial::function<void(int)> f2 = normal_fun;
 f2(-9);
 trivial::function<void()> f3 = std::bind(normal_fun, 31337);
 f3();
 trivial::function<void(Foo&, int)> f4 = &Foo::print_add;
 Foo foo(314159);
 f4(foo, 1);
 std::function<int(Foo const&)> f5 = &Foo::num;
 std::cout << "num: " << f5(foo) << '\n';
}
```

## References

1. https://iq.opengenus.org/std-function/