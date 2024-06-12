---
issueid: 53
tags:
- 编程语言
- Rust
title: Rust浅尝
date: 2020-11-01
updated: 2022-09-12
---
# Rust 基本语法学习

## 为什么会有此文

目标是给 Deno 项目贡献代码，结果发现我还不会 Rust ......，所以要先啃 Rust

我发现 rust，ruby 和掌握的 C 系风格语言很不同。

我从 C# 转 Java 时根本没花时间『特地』学习语法，因为他们两个太像了，直接找了一份开源代码对着抄，边抄边查文档就会了。

然而当我开始学 Rust 时我看到这语法就蒙蔽了。

你很习惯用之前的经验来套，结果发现，卧槽，你 `return` 哪去了， for 循环呢？ `'static`又是个啥，好不容易看到个 \<T> 以为碰到泛型了，结果后面的 where 又是个啥，搜完才发现类似于 T extends String。

语法都看不懂何谈抄？

所以决定将我 抄 官方文档的 example 和开源代码的过程作为笔记沉淀下来 :)

开始正文

学了这么多语言，我也有自己的学习流程

1. 模块系统，如何引入类（相对，绝对路径），模块的最小单元是由约束，关键词？文件？
2. 类型系统，是否区分 primitive 和 Object， pass by value or reference?
3. GC系统
4. 生态系统，有包管理器吗，如何安装第三方包
5. 线程模型，1:1 还是 M:N，是否实现了 yield, await, async, Promise, Future 等等异步模型
6. 语言特性，Green Thread，Ownership等等
7. 关键字，预处理，宏定义等杂项

上述七步互相穿插并持续配合官方文档，知乎文章，搜索引擎

## 模块引用-模块系统

这是某个文件的内容

```
use crate::{event, sys, Events, Interest, Token};
use log::trace;
#[cfg(unix)]
use std::os::unix::io::{AsRawFd, RawFd};
use std::time::Duration;
use std::{fmt, io};
```

可以看到，对于自己包使用 use crate，当引用外部包时使用 use std:: 或者 use log::trace
std:: 是 rust 的标准库

rust的模块系统是个树形结构，全部模块都挂载到 crate 这个root上，而以 `crate::xxx` 的方式引用模块被称为 绝对路径 引用

```
crate
 └── front_of_house
     ├── hosting
     │   ├── add_to_waitlist
     │   └── seat_at_table
     └── serving
         ├── take_order
         ├── serve_order
         └── take_payment
```

```
use crate::{event, sys};
```

```
src
 └── event
     ├── mod.rs
     │
     │
     sys 
     ├─ mod.rs
```

引用自己的代码用 crate 开头，比如 use crate::{ event };

如果是外部库比如 log，就使用 use log;

use park;

1. 首先查找 park.rs，找到停止
2. 查找park/mod.rs文件

上述查找算法不运行同时拥有 park/mod.rs 和 park.rs

(pub)mod park;(向外)声明模块，使用上述查找算法

**注意**

Rust中不以文件，而以 mod 关键字定义模块，如果你的文件没有 mod 关键词，那你的代码就挂载在父文件夹所在的模块

mod event; 向编译器声明一个模块，这样编译器才知道这个模块的存在并将其引入到 module tree 里，如下图所示

![image](https://user-images.githubusercontent.com/24750337/97392445-bddba800-191c-11eb-8991-ff8cc2d155b9.png)

我觉着对于模块，下面这句话**最精髓**

**mod 串成的树可以让编译器找到你全部的代码并编译**

**而 pub 可以让你显式将符号向其他模块导出**

如果你向接近一个符号，那从这个符号的最左侧一直到最右侧都必须 accessable

```rust
// src/event/event.rs
pub struct Event {
    inner: sys::Event,
}

// 这点和 js，python很不同
// js 使用文件路径直接就可以查找，而不需要先声明存在这个模块
// src/event/mod.rs
mod event;
mod events;
mod source;

pub use self::event::Event;
pub use self::events::{Events, Iter};
pub use self::source::Source;

```

> use crate::event::Event;

1. 找到src/
2. 找到event/mod.rs，发现定义了 event 模块(mod event;)
3. event.rs 中 pub 将Event从 event.rs 模块导出
4. pub use self::events::Event 先将模块导入 mod.rs 之后利用 pub 二次导出
5. 于是可以访问 Event struct

struct 只是个Item，可以替换为 function, trait

pub 会将 item 挂载到上一级 mod，如果直接在src/ 目录下，那就挂载到 crate 这个 Root module

### 约定优于配置

我们的代码往往有两种用途

##### 作为 binary

rust约定当 src/main.rs 存在时rust可以生成可执行文件

#### 作为 library

rust约定当 src/lib.rs 存在时 rust作为库文件使用，引用这个库时 rust 会读取 src/lib.rs 的信息

src/main.rs 和 src/lib.rs 可以共存，这时项目既可作为 binary 又可为 library，你引用库时需要使用库名

```toml
[package]
name = "minigrep"
version = "0.1.0"
authors = [""]
edition = "2018"
```

在main.rs引用lib时按照下面方式

```
use minigrep::print;
```

这很好理解，当开发者用你的库时也是以库名开头，这样 rust 才知道你引用的是谁，所以使用库名引用自己库代码时可提供一致的体验。

##### mod utils和 use utils的区别是

```
mod utils;
use utils::print;
```

mod告诉 rust 这里存在一个 module utils，请去找到 `utils/lib.rs` 或者 `utils.rs` 并挂载到 rust 的模块树

use 是将**已经**挂载到 模块树 的符号引入当前 scope，所以不要看到 use 就想到 name lookup 和 图 理论。

所以我们在 main 里为了使用自己的 mod，需要先 mod 声明挂载到模块树，然后 use 将符号引入 scope，main是入口点，main 之前并没有引入别的mod，所以我们要先 mod xxx 声明。

Rust 会从 main.rs 或者 lib.rs 这个Root开始寻找 mod xxx 来进行模块查找并挂载到 crate root

这和 JS 很不同， import 会 **查找并将这个符号引入**，rust 拆成了两步

和 Java 也不同, import com.chaochaogege.utils; 会 **查找并引入** 这个 class

https://github.com/rust-lang/book/issues/460

---

`use a::b::c` 中的 a 一定要是 lib名字

比如

```rust
// 自己库 error 下的 bad_resource_id
// 这里查找到了 src/error.rs 文件下的 bad_resource_id
use crate::error::bad_resource_id;
//第三方库 serde_json 下的 json
use serde_json::json;
// 标准库 std 下的 pin::Pin
use std::pin::Pin;
```

对于自己编写的 crate 必须**遵循先声明后使用**

```rs
// lib.rs
// 必须先声明存在 mod编译器才会去寻找 unix
// 因为下面 Selector 已经导入 unix/mod.rs 所以直接能从 unix:: Selector 引入
mod unix;
use crate::unix::Selector;

// src/unix/mod.rs
// 先声明当前目录下存在 selector 模块
mod selector;

// 将 selector 模块引入到 当前mod.rs并导出
pub(crate) use self::selector::{Selector};

// src/unix/selector.rs
pub struct Selector {
    id: usize,
}
```

如果使用第三方 crate 无须声明直接用 crate 名字引入

```rs
// 使用名字直接引入cargo.toml 中的 ntapi 依赖
use ntapi::ntrtl::RtlNtStatusToDosError;
```

#### C++ 的模块系统

我想尝试理解 rust 的模块系统为什么这么设计，所以借这个机会谈一下 CPP 模块系统

我们都有个习惯，http.h 存放 http 函数的声明，而http.cc 包含着 http 具体的实现逻辑

当你在main函数引用 http 的实现时会这样

```cpp
#include "http.h"
```

```cpp
// http.cc
#include "http.h"
int run(){
    return 0;
}

```

我们并没有在任何地方引用 http.cc ，那最后编译器如何将你的实现引入呢？

关键点在编译你的项目时你需要 gcc main.cpp http.cc -o main

编译器看到 main 中引用了 http.h，他就会在符号表记录这个符号，但他并不知道这个符号到底在哪，所以我们需要主动将这个 http.cc 文件编译进去，生成 http.o，编译器发现 http.o 中的 run 在 http.h 对应后，会将符号表中对应的符号地址替换成 run。

当然，我们编译大型项目时往往借助 makefile，makefile中指定你想要编译的全部 xx.cc 文件，linker会将这些 xxx.o 连接。

这和 rust 有什么相似点呢？

mod xxx; 向编译器声明这个模块的存在，xxx

use xxx; 编译器会查找这个模块，这样不需要我们和 cpp 一样指定全部要编译的文件。

## 函数返回

语句结束要加分号，如果没有分号表示 return

```rust
fn multiple(first_number_str: &str, second_numer_str: &str) -> Result<i32, ParseIntError> {
    // 由于 parse 可能发生异常，这里返回Result，OK可以将如果成功的值解析出来
    let first_number = first_number_str.parse::<i32>()?;
    let second_number = second_numer_str.parse::<i32>()?;
    // 这里没分号，返回了
    Ok(first_number * second_number)
}
```

如果

## Traits 约束 - 接口

就是接口，比如下面代码， println? 默认并不支持输出 struct，就像 Java 重写 toString 后才支持输出 class， rust 中需要实现 std::fmt::display 这个 trait（接口）。

derive(Debug) 实现 Debug 接口，而 Debug 这个 trait 实现了 std::fmt::Display 的 fmt 接口

{:?} 是以 Debug 输出，需要配合 derive(Debug)

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let rect1 = Rectangle {
        width: 30,
        height: 50,
    };
    println!("rect1 is {:?}", rect1);
}
```

下面的 trait 的声明和定义，你可以看到 trait 和 generic 很像

```rust
trait Add<Rhs=Self> {
    // 类似 typescript 的 type x = () => void; 语法
    // 定义一个Output便于函数使用
    // 由于 Trait 不能被直接实例化所以 Output 并没有绑定到具体的类型
    type Output;

    fn add(self, rhs: Rhs) -> Self::Output;
}
use std::ops::Add;

struct Millimeters(u32);
struct Meters(u32);

impl Add<Meters> for Millimeters {
    // 我们实现 Add 这个trait时将 Output 定为 Millimeter 类型
    type Output = Millimeters;

    fn add(self, other: Meters) -> Millimeters {
        Millimeters(self.0 + (other.0 * 1000))
    }
}
```

### Trait 和 范型区别

这是 Trait

```rust
struct Counter {
    count: u32,
}

impl Counter {
    fn new() -> Counter {
        Counter { count: 0 }
    }
}

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        // --snip--
        if self.count < 5 {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}

fn main() {}
```

这是范型，调用范型函数时每次调用都需指明范型参数的具体类型 T。

Trait 可理解为 具体的范型，trait 只能实现一次且实现时已经指定为哪个类型实现。
当我们调用 trait 方法时不再需要显式传递范型参数

https://doc.rust-lang.org/book/ch19-03-advanced-traits.html#specifying-placeholder-types-in-trait-definitions-with-associated-types

```rust
pub trait Iterator<T> {
    fn next(&mut self) -> Option<T>;
}
```

## 函数签名推断

有人会有疑问，为什么函数上要加类型约束，难道不能从调用方传递过来的类型推断吗

事实上函数可能会单独存放，它并不知道谁会调用它，函数需要自身携带有关的参数信息便于编译器进行调用参数校验。

## where keyword

```rust
pub fn deregister<S>(&self, source: &mut S) -> io::Result<()>
where
S: event::Source + ?Sized,
{
trace!("deregistering event source from poller");
source.deregister(self)
}
```

前面说了 trait 实现接口，当涉及到泛型时还会有泛型参数约束

比如 Java 中 `<T extends String>` 要求 T 必须继承自 String，rust 可以对泛型参数 S 进行约束，event::Source 和 Sized 的合体类型

---

## 移动语义

```rust
let args: Vec<String> = env::args().collect();
// 这是正确的
let query = &args[1];

// 下面代码就会出现 cannot move 的错误，究其原因是 String 类型不是 Copy 类型，String 对象内管理着一些内存，这些内存只能转移出去（move out）或者 Clone
// 是不是和 C++ 中的 Copy， Move语义很相似？
// https://stackoverflow.com/a/40075101/7529562
let query = args[1];
```

---

再看一个

```rust
let contents = fs::read_to_string(config.filename).expect("Something went wrong reading the file");
println!("{} contends are {}",config.filename, contents);
```

上面代码运行会出这个错误：

```
error[E0382]: borrow of moved value: `config.filename`
  --> src\main.rs:10:35
   |
8  |     let contents = fs::read_to_string(config.filename)
   |                                       --------------- value moved here
9  |         .expect("Something went wrong reading the file");
10 |     println!("{} contends are {}",config.filename, contents);
   |                                   ^^^^^^^^^^^^^^^ value borrowed here after move
   |
   = note: move occurs because `config.filename` has type `std::string::String`, which does not implement the `
Copy` trait
```

原因是 config.filename 所有(ownership) 转移到 read_to_string，println! 时没有所有权而报错

改为 fs::read_to_string(&config.filename)，以借用 borrow 方式解决

### 函数闭包

```rust
let expensive_closure = |num| {
    println!("calculating slowly...");
    thread::sleep(Duration::from_secs(2));
    num
};
```

没错，上面这个就是 函数闭包，我知道你以为的闭包是这样子

```js
let expensive_closure = function(num) {
    // xxx
}
```

rust 在文档里也解释为什么用 |num| 的形式

> this syntax was chosen because of its similarity to closure definitions in Smalltalk and Ruby.

说白了，就是从 Ruby 借鉴(抄)过来的嘛！虽然我觉着这语法挺难看的 :)

下面比较几个 rust 语法定义

```rust
// 函数定义
fn  add_one_v1   (x: u32) -> u32 { x + 1 }

// 添加上 类型约束 的闭包函数定义
let add_one_v2 = |x: u32| -> u32 { x + 1 };

// 去掉 类型约束 的闭包函数定义
let add_one_v3 = |x|             { x + 1 };

// 由于函数体只有一个表达式，所以 {} 也直接去掉
// 和 Javascript 的 let add_one_v4 = x => x + 1 很像
let add_one_v4 = |x|               x + 1  ;
```

### 静态生存期 （static Lifetime）

```rust
let s: &'static str = "I have a static lifetime.";
```

## 所有权

GC 语言不需要手动清理引用，cpp 没有 GC 需要你显式调用 free，但 free 一次分配只能调用一次，多次调用会出现不可预料的问题。

GC 语言的问题是当引用类型作为参数传递给函数时在函数的 scope 中会有一个新的变量来承接传递过来的引用，这时同一块内存就有了两个变量引用。问题出现了，我们怎么知道什么时候才能释放这块内存呢？

GC 语言不需要程序员关注，因为 GC 会帮你标记引用次数并使用算法感知到什么时候能释放。

Rust 如果没有所有权的概念那就和 CPP 一样需要程序员手动释放内存。

所有权就是引用类型在传递给函数或者其他的变量时上一个变量就不允许再访问原来的内存，这时内存的所有权给了新变量。rust 通过这种方式可以确保某一时刻始终只有一个变量 hold 这块内存，当这个变量离开属于它的 scope 时就可以调 drop。

**核心点是只有一个变量持有引用**，如果多个变量都持有，那在分别离开自己 scope 后多次调用 free释放同一块内存会导致 `memory corruption`

但多个变量都持有同一块内存的引用是非常有用的，Rust也提供了Rc，Arc来实现 counted references

Rust 中的基本数据类型Copy 速度会很快，所以允许旧变量赋值之后本身可用

```rs
// x 给y之后x还可用
let x = 5;
let y = x;

println!("x = {}, y = {}", x, y);
```

其他复杂类型赋值= 时会引用所有权系统，但自实现的 Copy Trait 除外（Rust不允许同时实现Copy和Drop方法）
https://kaisery.github.io/trpl-zh-cn/ch04-01-what-is-ownership.html#%E5%8F%AA%E5%9C%A8%E6%A0%88%E4%B8%8A%E7%9A%84%E6%95%B0%E6%8D%AE%E6%8B%B7%E8%B4%9D

看 rust 手册时还看到了 move，copy 的概念，这个就和 cpp 差不多了，本质是一种接口的设计，程序员可以实现这些接口来规定变量赋值时到底采取什么行为，cpp 中有几个概念：移动构造函数，拷贝构造函数

以及右值引用

如果你在调用函数结束需要继续使用 s1 ，那函数必须以引用 & 的方式，函数结束后所有权会还给 caller。 假如你有一个苹果，你需要别人借用之后返回给你，你可以用一个盒子把你的苹果包装起来，将这个盒子的所有权借给他（function），等借用完成盒子被销毁但你的苹果还在，其实引用（&）就是一个拦截器，可以拦截 function 对于内部变量的操作。

```rust
let s1 = String::from("hello");
let len = calculate_length(&s1);

fn calculate_length(s: &String) -> usize { // s is a reference to a String
    s.len()
} // H
```

```rust
let s1 = String::from("hello");
let len = calculate_length(s1);

// calculate_length 的 s 参数没有 &，那 s1 传递给 calculate_length 时值会被move
// 原理是 calculate_length 中创建一个临时变量来承接 s1 的内容那 s1 就不能引用自己原来的内容了
fn calculate_length(s: String) -> usize { // s is a reference to a String
    s.len()
} // H
```

看起来 rust 隐藏了 GC 的概念，但隐藏的代价是开发者不能想使用 GC 语言一样随意的返回引用、对变量进行操作。其实习惯也没什么感知了。

rust 中的 & 叫做 reference，你就可以认为是 cpp 中的 pointer。

rust 中解引用用 * 获取pointer值向的值

& 在rust中还有一个含义，借用(borrow)，上面说了，所有权关键点最终某一个时刻只应有一个量持有引用的内存

但其实 Rust 还允许同时存在多个不可变引用，或者只存在一个可变引用

```rust
let a = Box::new(1);
let b = &a;
let c = &a;
let d = &a;
let mut e = &a;
```

#### 如何形象描述 Rust 引用呢？

![Rust引用结构](https://user-images.githubusercontent.com/24750337/97800016-834f7380-1c6c-11eb-891e-ed3054b11a38.png)

上图说明 s 只是简单存放了指向 s1 的 pointer，s 对最右侧操作需要二次寻址。二次寻址确保了当 s 不再使用可以安全 drop 掉而不影响 s1

理论上能获取到pointer就获得了对数据修改的能力， 但 Rust 中引用默认是 immutable，想要修改需添加 mut 修饰。

mut 有个限制，在同一个 scope 只允许一个变量持有 mut 的引用

https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html#mutable-references

下面代码编译不通过。

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &mut s;
    let r2 = &mut s;

    println!("{}, {}", r1, r2);
}
```

```
$ cargo run
   Compiling ownership v0.1.0 (file:///projects/ownership)
error[E0499]: cannot borrow `s` as mutable more than once at a time
 --> src/main.rs:5:14
  |
4 |     let r1 = &mut s;
  |              ------ first mutable borrow occurs here
5 |     let r2 = &mut s;
  |              ^^^^^^ second mutable borrow occurs here
6 | 
7 |     println!("{}, {}", r1, r2);
  |                        -- first borrow later used here
```

引用总结

> 1. 在某一时刻，你只被允许只有 一个可变引用（mut） 或多个不可变引用
> 2. 引用必须总是合法，即：引用变量必须存活到被引用变量之后

---

下面代码编译不通过在于：对于引用类型的赋值不像其他语言的shadow copy一般，rust 会直接将 s1 持有的全部数据转移到 s2 中，从s2往后s1就不被设置为非法，不被允许访问原来的内存，这就是 Move。

```rust
let s1 = String::from("hello");
let s2 = s1;

println!("{}, world!", s1);
```

借用不会move 掉 s1 的内存，而暂时允许 s2 持有 s1 引用的内存。通过 & 的标记 Rust 会确保 s2 离开自己的scope时不会被调用 drop，因为 s2 的内存最终属于 s1

```rust
let s1 = String::from("hello");
let s2 = &s1;

println!("{}, world!", s1);
```

---

基本数据类型大小在编译时就可确认，stack中的数据复制性能远高于存放于 heap 的String类型，这时数据的赋值就是Copy

x 和 y 相等但不相同

```rust
let x = 5;
let y = x;

println!("x = {}, y = {}", x, y);
```

我们再想一下，为什么会有所有权的概念？GC存在的意义是正确释放不再使用的 heap 内存。

函数调用需要调用栈 stack 的大小必须可被计算，所以我们不能将大小可变的数据存到栈

Rust有四种基本类型(Integer, Floating, Boolean, Character)，两种组合类型(Tuple, Array)

Tuple，Array长度固定，也即编译时就必须可确认大小，**上面六种类型都会分配到 stack**

所有权关注的是分配到 heap 的数据类型，

看下面的代码，既然数组是基本类型，那函数调用时就会复制整个数组，你对 array_b 的修改不会反映到 array_a

这里不存在所有权转移，array_b 本质是复制了 array_a 而不是**偷走**了 array_a 的内存

为什么不偷走呢？因为 Array 类型 **size fixed**，编译时就可以分配好栈大小

```rust
#[test]
fn p () {
    let mut array_a = [1,2,3];
    c(array_a);
    fn c (mut array_b: [i32;3]) {
        array_b[1] = 2;
    }
    // [1, 2, 3]
    println!("{:?}",array_a)
}
```

想要在 p 里修改 array_a 就需要传递指针 & 并标记为可变 mut

```rust
#[test]
fn p () {
    let mut array_a = [1,2,3];
    c(&mut array_a);
    fn c (mut array_b: &mut [i32; 3]) {
        array_b[1] = 6;
    }
    // [1, 6, 3]
    println!("{:?}",array_a)
}
```

下面 s 是个 &str 类型，因为 HelloWorld 被存到 .data 段， s 是指向 .data 段某个地址的不可变引用

```rs
let s = "Hello, world!";
```

考虑 所有权 的关键点在于明确数据分配在 heap 中，这里为什么说是数据而不说变量呢？因为数据囊括可变和不可变，这两种都可以分配到 heap

具体可以读读下面链接，我记录的往往都是官方文档给的启发。

https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html

#### move

下面闭包孵化了新的线程，但线程运行的时间可能长于调用函数。

需要明确闭包函数对外部变量的所有权归属，改为 `let thread = thread::spawn(move || loop {`

后闭包会强行 take ownership

```rust
fn new (id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
    let thread = thread::spawn(|| loop {
        let job = receiver.lock().unwrap().recv().unwrap();
        println!("Worker {} got a job; executing.", id);
        job();
    });
    Worker{
        id, thread
    }
}
```

### 生命周期

2020/11/19 的思考

Rust始终在解决一个问题：如何保证内存安全？

所以对于包含 引用 的 `struct` 结构体，Rust 必须确保引用在  `struct` 实例存活过程中始终有效，不会出现野指针
在编译器无法确定引用和 struct 自身存活时间孰短孰长时，开发者必须通过生命周期标识 显式 保证 引用 存活时长。

```rs
struct User {
    username: &str,
    email: &str,
    sign_in_count: u64,
    active: bool,
}

fn main() {
    let user1 = User {
        email: "someone@example.com",
        username: "someusername123",
        active: true,
        sign_in_count: 1,
    };
}
```

上面代码编译器会抛出如下问题

```
error[E0106]: missing lifetime specifier
 -->
  |
2 |     username: &str,
  |               ^ expected lifetime parameter

error[E0106]: missing lifetime specifier
 -->
  |
3 |     email: &str,
  |            ^ expected lifetime parameter
```

看完了文档总结我的理解：

生命周期告诉 Rust 两个变量存活的时间长度

比如下面的代码，返回的 &'a str 一定是 x **或** y，或者说，&'a str 一定依赖了 x **或** y，

那通过生命周期标注 `'a` 告诉 Rust 编译器返回的值生命周期是 x **和** y **最小值**

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

不能在函数中返回局部变量的引用，函数调用完成 result 就被drop掉，所以返回的引用也是不合法的

这段代码过不了编译是因为 str 属于基本数据类型，调用结束函数栈就被pop掉，对于被pop掉的栈变量引用是不合法的。

```rust
fn longest<'a>(x: &str, y: &str) -> &'a str {
    let result = String::from("really long string");
    result.as_str()
}
```

当然，你可以返回 String

```rust
fn main() {
    let a = calculate_length();
    println!("{}",a)
}
fn calculate_length() -> String { // s is a reference to a String
    String::from("123")
} // H
```

#### 'static 生命周期

简单介绍一个语句

```rust
// 定义 Job 类型， dyn FnOnce 是多态函数，只允许被调用一次，实现了 Send trait，'static生命周期
// Box 要求编译器将 Job 类型放到 heap 中
type Job = Box<dyn FnOnce() + Send + 'static>;
```

#### 生命周期参数

你调用函数时编译器只能知道函数的参数类型，但并不知道传递给函数的变量要存活多久，如果函数永远不返回那传递的参数就没法释放，所以需要函数标注生命周期来确定传递的参数可以和xxx存活时间一样长。

https://www.reddit.com/r/rust/comments/bltnfv/simplest_best_explanation_of_lifetimes/emth1hk?utm_source=share&utm_medium=web2x&context=3
https://www.reddit.com/r/rust/comments/bltnfv/simplest_best_explanation_of_lifetimes/emrev4p?utm_source=share&utm_medium=web2x&context=3

### Async， Await 语义

再说 Rust 之前先用我最熟悉的 Javascript 举例子

```js
async function readFileFromDisk(path) {
    return await fs.readFile(path);
}
const pwd = await readFromDisk("/etc/passwd");

// 上面的 async function 会被翻译成这样
function readFileFromDisk(path) {
    return fs.readFile(path);
}
readFromDisk("/etc/passwd").then((d) => {
    const pwd = d;
});
```

JS 的例子是将异步 IO 转换成 Promise，这个 Promise 对应的 Rust 中的 Future，Java 中的 CompletableFuture（Java 中的 Future 是个同步调用,override 掉 get 方法）

理解这样概念时要明白怎么实现的，OS 只认识线程

如果 FD（File descriptor） 被设置为 block，那 Read 时没有数据就会被阻塞，既然阻塞当前调用线程就会被挂起，很好理解，所谓的协程， green thread 只不过是用户空间构造出来的概念，OS 只认线程，所以 IO 调用被阻塞线程就作为人质被挂起

如果 FD 是 setBlocking(false)，那就引出现在主流的异步实现方案

这个方案借助了 OS 提供的 IO 等待阻塞原语 IO-aware system blocking primitive，当 Read 没有数据读时会直接返回，可以将 FD 注册给 OS 然后 poll 等待，等数据读时 OS 会唤醒你的调用线程

poll 有两个实现，单线程/多线程池

Rust 中的 mio 是一个单线程实现，Read，Write 都在一个线程

Tokio 借助 mio 实现了线程池方案

问题是常用的 select 方案对 File 支持不友好

https://www.reddit.com/r/rust/comments/dh8ook/mio_for_file_operations/

现有的解决办法是开新的线程阻塞读取

两种阻塞方案
socket 的 read 不一定会有数据，epoll 在这种情况下可以封装事件循环

但如果你读取本地文件 read 一定会马上读取，如果对 file 使用 epoll，当调用 epoll_await 时马上会返回，因为文件这时已经就绪可读了

read 一定要放到某个 thread 才行，Linux 有的 AIO 了解一下

### 智能指针

#### Rc - Reference Counted

具有引用计数功能的指针，只能用于单线程

A，B，C，D 同时引用一块内存，如果知道 A 最长那可以将A设置为 ownership。但实际上不能确认谁的生命周期最长。Rc就是为了解决这个问题，它会在 count = 0 时释放内存。

### 无惧并发

下面代码线程捕获了 v，但 Rust 并不能确定线程运行多长时间，也就不能明确释放时机

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];

    let handle = thread::spawn(|| {
        println!("Here's a vector: {:?}", v);
    });

    drop(v); // oh no!

    handle.join().unwrap();
}
```

所以需要move语义，将线程引用的外层变量全部 take ownership，转移到线程上下文中。
https://doc.rust-lang.org/book/ch16-01-threads.html

#### Channel

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
    });

    let received = rx.recv().unwrap();
    println!("Got: {}", received);
}
```

### 语法杂项

#### 函数调用返回

语句结束要加分号，如果没有分号表示 return

```rust
fn multiple(first_number_str: &str, second_numer_str: &str) -> Result<i32, ParseIntError> {
    // 由于 parse 可能发生异常，这里返回Result，OK可以将如果成功的值解析出来
    let first_number = first_number_str.parse::<i32>()?;
    let second_number = second_numer_str.parse::<i32>()?;
    // 这里没分号，返回了
    Ok(first_number * second_number)
}
```

main 函数没有返回值，所以最后一行要添加分号

#### 闭包(closure) 和 block

看下面的代码， async 块并不是个 闭包 函数，而只是简单的 语法块，首先执行 async {}，然后返回的参数再传递给 `spawn`，那 spawn 拿到的就是个 Future 而不是 函数

```rust
mini_tokio.spawn(async {
    let when = Instant::now() + Duration::from_millis(10);
    let future = Delay { when };

    let out = future.await;
    assert_eq!(out, "done");
});
```

#### Marco 宏 - 元编程

Marco 可以理解为模式匹配，又或者为 正则表达式捕获组，匹配出特定模式的 token 来动态生成代码

test 宏支持重载，有两个写法。 expr 要求 left, right 必须为表达式(expression)

=> () 中的内容是 marco 展开的 block

```rs
macro_rules! test {
    ($left: expr; and $right: expr) => (
        println!("{:?} and {:?} is {:?}",stringify!($left),stringify!($right), $left && $right)
    );
    ($left: expr; or $right: expr) => (
        println!("{:?} or {:?} is {:?}", stringify!($left), stringify!($right), $left || $right);
    )
}

fn main() {
    test!( 1i32 + 1 ==2i32;and 2i32 * 2 == 4i32);
    // 这个也可以运行是由于Rust中 块(block) 也算是表达式
    test!( 1i32 + 1 ==2i32;and {
        let mut a = 1 + 1;
        2i32 * 2 == 4i32
    });
    test!( 1 + 2 == 3;or 2 + 3 == 4);
}
```

rust 中表达式的含义更广泛，https://doc.rust-lang.org/reference/expressions.html

看个更复杂的， ident 是个约束，要求 fn 必须是函数，右侧括号里是个正则， * 允许 0 到无穷多个，所以我在使用的时候添加,,,,,,编译也通过。

这里

```rs
{/**/{}/**/}
```

是个表达式，外层 {} 是宏定义外侧，内层的 {} 是块表达式

因为 Hexo 编译 markdown 会出错，所以用 /**/ 隔断下

```rs
=> {/**/{ //
// 
}/**/}
```

```rs
#[allow(unused_macros)]
macro_rules! syscall {
    ($fn: ident ( $($arg: expr),* $(,)* ) ) => {/**/{ //
        let res = unsafe { libc::$fn($($arg, )*) };
        if res == -1 {
            Err(std::io::Error::last_os_error())
        } else {
            Ok(res)
        }
    }/**/};
}
fn main() {
    let ep = syscall!(epoll_create(19 + 1,,,,,,,,,,,)).map(|ep| ep).unwrap();
}
```

#### struct

没想到一个 struct 我竟然要单独学一些

已经习惯了如下写法

```rust
struct Point {
    name: str,
}
```

结果看 mio 时发现还有这样的 struct，我没想到 Token 之后可以直接括号，一直看不懂什么意思，也不好检索。

最后没办法了只能看下 RFC，结果发现 struct Token(pub usize) 叫做 tuple struct，可以认为这种struct 的字段是 0

RFC在此

https://doc.rust-lang.org/reference/items/structs.html

struct 只是约束了存储空间，比如下面

```rs
// struct 是定义符号，即使写法不一样但还是根据存的类型来分配空间
struct Structure_Bool(bool);
struct Structure_Integer(i32);
assert_eq!(1,std::mem::size_of::<Structure_Bool>());
assert_eq!(4,std::mem::size_of::<Structure_Integer>());
```

这个过程告诉我们：

语言的语法其实没什么技术含量，看不懂这个语法可以看 RFC 学习，关键在于理解语法的内在含义，而不是表征

```rust
#[derive(Copy, Clone, Debug, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct Token(pub usize);

impl From<Token> for usize {
    fn from(val: Token) -> usize {
        val.0
    }
}
```

上面的 struct 可以解释如下

```rs
pub struct Token {
    0: usize,
}
```

#### enum 枚举

大一学过 C 语言的 union，rust中的枚举类型和其差不多， 存储空间有最大的类型决定并保证内存对齐
https://en.wikipedia.org/wiki/Tagged_union

> 枚举和 struct 一样可以定义方法。方法、函数只是一堆的指令，操作的数据才是关键。既然枚举也有数据，那就可以为其定义操作自身数据的方法。
> 只不过这 方法 和 类方法 一样其 self 都绑定到了要操作的内存

```rs
#![allow(unused)]
fn main() {
    enum Message {
        Quit,
        Move { x: i32, y: i32 },
        Write(String),
        ChangeColor(i32, i32, i32),
    }

    impl Message {
        fn call(&self) {
            // 在这里定义方法体
        }
    }

    let m = Message::Write(String::from("hello"));
    m.call();
}
```

#### Option 与 match

Rust调用返回 Option，你想消费 Some 值必须要处理 Some 和None（除非使用unwrap）

### extern crate

声明外部依赖，这个依赖将会在编译时指定 soname，然后 linker 会确保运行时这个 so 被加载到进程。

### pub 声明模块可见性

#### pub (crate)

声明只在当前 crate 内可见，意思是只在当前库可见（有cargo.toml的文件夹构成了一个crate）

### QA

#### String 和 str 区别

String是个对象，存放在stack上的数据大小固定

str 是常量，但编译时不能判断具体的长度，所以 str 的内容存放在 静态存储区，通过 &str 获取数据

#### #![no_std] 宏

Rust 的 std 标准库提供了许多特性，Vec，TCP Socket 假设代码运行在OS上，并且存在内存分配器 memory allocator 或者 network stack。

但如果开发自己的OS，这时 memory allocator是不存在的，那 std 库就不能使用，所以 no_std 告诉 Rust 编译器不要将标准库代码link进来。

https://www.reddit.com/r/rust/comments/9eyc21/noob_what_exactly_is_no_std_and_why_is_it_so/

经常和 std 比较的是 core 库

这是 Rust 中最纯净的，没有依赖的库，core不和任何平台绑定，只实现平台无关的逻辑

列举 core 中的两个函数就明白了

```rs
use core::cmp::min; // 计算最小值
use core::mem::size_of; // 计算类型占用的字节数 
```

#### FnOnce()

一次性函数，只能调用一次。往往是匿名函数（闭包函数）

#### Rust 为什么只允许一个变量持有 mut 变量？

mut变量可以被修改也隐含着变量不可控，多个变量持有同一个 mut 引用在多线程情况下及其容易出现 race condition。

rust 推荐如下方式处理多线程变量共享

1. 同一个变量多线程通过Mutex共享
2. channel
3. Arc

#### 到底什么是移动语义

我以为的移动语义

![image](https://user-images.githubusercontent.com/24750337/99526121-cd0ebc80-29d5-11eb-93af-489db17ccae4.png)

但实际**可能**的移动语义

![image](https://user-images.githubusercontent.com/24750337/99526133-d26c0700-29d5-11eb-9d43-782748cf08f4.png)

移动指的是物理内存上的地址没有变化，堆内数据 还在那里

而进程中虚拟地址却变化了，移动之后原来的pointer（虚线）指向不存在的内存空间

下面的Foo在函数中会分配到 stack，如果我们将实例化之后的 Foo 移动给别的函数会调用 memcopy 复制 Foo

问题是 Foo中含有 ptr 指针引用到原来的内存，如果prt指向 caller 的 stack 变量极其容易造成悬空指针，所以需要一种方式固定住这块内存

```rs
struct Foo {
    array: [Bar; 10],
    ptr : &'array Bar,
}
```

### 标准库解读

#### Arc

> A thread-safe reference-counting pointer. 'Arc' stands for 'Atomically Reference Counted'.

```rust
let (sender, receiver) = mpsc::channel();
let receiver = Arc::new(Mutex::new(receiver));
```

想再说下 [WeakMap](https://stackoverflow.com/questions/29413222/what-are-the-actual-uses-of-es6-weakmap)，拿 Javascript举例。

可以理解为 WeakMap 里存放的是弱引用

是个引用：可以找到你存放的Object
又不完全是个引用，GC不会在乎 WeakMap 里的引用，如果没其他对象引用即使WeakMap存在引用也会被GC，相较于Map，WeakMap可以一定程度防止内存泄漏

```js
var map = new Map();
function useObj(obj) {
    doSomethingWith(obj);
    var called = map.get(obj) || 0;
    called ++;
    if (called > 10) report();
    // map 持有全部obj的引用永不释放
    map.set(obj, called);
}
```

如果没有其他引用持有 obj们，那 GC 会自动清除 obj 且 map 中对原obj的引用也会被删除

```js
var map = new WeakMap();
function useObj(obj) {
    doSomethingWith(obj);
    var called = map.get(obj) || 0;
    called ++;
    if (called > 10) report();
    map.set(obj, called);
}
```

那 WeakMap 有啥引用场景呢？

最简单的是持有 DOM，一般而言 DOM 从DOM树上删除后就没用了，如果使用Map会造成 memory leak，这里使用 WeakMap 合适
