# 记录一次C++实验


今天有一个猜想：c++函数局部构造的class/struct，如果要返回给父函数，是通过移动语义实现的。
但我并不十分确定，于是做了个实验验证之。
<!--more-->

```c++
#include <iostream>
using namespace std;

class A {
public:
  A() : a{0} { cout << "constructing A\n"; }
  A(const A& other) : a{other.a} { cout << "copy constructing A\n"; }
  A(A&& other) : a{other.a} { cout << "move constructing A\n"; }

  void goo() { a = a * 2; cout << "calling goo\n"; }
private:
  int a;
};

A foo(A x) {
  x.goo();
  return x;
}

int main() {
  A x;
  A y = foo(x);
  return 0;
}
```

不加任何优化选项，编译运行。
输出结果如下：

> constructing A <br>
copy constructing A <br>
calling goo <br>
move constructing A

结果验证了我的猜想，即：返回局部构造的class/struct时，是移动语义，调用的是移动构造函数（前提是存在移动构造函数，如果只实现了拷贝构造函数，那么移动构造函数就不存在，这时只能调用拷贝构造函数）。
目的是将子函数构造的对象转交给父函数，从高层语义上讲，调用移动或拷贝构造函数应该都是可以的。但是，由于局部变量在函数结束时需要销毁，如果先拷贝再把自身销毁，效率有亏。而直接将自己掌握的资源移动给父函数，效率更高。所以，至少在g++编译器实现上，采用的是移动语义。至于c++标准对此有无规定，尚不得而知。

