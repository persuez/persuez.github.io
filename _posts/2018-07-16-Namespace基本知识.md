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
C++中namespace关键字定义了一个declarative region，所以在一个namespace中声明的标志符和其他namespace中声明的同名标志符是没有名字冲突的。

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
除了一般的作用域运算符（::）直接访问namespace中的成员外，我们还有两种方法可以帮助我们轻松地多次访问namespace中的成员：
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
这里需要注意的是using declaration 如果访问的是函数```void fetch()```，那么语法如下：```using Jack::fetch```。注意，我们并没有指定函数的返回类型，参数信息，唯一指定的就是函数名，所以using declaration会把该函数的所有重载类型（如果有）都引用进来。

#### using directive
using directive 将使namespace中的所有成员可访问。如我们平常用的```using namespace std;```就是一个using directive。我们可以将using directive放在任何地方，但这也会造成它的可以使用的范围不同，但注意使用范围并不代表scope或者potential scope，这个可以从下面的例子看到。和using declaration的另外一个大的不同地方是当它在局部使用时，声明相同的局部变量不会报错，而是声明的局部变量会覆盖掉namespace中的成员。如：

``` cpp
namespace Jill {
  double bucket(double n) { ... }
  double fetch;
  struct Hill { ... };
}

char fetch; // global variable

int main()
{
  using namespace Jill; // Jill::fetch没有局部作用域，因为它没有覆盖掉全局的fetch，并且还可以声明一个局部的fetch，所以作用范围和作用域是不同的。
  Hill Thrill;
  double water = bucket(2); // use Jill::bucket
  double fetch; // 注意：这里不会出现错误，而是覆盖了Jill::fetch
  cin >> fetch; // 读入局部的fetch
  cin >> Jill::fetch; // 读入Jill的fetch
  cin >> ::fetch; // 读入全局fetch
}

int foom()
{
  Hill top; // 出错！
  Jill::Hill top; // 正确
}
```

#### 用using declaration 还是 using directive
一般来说，using declaration比using directive 安全。因为
- using declaration明确的告诉我们可以用哪个成员。
- using diclaration 如果名字冲突了，编译器会报错；而using directive会覆盖了namespace的版本，而不做任何提示。

#### namespace 嵌套
基本语法如下：

``` cpp
namespace elements {
  namespace fire {
    int flame;
    ...
  }
  float water;
}
// 可以访问fire用
using namespace elements::fire;

namespace myth {
  using Jill::fetch;
  using namespace elements;
  using std::cout;
  using std::cin;
}
int main()
{
  std::cin >> myth::fetch;
  // 或者
  std::cin >> Jill::fetch;

  // 访问 elements
  using namespace myth; // using namespace myth; using namespace elements;这两条语句和上面一条是等价的

}
```

#### 别名
namespace可以有别名。可以用来简化嵌套namespace的使用：

``` cpp
namespace MEF = myth::elements::fire;
using MEF::flame;
```

### Unnamed namespace
基本语法如下：

``` cpp
namespace {
  int ice;
  int bandycoot;
}
```
它有以下特点：
- Unnamed namespace 就像在其前面用了using directive一样，我们可以直接访问其成员。
- 因为其没有名字，所以只能在当前定义的文件中使用。也就是说，这个提供了一种类似关键字```static```的能力，只能在内部链接。
