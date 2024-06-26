#c++ 学习笔记

## C++：Trivial、Standard-Layout 和 POD

### 引用

 - [c++ trivial/pod 是什么意思？](https://www.zhihu.com/question/472942396/answer/2009365067)
 - [C++：Trivial、Standard-Layout 和 POD](https://zhuanlan.zhihu.com/p/479755982)

### 六类特殊成员函数
 - 默认构造函数，`T::T()`
 - 拷贝构造函数，`T::T( (const) (volatile) T&)`
 - 移动构造函数， `T::T( (const) (volatile) T&&)`
 - 拷贝赋值运算符，`T::operator=( (const) (volatile) T&)`
 - 移动赋值运算符，即 `T::operator=( (const) (volatile) T&&)`
 - 析构函数，`T::~T()`

### trivial type

 1. 没有虚函数或虚基类。
 2. 由编译器生成默认的特殊成员函数（缺省或者`=default`形式）。
 3. 数据成员同样需要满足条件 1 和 2。
 - 空定义或者 =delete都不是Trivial了
 - 可以通过特征萃取工具分别去检测特殊成员函数
 ```c++
 #include <type_traits>
#include <iostream>

int main()
{
    using namespace std;

    cout << is_trivially_default_constructible<Foo>::value << std::endl;
    cout << is_trivially_copy_constructible<Foo>::value << std::endl;
    cout << is_trivially_move_constructible<Foo>::value << std::endl;
    cout << is_trivially_copy_assignable<Foo>::value << std::endl;
    cout << is_trivially_move_assignable<Foo>::value << std::endl;
    cout << is_trivially_destructible<Foo>::value << std::endl;
}
```
 - 六个is_trivially_* 的用处：在模板库中，就可以为对应的成员函数为 Trivial 的类型，做单独的特化，比如构造或者析构时可以不用做相应的操作。

### TriviallyCopyable
 - 除去默认构造函数， 其他五个都要满足trivial type。 
 - 对于满足 TriviallyCopyable 要求的类型，可以安全地使用 C 中 memcpy, memmove 等面向字节操作的函数，以提高程序的性能，而不用担心发生任何有关深浅拷贝的问题。
 - 比如std::copy可以实现优化成 memmove。 std::copy 只适合做不重叠的拷贝或者向前拷贝。重叠的向后拷贝应当交给 std::copy_backward。
 - 注意，自己实现的myCopy 不能被编译器优化成 memmove？编译器擅自将 myCopy 翻译成 memmove 是错误的优化！！！在目标区间和源区间有重叠的时候，memmove 执行的结果和优化前可能是不一致的。

### Standard Layout
当类（class 或 struct ）同时满足以下几个条件时是标准布局（standard-layout）类型：

- 没有虚函数或虚基类。
- 所有非静态数据成员都具有相同的访问说明符（public / protected / private）。
- 在继承体系中最多只有一个类中有非静态数据成员。
- 子类中的第一个非静态成员的类型与其基类不同。

第 4 个条件的产生是因为 C++ 允许优化不包含成员基类：

 - 在 C++ 标准中，如果基类没有任何数据成员，基类应不占用空间。所以，C++ 标准允许派生类的第一个成员与基类共享同一地址空间。
 - 但是，如果派生类的第一个非静态成员的类型和基类相同，由于 C++ 标准要求相同类型的不同对象的地址必须不同，编译器就会为基类分派一个字节的地址空间。

 ### PODType

 - std::is_trivial && std::is_standard_layout == true


 ##  C++ 中，“常量指针”（指向常量的指针）和“指针常量”（常量指针）
### 常量指针（指向常量的指针）
指向常量的指针意味着指针所指向的值是常量，不能通过该指针修改值。`const int* ptr;`

```cpp
#include <iostream>

int main() {
    int value = 10;
    const int* ptr = &value; // 指向常量的指针

    std::cout << "Value: " << *ptr << std::endl;

    // *ptr = 20; // 错误：不能通过指向常量的指针修改值

    value = 20; // 直接修改值是允许的
    std::cout << "Value: " << *ptr << std::endl;

    return 0;
}
```
在这个示例中，ptr 是一个指向常量的指针，你不能通过 *ptr 来修改 value 的值。

### 指针常量（常量指针）
常量指针意味着指针本身是常量，一旦指向了某个地址，不能再指向其他地址。语法：`int* const ptr;`

```cpp
#include <iostream>

int main() {
    int value1 = 10;
    int value2 = 20;
    int* const ptr = &value1; // 指针常量

    std::cout << "Value1: " << *ptr << std::endl;

    *ptr = 30; // 允许通过指针常量修改值
    std::cout << "Modified Value1: " << *ptr << std::endl;

    // ptr = &value2; // 错误：不能修改指针常量的地址

    return 0;
}
```
在这个示例中，ptr 是一个常量指针，它指向 value1，你不能将 ptr 指向其他地址，但是可以通过 *ptr 修改 value1 的值。
