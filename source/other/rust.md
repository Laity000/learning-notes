
# rust grammar note

- [Rust 程序设计语言 简体中文版](https://kaisery.github.io/trpl-zh-cn/title-page.html)

## 变量
let：变量默认不可改，需要改变时加mut修饰

const：常量可以在任何作用域中声明，包括全局作用域，这在一个值需要被很多部分的代码用到时很有用

mut变量 与 隐藏（同名变量再赋值）的区别：隐藏可以改变类型

## 数据类型



## 所有权

变量与数据交互的方式：移动，克隆

这里还隐含了一个设计选择：Rust 永远也不会自动创建数据的 “深拷贝”。因此，任何 自动 的复制都可以被认为是对运行时性能影响较小的。

- 变量的作用域。当变量离开作用域，Rust 为我们调用一个特殊的函数。这个函数叫做 drop，在这里 String 的作者可以放置释放内存的代码。Rust 在结尾的` } `处自动调用 drop。
- 栈上变量（基本类型）不用考虑所有权，有Copy trait特性：不需要clone函数了，如果一个类型实现了 Copy trait，那么一个旧的变量在将其赋值给其他变量后仍然可用。
- 任何一组简单标量值的组合都可以实现 Copy，任何不需要分配内存或某种形式资源的类型都可以实现 Copy 。如下是一些 Copy 的类型：

## 引用
- 引用：有一个对该变量的可变引用，你就不能再创建对该变量的引用，避免数据竞争
- 在任意给定时间，要么 只能有一个可变引用，要么 只能有多个不可变引用。

可变引用：
```rust
fn main() {
    let mut s = String::from("hello");

    change(&mut s);
}

fn change(some_string: &mut String) {
    some_string.push_str(", world");
}
```
- 创建一个引用的行为称为 借用

- slice 允许你引用集合中一段连续的元素序列，而不用引用整个集合。

- 字符串字面值就是 slice

let s = "Hello, world!";
这里 s 的类型是 &str：它是一个指向二进制程序特定位置的 slice。这也就是为什么字符串字面值是不可变的；&str 是一个不可变引用。

## struct

 - 我们可以使用 `字段初始化简写语法（field init shorthand）`来重写 build_user，这样其行为与之前完全相同，不过无需重复 username 和 email 了
文件名：src/main.rs
```rust
fn build_user(email: String, username: String) -> User {
    User {
        active: true,
        username,
        email,
        sign_in_count: 1,
    }
}
```

- 使用结构体更新语法为一个 User 实例设置一个新的 email 值，不过其余值来自 user1 变量中实例的字段

- 元组结构体有着结构体名称提供的含义，但没有具体的字段名，只有字段的类型。`struct Color(i32, i32, i32);`

## function

- &self 实际上是 self: &Self 的缩写。在一个 impl 块中，Self 类型是 impl 块的类型的别名。方法的第一个参数必须有一个名为 self 的Self 类型的参数，所以 Rust 让你在第一个参数位置上只用 self 这个名字来简化。

- `自动引用和解引用（automatic referencing and dereferencing）的功能`。方法调用是 Rust 中少数几个拥有这种行为的地方。

它是这样工作的：当使用 object.something() 调用方法时，Rust 会自动为 object 添加 &、&mut 或 * 以便使 object 与方法签名匹配。也就是说，这些代码是等价的：

p1.distance(&p2);
(&p1).distance(&p2);
第一行看起来简洁的多。这种自动引用的行为之所以有效，是因为方法有一个明确的接收者———— self 的类型。在给出接收者和方法名的前提下，Rust 可以明确地计算出方法是仅仅读取（&self），做出修改（&mut self）或者是获取所有权（self）。

### `关联函数`
所有在 impl 块中定义的函数被称为 `关联函数（associated functions）`，因为它们与 impl 后面命名的类型相关。我们可以定义不以 self 为第一参数的关联函数（因此不是方法），因为它们并不作用于一个结构体的实例。我们已经使用了一个这样的函数：在 String 类型上定义的 String::from 函数。

不是方法的关联函数经常被用作返回一个结构体新实例的构造函数。这些函数的名称通常为 new ，但 new 并不是一个关键字。例如我们可以提供一个叫做 square 关联函数，它接受一个维度参数并且同时作为宽和高，这样可以更轻松的创建一个正方形 Rectangle 而不必指定两次同样的值：

文件名：src/main.rs

impl Rectangle {
    fn square(size: u32) -> Self {
        Self {
            width: size,
            height: size,
        }
    }
}
关键字 Self 在函数的返回类型中代指在 impl 关键字后出现的类型，在这里是 Rectangle

## emun
可以将任意类型的数据放入枚举成员中：例如字符串、数字类型或者结构体。甚至可以包含另一个枚举！

```rust
struct Ipv4Addr {
    // --snip--
}

struct Ipv6Addr {
    // --snip--
}

enum IpAddr {
    V4(Ipv4Addr),
    V6(Ipv6Addr),
}
```

结构体和枚举还有另一个相似点：就像可以使用 impl 来为结构体定义方法那样，也可以在枚举上定义方法。


enum Option<T> {
    None,
    Some(T),
}

### match 控制流结构

绑定值的模式
匹配分支的另一个有用的功能是可以绑定匹配的模式的部分值。这也就是如何从枚举成员中提取值的。

```rust
#[derive(Debug)] // 这样可以立刻看到州的名称
enum UsState {
    Alabama,
    Alaska,
    // --snip--
}

enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter(UsState),
}
fn value_in_cents(coin: Coin) -> u8 {
    match coin { // 枚举变量
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter(state) => {
            println!("State quarter from {:?}!", state);
            25
        }
    }
}
```


匹配 Option<T>

```rust
    fn plus_one(x: Option<i32>) -> Option<i32> {
        match x {
            None => None,
            Some(i) => Some(i + 1),
        }
    }

    let five = Some(5);
    let six = plus_one(five);
    let none = plus_one(None);
```

Rust 中的匹配是 穷尽的（exhaustive）：必须穷举到最后的可能性来使代码有效。


通配模式和 _ 占位符

```rust
    let dice_roll = 9;
    match dice_roll {
        3 => add_fancy_hat(),
        7 => remove_fancy_hat(),
        _ => reroll(), //或者other
        // _ => (),
    }

    fn add_fancy_hat() {}
    fn remove_fancy_hat() {}
    fn reroll() {}
```

> 具体看18节：https://kaisery.github.io/trpl-zh-cn/ch18-01-all-the-places-for-patterns.html

### if let 简洁控制流

换句话说，可以认为 if let 是 match 的一个语法糖，它当值匹配某一模式时执行代码而忽略所有其他值。

可以在 if let 中包含一个 else。else 块中的代码与 match 表达式中的 _ 分支块中的代码相同，这样的 match 表达式就等同于 if let 和 else。

```rust
    let mut count = 0;
    if let Coin::Quarter(state) = coin {
        println!("State quarter from {:?}!", state);
    } else {
        count += 1;
    }
```

## trait

[详解 Rust 的 trait，通过接口让你的类型实现多态](https://www.cnblogs.com/traditional/p/17818185.html)

 - trait 就是 Rust 中的接口的概念，用于给泛型施加一个约束。（我理解trait+T是一个接口/基类）实现的类型（子类）必须要满足（实现）接口里的行为（方法）。如果trait里的方法默认实现了，子类就不需要实现了 --->我理解就类似c++里的通过元编程实现的静态多态https://www.kindem.xyz/post/39/ 核心思想都是约束行为
 - [奇异递归模板模式(Curiously Recurring Template Pattern)](https://zhuanlan.zhihu.com/p/54945314)


```rust
// 定义了两个方法，但 eat 方法有默认的实现
trait Animal {
    // 有声明，有实现（函数体）
    fn eat(&self) {
        println!("Animal 在吃东西")
    }
    // 只有声明，没有实现（不存在函数体）
    fn drink(&self);
}

struct Dog {
    name: String,
    category: &'static str
}

impl Animal for Dog {

    fn drink(&self) {
        println!("{} 在喝饮料，它是一只 {}", self.name, self.category);
    }
}
// 这里其实就可以理解参数是一个具有eat/drink行为的基类
//Rust 会先检测传给参数 的值是否能eat/drink
fn eat<T: Animal>(animal: &T) {
    animal.eat();
}
fn drink<T: Animal>(animal: &T) {
    animal.drink();
}

fn main() {
    let dog = Dog{name: "旺财".to_string(), category: "小狗"};
    let cat = Cat{name: "翠花".to_string(), category: "小猫"};
    // 没有实现 eat 方法，此时调用的是 trait 的默认实现
    eat(&dog);
    drink(&dog);  // 旺财 在喝饮料，它是一只 小狗
}
```
- Rust 要求所有变量都有类型

# 包管理

- crate 是 Rust 在编译时最小的代码单位。
- crate 可以包含模块，模块可以定义在其他文件，然后和 crate 一起编译，我们会在接下来的章节中遇到。
- 包（package）是提供一系列功能的一个或者多个 crate。一个包会包含一个 Cargo.toml 文件，阐述如何去构建这些 crate。Cargo 就是一个包含构建你代码的二进制项的包。Cargo 也包含这些二进制项所依赖的库。其他项目也能用 Cargo 库来实现与 Cargo 命令行程序一样的逻辑。
- 包中可以包含至多一个库 crate(library crate)。包中可以包含任意多个二进制 crate(binary crate)，但是必须至少包含一个 crate（无论是库的还是二进制的）。
- 如果一个包同时含有 src/main.rs 和 src/lib.rs，则它有两个 crate：一个二进制的和一个库的，且名字都与包相同。通过将文件放在 src/bin 目录下，一个包可以拥有多个二进制 crate：每个 src/bin 下的文件都会被编译成一个独立的二进制 crate。

### 模块

- 模块 让我们可以将一个 crate 中的代码进行分组，以提高可读性与重用性。因为一个模块中的代码默认是私有的，所以还可以利用模块控制项的 私有性。私有项是不可为外部使用的内在详细实现。我们也可以将模块和它其中的项标记为公开的，这样，外部代码就可以使用并依赖与它们。
使用模块：`pub mod garden;`
定义模块；
```rust
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}

        fn seat_at_table() {}
    }

    mod serving {
        fn take_order() {}

        fn serve_order() {}

        fn take_payment() {}
    }
}
```

Rust 如何在模块树中找到一个项的位置，我们使用路径的方式，就像在文件系统使用路径一样。为了调用一个函数，我们需要知道它的路径。

路径有两种形式：

绝对路径（absolute path）是以 crate 根（root）开头的全路径；对于外部 crate 的代码，是以 crate 名开头的绝对路径，对于当前 crate 的代码，则以字面值 crate 开头。
相对路径（relative path）从当前模块开始，以 self、super 或定义在当前模块中的标识符开头。

### 使用 pub 关键字暴露路径
- 模块公有并不使其内容也是公有的。模块上的 pub 关键字只允许其父模块引用它，而不允许访问内部代码。因为模块是一个容器，只是将模块变为公有能做的其实并不太多；同时需要更深入地选择将一个或多个项变为公有。
- 如果我们在一个结构体定义的前面使用了 pub ，这个结构体会变成公有的，但是这个结构体的字段仍然是私有的。

- 如果我们将枚举设为公有，则它的所有成员都将变为公有。我们只需要在 enum 关键字前面加上 pub，

### 使用 use 关键字将路径引入作用域

在作用域中增加 use 和路径类似于在文件系统中创建软连接（符号连接，symbolic link）。

-  use 语句只适用于其所在的作用域，如下错误
```rust
use crate::front_of_house::hosting;

mod customer {
    pub fn eat_at_restaurant() {
        hosting::add_to_waitlist();
    }
}
```
- use 将函数引入作用域的习惯用法: 要想使用 use 将函数的父模块引入作用域，我们必须在调用函数时指定父模块，这样可以清晰地表明函数不是在本地定义的，同时使完整路径的重复度最小化。
- 如你所见，使用父模块可以区分这两个 Result 类型。如果我们是指定 use std::fmt::Result 和 use std::io::Result，我们将在同一作用域拥有了两个 Result 类型，当我们使用 Result 时，Rust 则不知道我们要用的是哪个。
- 还有另一个解决办法：在这个类型的路径后面，我们使用 as 指定一个新的本地名称或者别名。

- 注意你只需在模块树中的某处使用一次 mod 声明就可以加载这个文件。一旦编译器知道了这个文件是项目的一部分（并且通过 mod 语句的位置知道了代码在模块树中的位置），项目中的其他文件应该使用其所声明的位置的路径来引用那个文件的代码，这在“引用模块项目的路径”部分有讲到。换句话说，mod 不是 你可能会在其他编程语言中看到的 "include" 操作。

### 使用 pub use 重导出名称



### 使用外部包

本地包名：crate 比如：`use crate::front_of_house::hosting::add_to_waitlist;`
在第二章中我们编写了一个猜猜看游戏。那个项目使用了一个外部包，rand，来生成随机数。为了在项目中使用 rand，在 Cargo.toml 中加入了如下行：

文件名：Cargo.toml

rand = "0.8.5"
在 Cargo.toml 中加入 rand 依赖告诉了 Cargo 要从 crates.io 下载 rand 和其依赖，并使其可在项目代码中使用。

接着，为了将 rand 定义引入项目包的作用域，我们加入一行 use 起始的包名，
```rust
use rand::Rng;

fn main() {
    let secret_number = rand::thread_rng().gen_range(1..=100);
}
```

嵌套路径来消除大量的 use 行
```rust
use std::{cmp::Ordering, io};
```

如果希望将一个路径下 所有 公有项引入作用域，可以指定路径后跟 *，glob 运算符

。使用 glob 运算符时请多加小心！Glob 会使得我们难以推导作用域中有什么名称和它们是在何处定义的。