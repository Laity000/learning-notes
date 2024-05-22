
[CppTemplateTutorial](https://github.com/wuye9036/CppTemplateTutorial)


2. Template的基本语法
2.1. 什么是模板(Template)
2.2. 类模板 (Class Template) 的基本语法
2.2.1. “模板类”还是“类模板”
2.2.2. Class Template的与成员变量定义
2.2.3. 模板的使用
2.2.4. 类模板的成员函数定义
2.3. 函数模板 (Function Template) 入门
2.3.1. 函数模板的声明和定义
2.3.2. 函数模板的使用
2.4. 整型也可是Template参数
2.5. 模板形式与功能是统一的
3. 模板元编程基础
3.1. 编程，元编程，模板元编程
3.2. 模板世界的If-Then-Else：类模板的特化与偏特化
3.2.1. 根据类型执行代码
3.2.2. 特化
3.2.3. 特化：一些其它问题
3.3. 即用即推导
3.3.1. 视若无睹的语法错误
3.3.2. 名称查找：I am who I am
3.3.3. “多余的” typename 关键字
3.4. 本章小结
4. 深入理解特化与偏特化
4.1. 正确的理解偏特化
4.1.1. 偏特化与函数重载的比较
4.1.2. 不定长的模板参数
4.1.3. 模板的默认实参
4.2. 后悔药：SFINAE
4.3. Concept “概念”：对模板参数约束的直接描述
4.3.1. “概念” 解决了什么问题
4.3.2. "概念"入门
5. 未完成章节

我们把通过类型绑定将类模板变成“普通的类”的过程，称之为模板实例化（Template Instantiate）。

归根结底，模板无外乎两点：

 - 函数或者类里面，有一些类型我们希望它能变化一下，我们用标识符来代替它，这就是“模板参数”；

 - 在需要这些类型的地方，写上相对应的标识符（“模板参数”）。

当然，这里的“可变”实际上在代码编译好后就固定下来了，可以称之为**编译期的可变性**。

# 类模版

在类外的成员函数实现的时候，必须要提供模板参数。

## 函数模版
函数模板的调用格式是：

`函数模板名 < 模板参数列表 > ( 参数 )`

不是所有的模版参数都能推导出来，比如模版参数用在返回值上。--> 所以，先写需要指定的模板参数，再把能推导出来的模板参数放在后面。

模板参数除了类型（`typename`关键字 ）外（包括基本类型、结构、类类型等），也可以是一个整型数（Integral Number）。这里的整型数比较宽泛，包括布尔型，不同位数、有无符号的整型，甚至包括指针。

```c++
template <uint8_t a, typename b, void* c> class B {};
template <bool, void (*a)()> class C {};
template <void (A<3>::*a)(int)> class D {};
template <int i> int Add(int a)	// 函数模板


B<7, A<5>, nullptr>	b; // 模板参数可以是一个无符号八位整数，可以是模板生成的类；可以是一个指针。
C<false, &foo> c;      // 模板参数可以是一个bool类型的常量，甚至可以是一个函数指针。
D<&A<3>::foo> d;       // 丧心病狂啊！它还能是一个成员函数指针！
int x = Add<3>(5); 
```
## 模版元编程（meta）

元编程中所有变化的量（或者说元编程的参数），都是类型，那么这样的编程，我们有个特定的称呼，叫“泛型”。

根据类型执行不同代码。要达成这一目的，模板并不是唯一的途径。比如之前我们所说的重载。


在模板代码中，这个“合适的机制”就是指“特化”和“部分特化（Partial Specialization）”，后者也叫“偏特化”。

部分特化/偏特化 和 特化 相当于是模板实例化过程中的if-then-else。这使得我们根据不同类型，选择不同实现的需求得以实现。

特化：
```c++
template <> struct X<int, float>
```

特化的时候，当所有类型都已经确定，我们就可以抛弃全部的模板参数，写出上面这样的形式：因为所有列表中所有参数都确定了，就不需要额外的形式参数了。

偏特化：
```c++
template <typename T, typename U> struct X            ;    // 0 
                                                           // 原型有两个类型参数
                                                           // 所以下面的这些偏特化的实参列表
                                                           // 也需要两个类型参数对应
template <typename T>             struct X<T,  T  > {};    // 1
template <typename T>             struct X<T*, T  > {};    // 2
template <typename T>             struct X<T,  T* > {};    // 3
template <typename U>             struct X<U,  int> {};    // 4
template <typename U>             struct X<U*, int> {};    // 5
template <typename U, typename T> struct X<U*, T* > {};    // 6
template <typename U, typename T> struct X<U,  T* > {};    // 7

template <typename T>             struct X<unique_ptr<T>, shared_ptr<T>>; // 8

// 以下特化，分别对应哪个偏特化的实例？
// 此时偏特化中的T或U分别是什么类型？

X<float*,  int>      v0;                       
X<double*, int>      v1;                       
X<double,  double>   v2;                          
X<float*,  double*>  v3;                           
X<float*,  float*>   v4;                          
X<double,  float*>   v5;                          
X<int,     double*>  v6;                           
X<int*,    int>      v7;                       
X<double*, double>   v8;
```
除了简单的指针、const和volatile修饰符，其他的类模板也可以作为偏特化时的“模式”出现，例如示例8，它要求传入同一个类型的unique_ptr和shared_ptr。

特化和它原型的类模板之间，是分别独立实现的。比如原型中的成员变量，特化没有使用时会报错.

模板是从最特殊到最一般形式进行匹配的

但是1和6，就很难说清楚谁更好了。一个说明了两者类型相同；另外一个则说明了两者都是指针。所以在这里，编译器也没办法决定使用那个，只好报出了编译器错误。

### 双阶段名称查找（Two phase name lookup）

名称查找/名称解析，是编译器的基石。对编译原理稍有了解的人，都知道“符号表”的存在及重要意义。考虑一段最基本的C代码

按照标准的意思，名称查找会在模板定义和实例化时各做一次，分别处理非依赖性名称和依赖性名称的查找。

```c++
template <typename T> struct Y
{
    // X可以查找到原型；
    // X<T>是一个依赖性名称，模板定义阶段并不管X<T>是不是正确的。
    typedef X<T> ReboundType;
	
    // X可以查找到原型；
    // X<T>是一个依赖性名称，X<T>::MemberType也是一个依赖性名称；
    // 所以模板声明时也不会管X模板里面有没有MemberType这回事。
    typedef typename X<T>::MemberType MemberType2;
	
    // UnknownType 不是一个依赖性名称
    // 而且这个名字在当前作用域中不存在，所以直接报错。
    typedef UnknownType MemberType3;				
};
```

### typename是做什么的？

简单来说，如果编译器能在出现的时候知道它是一个类型，那么就不需要typename，比如正常的表达式`a * b`。如果必须要到实例化的时候才能知道它是不是合法，那么定义的时候就把这个名称作为变量而不是类型。

```c++
template <typename T> void meow()
{
    T::a * b; // 这是指针定义还是表达式语句？
}
```
因此，C++标准规定，在没有typename约束的情况下认为这里T::a不是类型，因此T::a * b; 会被当作表达式语句（例如乘法）；而为了告诉编译器这是一个指针的定义，我们必须在T::a之前加上typename关键字，告诉编译器T::a是一个类型，这样整个语句才能符合指针定义的语法。

```c++
struct A;
template <typename T> struct B;
template <typename T> struct X {
    typedef X<T> TA; // 编译器当然知道 X<T> 是一个类型。
    typedef X    TB; // X 等价于 X<T> 的缩写
    typedef T    TC; // T 不是一个类型还玩毛
    
    // ！！！注意我要变形了！！！
    class Y {
        typedef X<T>     TD;          // X 的内部，既然外部高枕无忧，内部更不用说了
        typedef X<T>::Y  TE;          // 嗯，这里也没问题，编译器知道Y就是当前的类型，
                                      // 这里在VS2015上会有错，需要添加 typename，
                                      // Clang 上顺利通过。
        typedef typename X<T*>::Y TF; // 这个居然要加 typename！
                                      // 因为，X<T*>和X<T>不一样哦，
                                      // 它可能会在实例化的时候被别的偏特化给抢过去实现了。
    };
    
    typedef A TG;                   // 嗯，没问题，A在外面声明啦
    typedef B<T> TH;                // B<T>也是一个类型
    typedef typename B<T>::type TI; // 嗯，因为不知道B<T>::type的信息，
                                    // 所以需要typename
    typedef B<int>::type TJ;        // B<int> 不依赖模板参数，
                                    // 所以编译器直接就实例化（instantiate）了
                                    // 但是这个时候，B并没有被实现，所以就出错了
};
```

名称查找是语义分析的一个环节，模板内书写的 变量声明、typedef、类型名称 甚至 类模板中成员函数的实现 都要符合名称查找的规矩才不会出错；

C++编译器对语义的分析的原则是“大胆假设，小心求证”：在能求证的地方尽量求证 —— 比如两段式名称查找的第一阶段；无法检查的地方假设你是正确的 —— 比如typedef typename A<T>::MemberType X;在模板定义时因为T不明确不会轻易判定这个语句的死刑。


## 深入理解特化和偏特化

### 编译器完成整个模板匹配过程的场景

```c++
template <typename T> struct DoWork;	      // (0) 这是原型

template <> struct DoWork<int> {};            // (1) 这是 int 类型的特化
template <> struct DoWork<float> {};          // (2) 这是 float 类型的特化
template <typename U> struct DoWork<U*> {};   // (3) 这是指针类型的偏特化

DoWork<int>    i;  // (4)
DoWork<float*> pf; // (5)
```

首先，编译器分析(0), (1), (2)三句，得知(0)是模板的原型，(1)，(2)，(3)是模板(0)的特化或偏特化。我们假设有两个字典，第一个字典存储了模板原型，我们称之为`TemplateDict`。第二个字典`TemplateSpecDict`，存储了模板原型所对应的特化/偏特化形式。所以编译器在处理这几句时，可以视作

```c++
// 以下为伪代码
TemplateDict[DoWork<T>] = {
    DoWork<int>,
    DoWork<float>,
    DoWork<U*>                     
};
```

然后 (4) 试图以int实例化类模板DoWork。它会在TemplateDict中，找到DoWork，它有一个形式参数T接受类型，正好和我们实例化的要求相符合。并且此时T被推导为int。(5) 中的float*也是同理。

```c++
{   // 以下为 DoWork<int> 查找对应匹配的伪代码
    templateProtoInt = TemplateDict.find(DoWork, int);    // 查找模板原型，查找到(0)
    template = templatePrototype.match(int);              // 以 int 对应 int 匹配到 (1)
}

{   // 以下为DoWork<float*> 查找对应匹配的伪代码
    templateProtoIntPtr = TemplateDict.find(DoWork, float*) // 查找模板原型，查找到(0)
    template = templateProtoIntPtr.match(float*)            // 以 float* 对应 U* 匹配到 (3)，此时U为float
}
```
### 默认实参（Default Arguments）☆

```c++
template <typename T0, typename T1 = void> struct DoWork;
```

实际上，模板的默认参数不仅仅可以是一个确定的类型，它还能是以其他类型为参数的一个类型表达式。

```c++
include <complex>
include <type_traits>

template <typename T> T CustomDiv(T lhs, T rhs) {
    T v;
    // Custom Div的实现
    return v;
}

template <typename T, typename Enabled = std::true_type> struct SafeDivide {
    static T Do(T lhs, T rhs) {
        return CustomDiv(lhs, rhs);
    }
};

template <typename T> struct SafeDivide<
    T, typename std::is_floating_point<T>::type>{    // 偏特化A
    static T Do(T lhs, T rhs){
        return lhs/rhs;
    }
};

template <typename T> struct SafeDivide<
    T, typename std::is_integral<T>::type>{          // 偏特化B
    static T Do(T lhs, T rhs){
        return rhs == 0 ? 0 : lhs/rhs;
    }
};

void foo(){
    SafeDivide<float>::Do(1.0f, 2.0f);	// 调用偏特化A
    SafeDivide<int>::Do(1, 2);          // 调用偏特化B
    SafeDivide<std::complex<float>>::Do({1.f, 2.f}, {1.f, -2.f});
}
```
我们借助这个例子，帮助大家理解一下这个结构是怎么工作的：

对SafeDivide<int>
通过匹配类模板的泛化形式，计算默认实参，可以知道我们要匹配的模板实参是SafeDivide<int, true_type>

计算两个偏特化的形式的匹配：A得到<int, false_type>,和B得到 <int, true_type>

最后偏特化B的匹配结果和模板实参一致，使用它。

针对SafeDivide<complex<float>>
通过匹配类模板的泛化形式，可以知道我们要匹配的模板实参是SafeDivide<complex<float>, true_type>

计算两个偏特化形式的匹配：A和B均得到SafeDivide<complex<float>, false_type>

A和B都与模板实参无法匹配，所以使用原型，调用CustomDiv

### 变参模板（Variadic Template）

```c++
template <typename... Ts, typename U> class X {};              // (1) error!
template <typename... Ts>             class Y {};              // (2)
template <typename... Ts, typename U> class Y<U, Ts...> {};    // (3)
template <typename... Ts, typename U> class Y<Ts..., U> {};    // (4) error!
```

为什么(3)中， 模板参数和(1)相同，都是typename... Ts, typename U，但是编译器却并没有报错呢？

答案在这一节的早些时候。(3)和(1)不同，它并不是模板的原型，它只是Y的一个偏特化。回顾我们在之前所提到的，偏特化时，模板参数列表并不代表匹配顺序，它们只是为偏特化的模式提供的声明，也就是说，它们的匹配顺序，只是按照<U, Ts...>来，而之前的参数只是告诉你Ts是一个类型列表，而U是一个类型，排名不分先后。

## SFINAE

http://kaiyuan.me/2018/05/08/sfinae/

SFINAE 的含义应该已经比较清楚：编译器在将函数模板形参替换成实参的过程中，如果针对某个函数模板产生了不合法的代码，其不会直接抛出错误信息，而是继续尝试去匹配其他可能的重载函数。

SFINAE最主要的作用，是保证编译器在泛型函数、偏特化、及一般重载函数中遴选函数原型的候选列表时不被打断。除此之外，它还有一个很重要的元编程作用就是实现部分的编译期自省和反射。