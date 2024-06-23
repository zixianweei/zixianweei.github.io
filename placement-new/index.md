# C++中的placement new


通常情况下，操作符`new`用于动态分类内存；并且所分配内存的生命周期将不受作用域管理[^1]。与`malloc()`方法不同，操作符`new`会做两件事：分配内存；初始化对象。而`placement new`则是一种特殊的动态内存方法，其能够在已分配的内存空间上创建对象。因此，`placement new`只做了一件事：初始化对象。

<!--more-->

> Creates and initializes objects with dynamic storage duration, that is, objects whose lifetime is not necessarily limited by the scope in which they were created. If placement-params are provided, they are passed to the allocation function as additional arguments. Such allocation functions are known as ***placement new***, after the standard allocation function `void* operator new(std::size_t, void*)`, which simply returns its second argument unchanged. This is used to construct objects in allcated storage.

## 使用方式

``` cpp
#include <iostream>

class MyClass {
 public:
  MyClass() { std::cout << "MyClass::MyClass\n"; }
  ~MyClass() { std::cout << "MyClass::~MyClass\n"; }

  MyClass(const MyClass&) = delete;
  MyClass& operator=(const MyClass&) = delete;
  MyClass(MyClass&&) = delete;
  MyClass& operator=(MyClass&&) = delete;

 private:
  int m1;
  double m2;
};

int main() {
  {
    char buf[sizeof(MyClass)];
    void* p = buf;

    MyClass* o = new (p) MyClass();  // placement new
    o->~MyClass();  // Explicitly call the destructor for the placed object.
  }
  {
    MyClass* o = new MyClass();
    std::cout << "o address = " << o << "\n";

    o->~MyClass();
    o = new (o) MyClass();  // placement new
    std::cout << "o address = " << o << "\n";

    delete o;
  }

  return 0;
}
```

上方是一段使用`placement new`的场景：在栈上申请一段内存后，将这段内存的地址作为参数传递给操作符`new`。此外，对象的生命周期由用户负责：离开作用域时，系统会自动回收`buf`；在此之前，需要显式调用对象的析构函数。

## 使用场景

`placement new`能够在已分配内存上直接构造对象，省去了内存的分配的过程，可以重复利用已分配空间。但总的来说，并不推荐使用`placement new`，原因如下[^2]：

1. `placement new`需要用户自行保证已分配的内存空间能够放置对象，编译器和运行时都不会对空间是否足够进行检查。
2. 已分配空间可能存在对齐问题：虽然理论上已分配空间可以放置对象，但由于内存对齐，实际占用的空间会更大。

总的来说，编译器和运行时不会检查`placement new`是否正确完成，需要用户自行考虑空间大小内存对齐等问题。不管怎样，`placement new`不要随意使用，除非迫不得已；如果想要重复利用已分配内存空间，用户应当选择使用内存池管理内存而不使用`placement new`。

[^1]: [cppreference.com - new expression](https://en.cppreference.com/w/cpp/language/new)
[^2]: [isocpp - FAQ:Destructors](https://isocpp.org/wiki/faq/dtors)

