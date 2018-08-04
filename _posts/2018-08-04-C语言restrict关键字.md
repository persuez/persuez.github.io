---
published: true
tags:
  - ANSI C
author: persuez
---
### restrict关键字
今天看APUE，发现了fgets的原型是```char *fgets(char *restrict buf, int n, FILE *restrict fp);```，然后一脸懵逼，```restrict```是什么鬼，于是跑去查了一下，原来和```pointer aliasing```有关，和编译器优化有关。

1. ```pointer aliasing```就是两个（多个）指针指向同一块内存区域。因为这个问题编译器不能优化代码（指针解引用）。如：
  ``` c
  int foo(int *a, int *b)
  {
    *a = 1;
    *b = 2;
    return *a + *b;
  }
  ```
  上面的代码不保证```foo```一定返回3,因为如果a和b指向的是同一块区域，那么```*b = 2;```这条语句已经改变了*a的值，因此结果返回4。

2. 如果我们已经知道a和b不可能指向同一块区域，使编译器可以进行优化，那么我们可以使用```restrict```关键字。如：
  ``` c
  int foo(int *restrict a, int *restrict b)
  {
    *a = 1;
    *b = 2;
    return *a + *b;
  }
  ```
  但是要注意的是，因为编译器知道不了a和b是否真的是一定不会指向同一块区域，所以这个要靠开发者决定。如果a和b指向了同一块区域，那么行为是**没有定义的**。

---
### 参考
[pointer aliasing](https://en.wikipedia.org/wiki/Pointer_aliasing)

[restrict](https://en.wikipedia.org/wiki/Restrict)

[一篇有趣的关于编译器优化的文章](https://archive.is/20130112201318/http://www.futurechips.org/tips-for-power-coders/how-to-trick-cc-compilers-into-generating-terrible-code.html)
