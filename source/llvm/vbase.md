# c++虚继承在llvm ir的形式

## 多重继承

详见我的上一篇笔记：https://laity000.github.io/learning-notes/llvm/vtable.html

## vbase

在普通的继承或多重继承中，一个子类和他的基类、基类的基类、...的成员变量都是嵌入在一起的，虚函数放在一个vtable上，多重继承有多个vptr。如果存在菱形继承的场景，就会导致基类的数据和虚函数存在多个。如下例子所示。

https://godbolt.org/z/KzPd4x6s9

```c++
class A {
public:
  int a;
  virtual void v();
};

class B : public A { // class B : public virtual A {
public:
  int b;
  virtual void w();
};

class C : public A { // class C : public virtual A {
public:
  int c;
  virtual void x();
};

class D : public B, public C {
public:
  long d;
  virtual void y();
};
```

在菱形继承中, A的数据（成员 a）在 D 的对象布局中存在两次，并且A 的虚函数在 vtable 中也出现两次（A::v()）。

```
                           +-----------------------+
                           |     0 (top_offset)    |
                           +-----------------------+
d --> +----------+         | ptr to typeinfo for D |
      |  vtable  |-------> +-----------------------+
      +----------+         |         A::v()        |
      |     a    |         +-----------------------+
      +----------+         |         B::w()        |
      |     b    |         +-----------------------+
      +----------+         |         D::y()        |
      |  vtable  |---+     +-----------------------+
      +----------+   |     |   -12 (top_offset)    |
      |     a    |   |     +-----------------------+
      +----------+   |     | ptr to typeinfo for D |
      |     c    |   +---> +-----------------------+
      +----------+         |         A::v()        |
      |     d    |         +-----------------------+
      +----------+         |         C::x()        |
                           +-----------------------+
```

如果我们在表示D时只能有一个A的副本，那么我们就不能再用嵌入C到D中的“技巧”（以及在D的vtable中嵌入D的C部分的vtable）来解决问题了。

使用虚继承，可以得到D的唯一一份内存布局。如下。

```
                                   +-----------------------+
                                   |   20 (vbase_offset)   |
                                   +-----------------------+
                                   |     0 (top_offset)    |
                                   +-----------------------+
                                   | ptr to typeinfo for D |
                      +----------> +-----------------------+
d --> +----------+    |            |         B::w()        |
      |  vtable  |----+            +-----------------------+
      +----------+                 |         D::y()        |
      |     b    |                 +-----------------------+
      +----------+                 |   12 (vbase_offset)   |
      |  vtable  |---------+       +-----------------------+
      +----------+         |       |    -8 (top_offset)    |
      |     c    |         |       +-----------------------+
      +----------+         |       | ptr to typeinfo for D |
      |     d    |         +-----> +-----------------------+
      +----------+                 |         C::x()        |
      |  vtable  |----+            +-----------------------+
      +----------+    |            |    0 (vbase_offset)   |
      |     a    |    |            +-----------------------+
      +----------+    |            |   -20 (top_offset)    |
                      |            +-----------------------+
                      |            | ptr to typeinfo for D |
                      +----------> +-----------------------+
                                   |         A::v()        |
                                   +-----------------------+
```

可以看到构造的时候是对每个基类平铺而不是嵌套了。

注意这里struct D的数据内存布局， 按照B、C、D、A的顺序。虚基类放在最后。与普通的继承，自身D放在最后不同。

按照8字节对齐

```mlir
%struct.D = type { %struct.B.base, [4 x i8], %struct.C.base, i64, %struct.A.base }
%struct.B.base = type <{ ptr, i32 }>
%struct.C.base = type <{ ptr, i32 }>
%struct.A.base = type <{ ptr, i32 }>
```

D的vtable对应的llvm ir
```c++
@vtable for D = linkonce_odr dso_local unnamed_addr constant { [5 x ptr], [4 x ptr], [4 x ptr] } { 
  [5 x ptr] [ptr inttoptr (i64 40 to ptr), ptr null, ptr @typeinfo for D, ptr @B::w(), ptr @D::y()], 
  [4 x ptr] [ptr inttoptr (i64 24 to ptr), ptr inttoptr (i64 -16 to ptr), ptr @typeinfo for D, ptr @C::x()], 
  [4 x ptr] [ptr null, ptr inttoptr (i64 -40 to ptr), ptr @typeinfo for D, ptr @A::v()] }, comdat, align 8
```
https://itanium-cxx-abi.github.io/cxx-abi/abi.html#vtable-components

这里比上文多个vbase字段：
- Virtual Base (vbase) offsets: 每个虚继承的派生类都会添加这个字段，当前虚表与公共基类虚表的内存位置偏移量。根据上述struct D的数据内存布局，B、C、A分别是40,24,0
  - 24 = size(%struct.C.base)12 + size(i64)8 + algin(4)
- top-vtbale--offset: 到vtable top的偏移。上述struct D中B、C、A分别是0,-16,-40
  - -16 = size(%struct.B.base)12 + size([4 x i8])4 
  - -40 = size(%struct.B.base)12 + size([4 x i8])4 + size(%struct.C.base)12 + size(i64)8 + algin(4) 

同理B-in-D的vtable llvm ir:

```c++
@construction vtable for B-in-D = linkonce_odr dso_local unnamed_addr constant { [4 x ptr], [4 x ptr] } { 
  [4 x ptr] [ptr inttoptr (i64 40 to ptr), ptr null, ptr @typeinfo for B, ptr @B::w()], 
  [4 x ptr] [ptr null, ptr inttoptr (i64 -40 to ptr), ptr @typeinfo for B, ptr @A::v()] }, comdat, align 8
```
## VTT

每new一个虚继承的子类，就会创建一个VTT。

VTT 代表 virtual-table table，即记录虚表的表。

D的VTT总计7个不同的虚指针，它们指向3张虚表的7个不同位置。拿到了指向虚表里第一个虚函数的地址的指针

如下图。

```
B-in-D
                                               +-----------------------+
                                               |   20 (vbase_offset)   |
            VTT for D                          +-----------------------+
+-------------------+                          |     0 (top_offset)    |
|    vtable for D   |-------------+            +-----------------------+
+-------------------+             |            | ptr to typeinfo for B |
| vtable for B-in-D |-------------|----------> +-----------------------+
+-------------------+             |            |         B::w()        |
| vtable for B-in-D |-------------|--------+   +-----------------------+
+-------------------+             |        |   |    0 (vbase_offset)   |
| vtable for C-in-D |-------------|-----+  |   +-----------------------+
+-------------------+             |     |  |   |   -20 (top_offset)    |
| vtable for C-in-D |-------------|--+  |  |   +-----------------------+
+-------------------+             |  |  |  |   | ptr to typeinfo for B |
|    vtable for D   |----------+  |  |  |  +-> +-----------------------+
+-------------------+          |  |  |  |      |         A::v()        |
|    vtable for D   |-------+  |  |  |  |      +-----------------------+
+-------------------+       |  |  |  |  |
                            |  |  |  |  |                         C-in-D
                            |  |  |  |  |      +-----------------------+
                            |  |  |  |  |      |   12 (vbase_offset)   |
                            |  |  |  |  |      +-----------------------+
                            |  |  |  |  |      |     0 (top_offset)    |
                            |  |  |  |  |      +-----------------------+
                            |  |  |  |  |      | ptr to typeinfo for C |
                            |  |  |  |  +----> +-----------------------+
                            |  |  |  |         |         C::x()        |
                            |  |  |  |         +-----------------------+
                            |  |  |  |         |    0 (vbase_offset)   |
                            |  |  |  |         +-----------------------+
                            |  |  |  |         |   -12 (top_offset)    |
                            |  |  |  |         +-----------------------+
                            |  |  |  |         | ptr to typeinfo for C |
                            |  |  |  +-------> +-----------------------+
                            |  |  |            |         A::v()        |
                            |  |  |            +-----------------------+
                            |  |  |
                            |  |  |                                    D
                            |  |  |            +-----------------------+
                            |  |  |            |   20 (vbase_offset)   |
                            |  |  |            +-----------------------+
                            |  |  |            |     0 (top_offset)    |
                            |  |  |            +-----------------------+
                            |  |  |            | ptr to typeinfo for D |
                            |  |  +----------> +-----------------------+
                            |  |               |         B::w()        |
                            |  |               +-----------------------+
                            |  |               |         D::y()        |
                            |  |               +-----------------------+
                            |  |               |   12 (vbase_offset)   |
                            |  |               +-----------------------+
                            |  |               |    -8 (top_offset)    |
                            |  |               +-----------------------+
                            |  |               | ptr to typeinfo for D |
                            +----------------> +-----------------------+
                               |               |         C::x()        |
                               |               +-----------------------+
                               |               |    0 (vbase_offset)   |
                               |               +-----------------------+
                               |               |   -20 (top_offset)    |
                               |               +-----------------------+
                               |               | ptr to typeinfo for D |
                               +-------------> +-----------------------+
                                               |         A::v()        |
                                               +-----------------------+
```

```c++
@VTT for D = linkonce_odr unnamed_addr constant [7 x ptr] [
  ptr getelementptr inbounds ({ [5 x ptr], [4 x ptr], [4 x ptr] }, ptr @vtable for D, i32 0, inrange i32 0, i32 3), 
  ptr getelementptr inbounds ({ [4 x ptr], [4 x ptr] }, ptr @construction vtable for B-in-D, i32 0, inrange i32 0, i32 3), 
  ptr getelementptr inbounds ({ [4 x ptr], [4 x ptr] }, ptr @construction vtable for B-in-D, i32 0, inrange i32 1, i32 3), 
  ptr getelementptr inbounds ({ [4 x ptr], [4 x ptr] }, ptr @construction vtable for C-in-D, i32 0, inrange i32 0, i32 3), 
  ptr getelementptr inbounds ({ [4 x ptr], [4 x ptr] }, ptr @construction vtable for C-in-D, i32 0, inrange i32 1, i32 3), 
  ptr getelementptr inbounds ({ [5 x ptr], [4 x ptr], [4 x ptr] }, ptr @vtable for D, i32 0, inrange i32 2, i32 3), 
  ptr getelementptr inbounds ({ [5 x ptr], [4 x ptr], [4 x ptr] }, ptr @vtable for D, i32 0, inrange i32 1, i32 3)], comdat, align 8
```

## constructor

当我们new D()时，分析下构造函数`@D::D`是如何初始化VTT和vtable的

- 首先，this指针基于top需要偏移40，调用A的构造函数
- 然后，调B的构造函数，B是主基类，top-offset是0不需要偏移this指针，并且传入VTT中B的索引指针`ptr @VTT for D, i64 0, i64 1`
  - 构造函数`@B::B`
- 接着，this指针基于top需要偏移16，调用C的构造函数， 并且传入VTT中C的索引指针`ptr @VTT for D, i64 0, i64 3`
  - 构造函数`@C::C` 
- 最后分别将D的三个vptr关联对应的vtable上

```C++
define linkonce_odr dso_local void @D::D()(ptr noundef nonnull align 8 dereferenceable(40) %0) unnamed_addr #4 comdat align 2 !dbg !63 {
  %2 = alloca ptr, align 8
  store ptr %0, ptr %2, align 8
  call void @llvm.dbg.declare(metadata ptr %2, metadata !65, metadata !DIExpression()), !dbg !67
  %3 = load ptr, ptr %2, align 8
  %4 = getelementptr inbounds i8, ptr %3, i64 40, !dbg !68
  call void @A::A()(ptr noundef nonnull align 8 dereferenceable(12) %4) #7, !dbg !68
  call void @B::B()(ptr noundef nonnull align 8 dereferenceable(12) %3, ptr noundef getelementptr inbounds ([7 x ptr], ptr @VTT for D, i64 0, i64 1)) #7, !dbg !68
  %5 = getelementptr inbounds i8, ptr %3, i64 16, !dbg !68
  call void @C::C()(ptr noundef nonnull align 8 dereferenceable(12) %5, ptr noundef getelementptr inbounds ([7 x ptr], ptr @VTT for D, i64 0, i64 3)) #7, !dbg !68
  store ptr getelementptr inbounds ({ [6 x ptr], [4 x ptr], [4 x ptr] }, ptr @vtable for D, i32 0, inrange i32 0, i32 3), ptr %3, align 8, !dbg !68
  %6 = getelementptr inbounds i8, ptr %3, i64 40, !dbg !68
  store ptr getelementptr inbounds ({ [6 x ptr], [4 x ptr], [4 x ptr] }, ptr @vtable for D, i32 0, inrange i32 2, i32 3), ptr %6, align 8, !dbg !68
  %7 = getelementptr inbounds i8, ptr %3, i64 16, !dbg !68
  store ptr getelementptr inbounds ({ [6 x ptr], [4 x ptr], [4 x ptr] }, ptr @vtable for D, i32 0, inrange i32 1, i32 3), ptr %7, align 8, !dbg !68
  ret void, !dbg !68
}
```
## 引用

- [What is the VTT for a class?](https://stackoverflow.com/questions/6258559/what-is-the-vtt-for-a-class)
- https://www.wenfh2020.com/2023/08/22/cpp-inheritance/#top