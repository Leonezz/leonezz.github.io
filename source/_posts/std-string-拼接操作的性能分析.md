---
title: std::string 拼接操作的性能分析
comment: true
math: true
date: 2024-06-23 15:34:26
update: 2024-06-23 15:34:26
tags:
    - C++
    - string
categories:
    - 知识类
---

偶然的机会在工作中对 `std::string` 的拼接操作进行了对比, 本文记录对 4 种拼接方式的简单 benchmark:
1. `+` 操作符
2. `+=` 操作符
3. `append` 成员
4. `stringstream` 的 `<<` 操作符

<!--more-->

简单比较了下以上四种操作的性能, 初步结论是性能上 `append` $\approx$ `+=` > `+` > `stringstream`

下面是测试代码:
```cpp
#include <assert.h>
#include <chrono>
#include <functional>
#include <iostream>
#include <sstream>
#include <string>
#include <string_view>
#include <utility>

template <typename Functor, typename... Args,
          typename = decltype(std::declval<Functor>()(std::declval<Args>()...))>
void testWrapper(std::string_view name, Functor &&f, Args &&...args) {
  const auto beginTime{std::chrono::steady_clock::now()};
  std::forward<Functor>(f)(std::forward<Args>(args)...);
  const auto endTime{std::chrono::steady_clock::now()};
  std::cout << "test " << name << " runs for "
            << std::chrono::duration_cast<std::chrono::microseconds>(endTime -
                                                                     beginTime)
                   .count()
            << " us\n";
}

std::string testStringAdd(const std::string &param1) {
  std::string str = param1 + "111" + "222" + "333" + "444" + "555" + "666" +
                    "777" + "888" + "999" + "000";
  return str;
}

std::string testStringAddEqual(const std::string &param1) {
  std::string str = param1;
  str += "111";
  str += "222";
  str += "333";
  str += "444";
  str += "555";
  str += "666";
  str += "777";
  str += "888";
  str += "999";
  str += "000";
  return str;
}

std::string testStringAppend(const std::string &param1) {
  std::string str = param1;
  str.append("111");
  str.append("222");
  str.append("333");
  str.append("444");
  str.append("555");
  str.append("666");
  str.append("777");
  str.append("888");
  str.append("999");
  str.append("000");
  return str;
}

std::string testStringStream(const std::string &param1) {
  std::stringstream ss;
  ss << param1 << "111"
     << "222"
     << "333"
     << "444"
     << "555"
     << "666"
     << "777"
     << "888"
     << "999"
     << "000";
  return ss.str();
}

int main() {
  const auto param = "test";
  const auto str{testStringAdd(param)};
  const auto run_test = [&str](size_t n, auto f, const std::string &s) {
    for (size_t i{}; i < n; ++i) {
      assert(f(s) == str);
    }
  };

  constexpr size_t cnt{1000000};
  testWrapper("add", run_test, cnt, testStringAdd, param);
  testWrapper("add equal", run_test, cnt, testStringAddEqual, param);
  testWrapper("append", run_test, cnt, testStringAppend, param);
  testWrapper("string stream", run_test, cnt, testStringStream, param);
}
```

也可以在 compiler explorer 上查看: https://godbolt.org/z/cMPddcrfq
