# llvm ir笔记

## 一些概念摘要

- LLVM IR 是强类型的
- LLVM IR 不区分有符号整数和无符号整数。但是指令会区别有符号和无符号
- LLVM IR 假定是二进制补码有符号整数，因此说截断同样适用于有符号和无符号整数。
- 全局符号以@开头。
- 局部符号以%开头。
- 所有符号必须声明或定义。
- [全局变量和栈上变量皆指针](https://evian-zhang.github.io/llvm-ir-tutorial/03-数据表示/03-数据的使用.html#全局变量和栈上变量皆指针)

## 站在前人的基础上

- [LLVM IR入门指南](https://evian-zhang.github.io/llvm-ir-tutorial/index.html) ：这篇博客应该是中文里介绍llvm ir最基础的，最好的一篇了，适合入门。作者不是一上来就对llvm每个ir进行介绍，而是从体系结构的角度描述了llvm ir的整体概念。特别是第三章介绍的数据表示

- [Mapping High Level Constructs to LLVM IR](https://mapping-high-level-constructs-to-llvm-ir.readthedocs.io/en/latest/index.html)：这篇博客对llvm ir介绍的十分详细，并介绍了一些c++的实现，如class，constructer，vtable。不过他ir版本有些老了，我后面会用llvm17举例

- [LLVM Language Reference Manual](https://llvm.org/docs/LangRef.html#runtime-preemption-specifiers)：想要详细了解还是要看官方的百科全书

- [A Complete Guide to LLVM for Programming Language Creators](https://mukulrathi.com/create-your-own-programming-language/llvm-ir-cpp-api-tutorial/)这篇博客介绍了如何使用api来创建llvm ir


## 自己的一些理解

编译过程中无外乎涉及到的基本概念，符号，符号表，类型系统，数据布局使用。可以从这几个角度思考llvm ir的设计，加深自己的理解。

### 符号与符号表

- 符号通常指程序中使用的变量、函数、类型以及其他标识符的名称。
- 编译器通过符号表来管理这些符号，符号表是一种数据结构，用于存储程序中所有的符号及其相关信息。符号表中的每个条目都包含了一个符号的名称、类型、存储位置等信息。
- LLVM中符号表在数据布局上分为三类：
  - 寄存器中的符号：`%local_variable = add i32 1, 2` （局部变量赋值要符合SSA形式）
  - 栈上的符号：`%local_variable = alloca i32`
  - 数据区里的符号：指的是那些全局符号，如下：

 ```c++
int a;
extern int b;
static int c;
void d(void);
void e(void) {}
static void f(void) {}
 ```

```c++
@a = dso_local global i32 0, align 4
@b = external global i32, align 4
@c = internal global i32 0, align 4
declare void @d()
define dso_local void @e() {
  ret void
}
define internal void @f() {
  ret void
}
```

- 链接类型：对于链接类型，我们常用的主要有什么都不加（默认为external）、private和internal。
- 可见性：主要分为三种default, hidden和protected，这里主要的区别在于符号能否被重载。default的符号可以被重载，而protected的符号则不可以；此外，hidden则不将变量放在动态符号表中，因此其它的模块不可以直接引用这个符号。

- 可抢占性：在我们日常看到的LLVM IR中，会经常见到dso_local这样的修饰符，在LLVM中被称作运行时抢占性修饰符。

### 类型系统

- [常量](https://mapping-high-level-constructs-to-llvm-ir.readthedocs.io/en/latest/basic-constructs/constants.html#constants)
- 各种基本类型、派生类型，官方详细介绍：https://llvm.org/docs/LangRef.html#type-system
- [structure]()、[union](https://mapping-high-level-constructs-to-llvm-ir.readthedocs.io/en/latest/basic-constructs/unions.html)：官方介绍https://llvm.org/docs/LangRef.html#structure-type，注意packed structure的区别
- 函数类型：https://mapping-high-level-constructs-to-llvm-ir.readthedocs.io/en/latest/basic-constructs/functions.html
- 元数据metadata
- 类型转换指令：https://mapping-high-level-constructs-to-llvm-ir.readthedocs.io/en/latest/basic-constructs/structures.html

### 数据使用

## c++特性的llvm ir实现

### class和construct的实现

例子：https://compiler-explorer.com/z/e77T7sTP1

```c++
#include <stddef.h>

class Foo {
public:
    Foo() {
        _length = 0;
    }

    size_t GetLength() const {
        return _length;
    }

    void SetLength(size_t value) {
        _length = value;
    }

private:
    size_t _length;
};

int main() {
    Foo f;
    f.SetLength(1);
    return 0;
}
```

首先定义Foo的类型，只包括一个_length成员（如果类中有虚函数的话，还是首地址添加一个vtable*成员）

```
%class.Foo = type { i64 }
```

定义构造函数：注意llvm15之后使用的都是不透明指针ptr类型，而不是Foo*。可以看到这里会对成员变量初始化

```c++
define linkonce_odr dso_local void @Foo::Foo()(ptr noundef nonnull align 8 dereferenceable(8) %this) unnamed_addr comdat align 2 {
entry:
  %this.addr = alloca ptr, align 8
  store ptr %this, ptr %this.addr, align 8
  %this1 = load ptr, ptr %this.addr, align 8
  %_length = getelementptr inbounds %class.Foo, ptr %this1, i32 0, i32 0
  store i64 0, ptr %_length, align 8
  ret void
}
```

定义成员函数：

```c++
define linkonce_odr dso_local void @Foo::SetLength(unsigned long)(ptr noundef nonnull align 8 dereferenceable(8) %this, i64 noundef %value) comdat align 2 {
entry:
  %this.addr = alloca ptr, align 8
  %value.addr = alloca i64, align 8
  store ptr %this, ptr %this.addr, align 8
  store i64 %value, ptr %value.addr, align 8
  %this1 = load ptr, ptr %this.addr, align 8
  %0 = load i64, ptr %value.addr, align 8
  %_length = getelementptr inbounds %class.Foo, ptr %this1, i32 0, i32 0
  store i64 %0, ptr %_length, align 8
  ret void
}
```

main函数中使用：会alloca这个class的，并调用构造函数。

```c++
define dso_local noundef i32 @main() {
entry:
  %retval = alloca i32, align 4
  %f = alloca %class.Foo, align 8
  store i32 0, ptr %retval, align 4
  call void @Foo::Foo()(ptr noundef nonnull align 8 dereferenceable(8) %f)
  call void @Foo::SetLength(unsigned long)(ptr noundef nonnull align 8 dereferenceable(8) %f, i64 noundef 1)
  ret i32 0
}

```

### vtable in llvm ir

例子：https://compiler-explorer.com/z/afv6GPdPb

```c++
class A {
public:
  A() {a = 0;}
 virtual void foo() {};
 virtual void bar() {};
 int a;
 int b;
};
class B : public A {
public:
 B() {a = 1; b = 2; c = 3;}
 void bar() override {};
 int b;
 int c;
};

void func(A *a) {
 a->bar();
}

int main() {
    A *a = new B();
    func(a);
    return 0;
}
```

type：

```llvm
%class.A = type { ptr, i32, i32 }
%class.B = type { %class.A, i32, i32 }
```

vtable:

- 如果class上有虚函数，那么这个class都有一个vtable虚函数表
- 可以看出vtable在ir上是一个指针数组的全局变量
- 多重继承的子类中是一个多维指针数组的vtable， 如{ [4 x ptr], [3 x ptr] }（思考下为什么不采用平铺的数组方式？）并且该子类对象有多个虚表指针
- 并且编译器已经帮我们把每个vtable中的相关虚函数都放好了，相同虚函数的位置在每个vtable的索引都是固定的。因为子类会覆盖。

```
@vtable for A = linkonce_odr dso_local unnamed_addr constant { [4 x ptr] } { [4 x ptr] [
	ptr null, 
	ptr @typeinfo for A,
	ptr @A::foo(), 
	ptr @A::bar()] }, comdat, align 8
@vtable for B = linkonce_odr dso_local unnamed_addr constant { [4 x ptr] } { [4 x ptr] [
	ptr null, 
	ptr @typeinfo for B, 
	ptr @A::foo(), 
	ptr @B::bar()] }, comdat, align 8
@vtable for __cxxabiv1::__si_class_type_info = external global ptr
@vtable for __cxxabiv1::__class_type_info = external global ptr

@typeinfo name for A = linkonce_odr dso_local constant [3 x i8] c"1A\00", comdat, align 1
@typeinfo name for B = linkonce_odr dso_local constant [3 x i8] c"1B\00", comdat, align 1

@typeinfo for A = linkonce_odr dso_local constant { ptr, ptr } { 
	ptr getelementptr inbounds (ptr, ptr @vtable for __cxxabiv1::__class_type_info, i64 2), 
	ptr @typeinfo name for A }, comdat, align 8
@typeinfo for B = linkonce_odr dso_local constant { ptr, ptr, ptr } { 
	ptr getelementptr inbounds (ptr, ptr @vtable for __cxxabiv1::__si_class_type_info, i64 2), 
	ptr @typeinfo name for B, 
	ptr @typeinfo for A }, comdat, align 8
```

typeinfo：

- 首先是 type_info 方法的辅助类，是 `__cxxabiv1` 里的某个类。 对于启用了 RTTI 的类来说，
  - 所有的基础类（没有父类的类）都继承于`_class_type_info`,
  - 所有的基础类指针都继承自`__pointer_type_info`，
  - 所有的单一继承类都继承自`__si_class_type_info`，
  - 所有的多继承类都继承自`__vmi_class_type_info`。

- 然后是指向存储类型名字的指针，如果有继承关系，则最后是指向父类的 typeinfo 的记录。

B::B()构造函数初始化过程：

- 首先调用构造函数A::A()初始化，
- 如果存在虚函数的话，虚表指针被初始化指向虚表的第一个虚函数的位置。
  - 这里`getelementptr 0 2`指向`@vtable for B`第三个索引的位置，即第一个虚函数的位置
- 虚表指针存放在class结构体的第一个参数，即this指针（class对象）的基址上（注意多重继承的子类对象会有多个虚指针）
  - 这里将`@vtable for B`虚函数的基址（vtable的第三个索引）赋值到this的基址，即`%class.A = type { ptr, i32, i32 }`的ptr
- 最后对成员变量初始化`{a = 1; b = 2; c = 3;}`，分别对应：
  - %class.A = type { ptr, **i32**, i32 }
  - %class.B = type { %class.A, **i32**, i32 }  （这里可以看出如果基类和子类有相同的变量赋值时，如`b=2`，父类不可见，是对当前类变量的赋值）
  - %class.B = type { %class.A, i32, **i32** }

```c++
define linkonce_odr dso_local void @B::B()(ptr noundef nonnull align 8 dereferenceable(24) %this) unnamed_addr comdat align 2 {
entry:
  %this.addr = alloca ptr, align 8
  store ptr %this, ptr %this.addr, align 8
  %this1 = load ptr, ptr %this.addr, align 8
  call void @A::A()(ptr noundef nonnull align 8 dereferenceable(16) %this1)
  store ptr getelementptr inbounds ({ [4 x ptr] }, ptr @vtable for B, i32 0, inrange i32 0, i32 2), ptr %this1, align 8
  %a = getelementptr inbounds %class.A, ptr %this1, i32 0, i32 1
  store i32 1, ptr %a, align 8
  %b = getelementptr inbounds %class.B, ptr %this1, i32 0, i32 1
  store i32 2, ptr %b, align 8
  %c = getelementptr inbounds %class.B, ptr %this1, i32 0, i32 2
  store i32 3, ptr %c, align 4
  ret void
} 
```

调用虚函数：

首先需要取到该class的vtable地址，就是this指针的基址位置（构造函数的时候初始化好的）

取到vtable的虚函数指针，相关偏移是固定，编译器可以提前计算到，比如这里`call a->bar()`的在vtable的索引固定都是1

```
define dso_local void @func(A*)(ptr noundef %a) {
entry:
  %a.addr = alloca ptr, align 8
  store ptr %a, ptr %a.addr, align 8
  %0 = load ptr, ptr %a.addr, align 8
  %vtable = load ptr, ptr %0, align 8
  %vfn = getelementptr inbounds ptr, ptr %vtable, i64 1
  %1 = load ptr, ptr %vfn, align 8
  call void %1(ptr noundef nonnull align 8 dereferenceable(16) %0)
  ret void
}
```

https://llvm.org/devmtg/2021-11/slides/2021-RelativeVTablesinC.pdf

## getelementptr理解







