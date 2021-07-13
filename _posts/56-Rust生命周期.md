---
title: Rust生命周期
date: 2021-07-01
updated: 2021-07-03
issueid: 56
tags:
- Rust
---
# 生命周期的抽象

**将LT想象成scope不太容易理解，可以将其想象成链。标注同一个的引用必须共存亡。通过 'a, 多个引用链在一起。**
![将引用比作绳子]](https://user-images.githubusercontent.com/24750337/114650520-fb8a8c80-9d14-11eb-93a0-3ee191ff4938.png)

https://doc.rust-lang.org/stable/book/ch10-03-lifetime-syntax.html
https://www.zhihu.com/question/30861652/answer/132841992

# 为什么生命周期被如此设计

跨函数的变量生存期分析及其复杂，要分析完成各种条件语句，且需要的值只有在运行时才能确认，这就加大了编译器分析的复杂度（又可认为不可能进行分析），Rust通过在函数，结构体上进行生命周期标注，将分析的范围限定到函数内部，从而完成整个分析的过程。这就是为何生命之后需要在 函数， 结构体 上进行 `'a` 标注

[官方文档上有过类似的解释](https://doc.rust-lang.org/stable/book/ch10-03-lifetime-syntax.html)
> lifetime syntax is about connecting the lifetimes of various parameters and return values of functions. Once they’re connected, Rust has enough information to allow memory-safe operations and disallow operations that would create dangling pointers or otherwise violate memory safety.

## 生命周期和临时借用

### 复现

```rs
struct NumRef<'a> (&'a i32);

impl<'a> NumRef<'a> {
    fn some_method(&'a mut self) {

    }
}

fn main() {
    let mut num_ref = NumRef(&5);
    num_ref.some_method(); // #1
    num_ref.some_method(); // #2
    println!("{:?}", num_ref);
}
```

上面将 some_method 的 self 和整个 NumRef的 'a 关联，这意味着 some_method只能被调用一次。
rustc调用 #1 时借用了 self，由于 `&'a mut self`，所以 #1 调用完后也不能释放掉这次借用，第二次调用 some_method 出现第二次借用。

### 纠正

```rs
struct NumRef<'a> (&'a i32);

impl<'a> NumRef<'a> {
    fn some_method(&mut self) {}

    // 上面省略了LT，desugar 后是
    fn some_method_desugared<'b>(&'b mut self){}
}

fn main() {
    let mut num_ref = NumRef(&5);
    num_ref.some_method();
    num_ref.some_method();
    println!("{:?}", num_ref);
}
```
'b 和整个 struct 的 'a 无关，所以some_method的可变借用不需要和整个struct对齐，第二次调用可重新借用。

通过上面例子说明，代码能编译通过只是确保内存安全，生命周期的标注不一定是对的。

以上来自

> https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md#5-if-it-compiles-then-my-lifetime-annotations-are-correct

## 生命周期省略 '_

https://doc.rust-lang.org/stable/book/ch10-03-lifetime-syntax.html#lifetime-elision

> The first rule is that each parameter that is a reference gets its own lifetime parameter. In other words, a function with one parameter gets one lifetime parameter: fn foo<'a>(x: &'a i32); a function with two parameters gets two separate lifetime parameters: fn foo<'a, 'b>(x: &'a i32, y: &'b i32); and so on.
>The second rule is if there is exactly one input lifetime parameter, that lifetime is assigned to all output lifetime parameters: fn foo<'a>(x: &'a i32) -> &'a i32.
>The third rule is if there are multiple input lifetime parameters, but one of them is &self or &mut self because this is a method, the lifetime of self is assigned to all output lifetime parameters. This third rule makes methods much nicer to read and write because fewer symbols are necessary.

### 生命周期何时可以省略呢？
**首先明确一点，Rust中每个引用都必须有生命周期，如果没写且编译通过，那属于生命周期省略**

生命周期下面简称 LT
1. 先将每个函数参数都标注唯一的 LT, 'a, 'b, 'c, 'd
2. 如果入参只有一个LT，显而易见， 出参引用的生命周期一定会个 入参的 LT一致，也即，一个 入参引用 和 全部的出参 LT一样 
3. 如果入参多个LT，且是成员函数（&self 或 &mut self），那全部的出参LT都和(&self 或 &mut self)一致。

上述三个规则结束后，如果还有 出参 的LT没被确定，就必须**显式**添加LT标注

如果只有入参有引用，返回类型没引用，不需要标注生命周期。为什么呢？因为LT存在是防止空指针引用，如果返回值没引用函数内部的内存，就不会出现这个问题（单线程下）。

这个是官方的例子
https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=5c4baf83eac0666c47ef3b7b2ec027d9

下面的是我改的
https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=3ae5b85c9f35375f866ad3ddfa3bee47


```rs
struct Spawner{

}

impl fmt::Debug for Spawner {
    // fmt::Formatter 结构体又生命周期标注
    // struct Formatter<'a>
    // '_ 意思是我传递一个生命周期，这个周期以你Formatter<'a>为准
    fn fmt(&self, fmt: &mut fmt::Formatter<'_>) -> fmt::Result {
        fmt.debug_struct("Spawner").finish()
    }
}
```

## 'static 生命周期
当 struct 引用其他内存时需要生命周期标识

'static 告诉编译器，这个 name 生命周期比 Person 结构体 无限长。

https://www.reddit.com/r/rust/comments/o9w6rl/rust_traits_and_static/h3dszio?utm_source=share&utm_medium=web2x&context=3

生命周期只是个标注，编译器根据标注进行生命周期推导，如果发现对不上还是会报错。

```rs
struct Person {
    name: &'static str,
}

impl Person {
    fn new(name: &'static str) -> Person {
        Person { name: name }
    }
}
fn main() {
    let nobody: Person = Person::new("nobody");
}
```

曾以为生命周期随便写也没关系。当你创建或者返回具有生命周期标识的struct时也必须在创建、赋值、返回时指明声明长度让编译器推导

比如下面
```rs
pub(crate) struct BasicScheduler<P: Park> {
    /// Inner state guarded by a mutex that is shared
    /// between all `block_on` calls.
    inner: Mutex<Option<Inner<P>>>,

    /// Notifier for waking up other threads to steal the
    /// parker.
    notify: Notify,

    /// Sendable task spawner
    spawner: Spawner,
}
struct InnerGuard<'a, P: Park> {
    inner: Option<Inner<P>>,
    basic_scheduler: &'a BasicScheduler<P>,
}
impl<P: Park> BasicScheduler<P> {
    // InnerGuard struct有周期标注，所以你返回时也必须进行标注
    // 这里还特殊的一点是，P 是个泛型，实现 Park 这个接口（trait）
    // <> 里同时带了 生命周期标注 和 泛型参数
    fn take_inner(&self) -> Option<InnerGuard<'_,P>> {
        Some(InnerGuard {
            inner: Some(inner),
            basic_scheduler: &self,
        })
    }
}
```

生命周期只是个注解Annotation，告诉编译器你的引用存活到什么时间，但如果编译器推导出的生命长度与你的标注不同，编译器并不会帮你延长变量的存活期，而是会complain，编译出错让你再检查
https://www.zhihu.com/question/30861652/answer/132841992

## https://doc.rust-lang.org/stable/book/ch10-03-lifetime-syntax.html#lifetime-annotations-in-function-signatures

当在函数上标注生命周期时，总是在函数签名，而不是函数体标注。在函数内部的变量编译器可以很轻松的分析，但对于引用的外部的变量，或者从函数参数传进来的变量，编译器无法自己分析，需要手动标注生命周期。

> When annotating lifetimes in functions, the annotations go in the function signature, not in the function body. Rust can analyze the code within the function without any help. However, when a function has references to or from code outside that function, it becomes almost impossible for Rust to figure out the lifetimes of the parameters or return values on its own. The lifetimes might be different each time the function is called. This is why we need to annotate the lifetimes manually.


你如何标注生命周期取决于你的函数在做什么。如果我们将 longest 函数改为总是返回第一个入参，那我们不需要在y上标注。

比如

```rs
// 返回参数只和 x 有关
fn longest<'a> (x: &'a str, y: &str) -> &'a str {
    x
}
```
在上面例子中，只为 x 和返回值标注生命周期参数，这是因为 y 和 x，返回值的生命周期没有任何关系

## 下面这段话很重要

当从函数中返回引用时，返回值的生命周期参数**必须匹配**函数入参的其中之一。如果返回的引用和任何一个入参都没有关系，那返回值一定引用了在函数内创建的变量，这非常危险，因为离开函数作用域之后被引用的值会被销毁

> When returning a reference from a function, the lifetime parameter for the return type needs to match the lifetime parameter for one of the parameters. If the reference returned does not refer to one of the parameters, it must refer to a value created within this function, which would be a dangling reference because the value will go out of scope at the end of the function. Consider this attempted implementation of the longest function that won’t compile:

编写函数，结构体时，最简单的生命周期标注是给全部的引用都加上相同生命周期，这样最起码可以确保万无一失

如果函数某个引用&c并不和其他引用有'a的关系，可以给&c添加 'b，不过这种情况很少出现