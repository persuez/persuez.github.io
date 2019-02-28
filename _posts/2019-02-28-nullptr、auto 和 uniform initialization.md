---
published: true
tags:
  - C++ 11
author: persuez
---
本篇介绍一些 C++ 11 的小知识，其中包括：
1. `nullptr`关键字
2. `auto`关键字
3. `uniform initialization`

先介绍个小知识，C++ 11后的编译器可以识别：`vector <list<int>>`，不一定要写成`vector <list<int> >`。

---

## nullptr 关键字
C++ 11 为了区分 NULL 和 0,引入了一个新的关键字，即 nullptr。这也是一个常量值，它的类型是 `std::nullptr_t`，在头文件 <cstddef> 中定义：`typedef decltype(nullptr) nullptr_t;`。下面有个例子：
```
void f(int);

void f(void *);

f(0);
f(NULL);
f(nullptr);
```
对于 f 函数的三个调用会分别调用哪个原型呢？答案是 f(0)调用第一个，f(nullptr) 调用第二个，而 f(NULL) 则要看 NULL 的定义了，如果是将 NULL 定义为 0，则调用第一个，但有的编译器将 NULL 定义成为 0L（长整型），则相当于调用 f(0L)，那么将会出现二义性（调用时都要进行类型转换，那要转成哪个咧，编译器才不会知道），不知道有没有将 NULL 定义成 nullpr 的，那将调用第二个。

---

## auto 关键字
C++ 11  中的`auto`关键字和在这之前所表示的含义不一样，C++ 11 之前的 auto 表示变量的生命周期离开作用域被自动销毁，也就是平常所说的局部变量，C++ 11 中的 auto 表示编译器自动推导类型，如：
```
auto i = 24; // i 为 int
double f();
auto d = f(); //d 为 double

vector <string> a;
auto pos = a.begin(); // 用处1
auto l = [](int) -> bool {
  ...
}; // 用处2
```
auto 关键字一般用于变量类型比较长（用处1）或比较复杂（用处2），但我们不能太过于依赖于 auto，因为 C++ 是一门类型强敏感的语言，我们需要时刻注意变量的类型。

## uniform initialization
一致性初始化，顾名思义，即是可以按照统一的方法来初始化变量，语法是用到大括号 `{}`，背后是用了`initializer_list`。例子如：
```
int values[]{1,2,3};
vector <int> {1,2,3,4,5};
complex<double> c{1.0,2.3};
```
背后的原理是通过 `initializer_list<T>`，这个东西的背后是 `std::array<T, n>`，如果所构造的类型有参数为 initializer_list 的构造函数（可能是编译器自己构造的，仅供其使用的函数），则调用该函数；否则要求有相同数量参数的构造函数，如 complex 类要有一个带有两个形参的构造函数，然后编译器将 array<T, n> 中的元素逐个拆分然后传进该函数调用。
**注意**：用 uniform initialization 赋初值不允许 narrowing conversion（所有可能把原值窄化的转换）。即：
```
int a{1.0}; // error
int a = {1.0}; // error
int a(1.0);  // ok
complex<int> c{1.0,2}; //  error
char ch1{3}; // ok
char ch2{9999}; // error
int a{}; //  a == 0
int *p{}; //  p == nullptr
```
