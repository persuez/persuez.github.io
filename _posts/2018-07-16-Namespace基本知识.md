---
published: true
tags:
  - C++
author: persuez
---
## Namespace
C++中namespace简单来说就是用来控制标志符（如变量，函数，类等）的名字冲突的。

## 简单术语
- *declarative region*: 指标志符声明的区域。具体见图一。
![declarative region](https://ws1.sinaimg.cn/large/006aPatNgy1ftc040hj5gj30yh0x6n0f.jpg)
- *potential scope*: 指从该标志符的声明点到其declarative region的终点。也就是一个标志符的最大可能作用域。具体见图二。
- *scope*: 指标志符的可见区域。具体见图二。
![potential scope and scope](https://ws1.sinaimg.cn/large/006aPatNgy1ftc0gdaotxj312x0x6adc.jpg)

## namespace关键字
C++中namespace关键字定义了一个declarative region，所所以在一个namespace中声明的标志符和其他namespace中声明的同名标志符是没有名字冲突的。

### 基本语法
我们定义两个namespace，分别命名为Jack和Jill，如下：
``` cpp
// ns.hpp
namespace Jack {
  double pail; // variable declaration
  void fetch(); // function prototype
  int pal; // variable declaration
  struct Well { ... }; // structure declaration
}

namespace Jill {
  double bucket(double n) { ... } // function definition
  double fetch; // variable declaration
  int pal; // variable declaration
  struct Hill { ... }; // structure declaration
}
```
namespace 可以定义在全局范围内，也可以嵌套在其他namespace中。如果namespace中声明了一些函数或成员函数，那么我们可以通过以下语法定义函数：
``` cpp
// ns.cpp
namespace Jack {
  void fetch()
  {
    ...
  }
}
```
### 访问namespace中的成员
有两种方法可以访问namespace中的成员：
- *using* declaration
- *using* directive
简单来说，前者可以让我们访问特定的成员，而后者则可以让我们访问整个namespace。下面就具体介绍两种方法：
#### using declaration
这是书本推荐的访问方法。**using declaration** 将访问的成员加入到它出现的declarative region中。具体用法直接看例子，注释也在样例中：

``` cpp
namespace Jill {
  double bucket(double n) { ... }
  double fetch;
  struct Hill { ... };
}

char fetch; // global variable

int main()
{
  using Jill::fetch; // 一个 using declaration; 它将fetch放到了当前的block declarative region，相当于声明了一个局部变量fetch
  double fetch; // 这将发生错误！因为上条语句已经相当于声明了一个局部变量fetch
  cin >> fetch; // 读入Jill::fetch
  cin >> ::fetch; // 读入全局的fetch
  ...
}
```
