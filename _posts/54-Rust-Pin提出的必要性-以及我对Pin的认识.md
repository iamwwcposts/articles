---
title: Rust-Pin提出的必要性-以及我对Pin的认识
date: 2021-06-08
updated: 2021-07-13
issueid: 54
tags:
- Rust
---
## 我对Pin 的整体理解 - 为了解决unsafe场景下move问题

### 提出的必要性 - 不依赖Pin能否做到希望的 不被move ?

Pin的作用是防止move，但如果程序员小心处理，那就不会出错。为什么还需要Pin呢？

既然在某些场景下move是错的，那作为安全的编程语言，就需要显式限制这些场景。**不能将安全交给人的直觉**保证，**这是Rust编译器的责任，不是人的**。

反面例子是C++，程序员犯错的时候还少吗？

明确一点，即使有Pin，你如果想出 UB，通过 unsafe **一定可以** 做到，我只是想说明，Pin不是银弹，万能药。

即使别人眼中UB是错的，但目的不同，对正确的理解不同。如果想要的就是UB的结果，那这UB对你而言是正确的，但这种超出语言的范畴，Pin也不是为这种场景（人的主观）而生的。

在代码整个链路不使用任何unsafe情况下，哪怕没有Pin，你也不会写出UB的代码。编译器的静态检查会教你做人。

unsafe 作为 rust 的一部分，其存在是有意义的，Pin也主要针对这场景。 （我写的，有点像三段论。。。

Pin的意义在于，当你不想出错时，通过 Pin 藏起 `&mut T`，从语法上一定程度限制 unsafe 的危险性。

为什么限定unsafe，在safe rust中，构造自引用并且用起来都十分困难，UB受到 borrow checker的严格限制。不会由于move特性而导致错误。

一个例子：
> 
> 小明你是库tokio的提供者，小红调用小明的函数，该函数利用 unsafe 返回给小红一个结构 X。X的字段 Y 自引用了 &X
> 
> （利用unsafe构造出自引用结构。当然，只是用自引用结构举例子，Pin并不只是用于自引用，比如 tokio-executor 获取 X(task) 的地址，用于 poll 之后的 wake，Pin只是表达不能move）。
> 
> 小红之后使用X，进行了它认为正确的move（move在rust中很常见，而且用的是safe的函数，在小红眼里是正确的）。但这时 Y 错误引用到原来的 &X。
> 
> 那怎么解决呢？
> 
> 小明可以不返回X，而是返回 `Pin<X>`
> 

所以Pin是一种规范，一种交流的**协议**。

如果全代码链路都可小心翼翼处理move，不出现因为move导致的UB，那 Pin 就不是必须

还是上面的例子

小红并没有调用任何 unsafe 的代码，是小明内部使用unsafe，导致了UB

Pin是为了增强鲁棒性（指控制系统在**一定**（结构，大小）的参数摄动下，维持其它某些性能的特性。**一定**也从侧面佐证上文说的，Pin 不是银弹，万能药，只是从语法上一定程度限制 unsafe 的危险性）

任何 **非unsafe** 的代码和 `Pin` 一样，都会被编译器进行静态检查


> 就比如 future-rs，为什么需要使用 Pin？
> 
> future 的 API 内部完全可以保证不会 move 导致 UB，但是 Future 需要给广大开发者使用，广大开发者很容易在 poll 里面把你的 Future 自引用结构体 move 掉


----------------------------

关于 `self referential part`

编写自引用结构，并不是一定需要 `Pin`。

例子，你将整个结构都放到 heap 上，用户 move 只是 move 了 pointer，并不会破坏任何东西(doesn't break anything)。

------------------------------------------

下面是我(nuclearwwc)和alice 关于Pin的对话

> nuclearwwc — Today at 11:00 PM
> 
> Hi, Alice!  some streams contain unsafe code that would be incorrect if the stream is moved. So I have a question about this. If  a developer can write code to avoid move very carefully. Does that mean we don't need the Pin? From my understand, as a safety language, Rust should **explicit** to restrict developer's code of conduct instead of rely on them intuition. That't why we still need Pin even though in unsafe scope. right?
> 
> Alice Ryhl — Today at 11:01 PM
> 
> Sure. The Pin type is only necessary if you can't trust the user to not move it.
> And generally when you write unsafe code and expose it in a library, then the library is generally considered incorrect if there is a way to use that library to invoke undefined > behavior without having the user of the library use unsafe code.
> Being robust against any type of non-unsafe code requires compile-time checks such as Pin
> There are some cases where you don't need Pin to correctly write a self-referential struct. This can happen if the self-referential part is on the heap, since then the user moving the > struct doesn't break anything.


https://discord.com/channels/500028886025895936/500336333500448798/847489538414608444


## Pin用于防止变量被误 move

其中之一应用场景是解决async中 Future 自身被move，导致executor唤醒失败的问题。


本小结来自以下文章

> https://fasterthanli.me/articles/pin-and-suffering

最常见的Pin方式是 `Box::pin(x)` 或者 `Pin::new(x)`，但后者要求 `x: Unpin`

对于一个结构体，当每个成员都是 `Unpin` 时，整个 `struct` 就是 `Unpin`

rust中默认类型都是Unpin的，也即，即使被Pin住，可以move

当需要开发自引用结构时，可以将 struct 某个自引用字段用Pin包裹，这样struct就是 !Unpin，想要Pin住这种结构常规用 `Pin::new(x)`不行了。必须 Box::pin(x);

典型的自引用结构是 linked_list

https://github.com/tokio-rs/tokio/blob/6b9bdd5ca25bd3f30589506de911db73f7dbf8b4/tokio/src/time/driver/entry.rs#L299


TimerEntry -> TimerShared -> TimerSharedPadded#pointers -> LinkedList -> TimerShared

由于TimerShared结构间接自引用，所以整个链条都需要通过 PhantomPined 来 !Unpin

---------------------

## Unpin

Unpin是指 **即使 struct 被Pin住，struct 也可以被移动**，并不是指不能被移动！

Types that can be safely moved **after** being pinned.

https://doc.rust-lang.org/std/marker/trait.Unpin.html

x.await要防止上一行存的引用在下一行被move，所以poll future时临时Pin住

Future是编译器翻译成的状态机，编译器会将await点用到的变量保存到struct，默认赋值这些变量时通过 `*Self = xxx`，由于Self永远是自身，所以move是安全的（是通过Self相对引用）。

但如果我们future的代码里获取了绝对地址，并且用在await，编译器会将这地址保存到struct，这时调用poll()被move就会出问题。为了兼容这情况，poll的Self就必须被Pin住。


```rs
async fn pin_example() -> i32 {
    let array = [1, 2, 3];
    // async第一次执行后
    // 获取array里面的绝对地址
    // 这些代码就将 Future 变成了自引用结构
    // Future通过poll时禁止move来避免move后引用绝对地址导致的 UB
    let element = &array[2];
    async_write_file("foo.txt", element.to_string()).await;
    *element
}
```

也为了兼容这情况，Future结构是 !Unpin

https://os.phil-opp.com/async-await/#pinning

Future在第一次poll之前 move 是没问题的，因为async函数里的代码被封装进 Future#poll 里，只要还从未调用过这函数，poll里可能存在的生成自引用的代码就不会被执行，那这时移动就没问题

**It is worth noting that moving futures before the first poll call is fine**. This is a result of the fact that **futures are lazy and do nothing until they're polled for the first time**. The start state of the generated state machines therefore only contains the function arguments, but no internal references

--------------------

这两行 tokio::pin! 如果注释掉会出问题
https://github.com/willdeeper/shadowsocks-rust/blob/6ff4f5b04b8e22d36cd54477cfcadf8cb403de21/bin/ssserver.rs#L308

究其原因，是由于 select 要求 Unpin，而 server 是 Future，对Future本身是 !Unpin（注意，!Unpin是编译器不会为其实现Unpin），由于server和abort_signal会被状态机放到 struct 保存状态，所以 tokio::pin 只是简单检查了是 ownership，就Pin::new_unchecked获取Pin


关于状态机和 !Unpin，可以看我写的 `async是如何使用状态机实现的`（本地文档库 计算机理论/），搜索!Unpin，会有详细介绍。

## 如果不Pin住实现Future的struct会有可能发生Panic

> 左侧顶部三个点的按钮可以开启RUST_BACKTRACE 看到栈回溯 

https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=8fde70c2583dbcb2a4cf3d929a638b8b

```rs
use futures::Future;
use std::{
    mem::swap,
    pin::Pin,
    task::Poll,
    time::Duration
};
use tokio::{
    macros::support::poll_fn,
    time::sleep
};

#[tokio::main]
async fn main() {
    let mut sleep1 = sleep(Duration::from_secs(1));
    let mut sleep2 = sleep(Duration::from_secs(1));

    {
        let mut sleep1 = unsafe { Pin::new_unchecked(&mut sleep1)};
        poll_fn(|cx| {
            let _ = sleep1.as_mut().poll(cx);
            Poll::Ready(())
        }).await;
    }
    swap(&mut sleep1, &mut sleep2);
    sleep1.await;
    sleep2.await;
}
```

上面代码会panic，原因就是没有真正Pin住，被swap出去，导致tokio唤醒了错误的sleep future

```
$ RUST_BACKTRACE=1 cargo run --quiet
thread 'tokio-runtime-worker' panicked at 'assertion failed: cur_state < STATE_MIN_VALUE', /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/time/driver/entry.rs:174:13
stack backtrace:
   0: rust_begin_unwind
             at /rustc/cb75ad5db02783e8b0222fee363c5f63f7e2cf5b/library/std/src/panicking.rs:493:5
   1: core::panicking::panic_fmt
             at /rustc/cb75ad5db02783e8b0222fee363c5f63f7e2cf5b/library/core/src/panicking.rs:92:14
   2: core::panicking::panic
             at /rustc/cb75ad5db02783e8b0222fee363c5f63f7e2cf5b/library/core/src/panicking.rs:50:5
   3: tokio::time::driver::entry::StateCell::mark_pending
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/time/driver/entry.rs:174:13
   4: tokio::time::driver::entry::TimerHandle::mark_pending
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/time/driver/entry.rs:591:15
   5: tokio::time::driver::wheel::Wheel::process_expiration
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/time/driver/wheel/mod.rs:251:28
   6: tokio::time::driver::wheel::Wheel::poll
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/time/driver/wheel/mod.rs:163:21
   7: tokio::time::driver::<impl tokio::time::driver::handle::Handle>::process_at_time
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/time/driver/mod.rs:269:33
   8: tokio::time::driver::<impl tokio::time::driver::handle::Handle>::process
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/time/driver/mod.rs:258:9
   9: tokio::time::driver::Driver<P>::park_internal
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/time/driver/mod.rs:247:9
  10: <tokio::time::driver::Driver<P> as tokio::park::Park>::park
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/time/driver/mod.rs:398:9
  11: <tokio::park::either::Either<A,B> as tokio::park::Park>::park
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/park/either.rs:30:29
  12: <tokio::runtime::driver::Driver as tokio::park::Park>::park
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/runtime/driver.rs:198:9
  13: tokio::runtime::park::Inner::park_driver
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/runtime/park.rs:205:9
  14: tokio::runtime::park::Inner::park
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/runtime/park.rs:137:13
  15: <tokio::runtime::park::Parker as tokio::park::Park>::park
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/runtime/park.rs:93:9
  16: tokio::runtime::thread_pool::worker::Context::park_timeout
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/runtime/thread_pool/worker.rs:422:13
  17: tokio::runtime::thread_pool::worker::Context::park
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/runtime/thread_pool/worker.rs:398:20
  18: tokio::runtime::thread_pool::worker::Context::run
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/runtime/thread_pool/worker.rs:328:24
  19: tokio::runtime::thread_pool::worker::run::{{closure}}
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/runtime/thread_pool/worker.rs:303:17
  20: tokio::macros::scoped_tls::ScopedKey<T>::set
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/macros/scoped_tls.rs:61:9
  21: tokio::runtime::thread_pool::worker::run
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/runtime/thread_pool/worker.rs:300:5
  22: tokio::runtime::thread_pool::worker::Launch::launch::{{closure}}
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/runtime/thread_pool/worker.rs:279:45
  23: <tokio::runtime::blocking::task::BlockingTask<T> as core::future::future::Future>::poll
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/runtime/blocking/task.rs:42:21
  24: tokio::runtime::task::core::CoreStage<T>::poll::{{closure}}
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/runtime/task/core.rs:235:17
  25: tokio::loom::std::unsafe_cell::UnsafeCell<T>::with_mut
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/loom/std/unsafe_cell.rs:14:9
  26: tokio::runtime::task::core::CoreStage<T>::poll
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/runtime/task/core.rs:225:13
  27: tokio::runtime::task::harness::poll_future::{{closure}}
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/runtime/task/harness.rs:422:23
  28: <std::panic::AssertUnwindSafe<F> as core::ops::function::FnOnce<()>>::call_once
             at /home/amos/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/std/src/panic.rs:322:9
  29: std::panicking::try::do_call
             at /home/amos/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/std/src/panicking.rs:379:40
  30: __rust_try
  31: std::panicking::try
             at /home/amos/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/std/src/panicking.rs:343:19
  32: std::panic::catch_unwind
             at /home/amos/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/std/src/panic.rs:396:14
  33: tokio::runtime::task::harness::poll_future
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/runtime/task/harness.rs:409:19
  34: tokio::runtime::task::harness::Harness<T,S>::poll_inner
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/runtime/task/harness.rs:89:9
  35: tokio::runtime::task::harness::Harness<T,S>::poll
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/runtime/task/harness.rs:59:15
  36: tokio::runtime::task::raw::poll
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/runtime/task/raw.rs:104:5
  37: tokio::runtime::task::raw::RawTask::poll
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/runtime/task/raw.rs:66:18
  38: tokio::runtime::task::Notified<S>::run
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/runtime/task/mod.rs:171:9
  39: tokio::runtime::blocking::pool::Inner::run
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/runtime/blocking/pool.rs:278:17
  40: tokio::runtime::blocking::pool::Spawner::spawn_thread::{{closure}}
             at /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.4.0/src/runtime/blocking/pool.rs:258:17
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```


> Well, precisely because that's a **heap allocation**! If you're holding a Sleep on the stack, and you pass it around to another function, or store it in a struct, or whatever, it actually moves — its address changes.
>
> But if you're holding a Box<Sleep>, well, then you're only holding a pointer to a Sleep that lives somewhere in heap-allocated memory. That somewhere will never change, ie. **the Sleep itself will never move**. The pointer to Sleep can be passed around, and everything is fine.

--------------


## Tokio#spawn要求Future必须是Pin

https://github.com/tokio-rs/tokio/blob/6b9bdd5ca25bd3f30589506de911db73f7dbf8b4/tokio/src/runtime/task/core.rs#L221

整个Task都被分配到heap，所以不会move，需要poll future时通过下面

```rs

// Safety: The caller ensures the future is pinned.
let future = unsafe { Pin::new_unchecked(future) };
```

可获得Pin

## Pin on stack

为什么要记录下这个呢？是因为看 shadowsocks-rust时使用的 pin! 里面竟然是 `new_unchecked()`

https://github.com/shadowsocks/shadowsocks-rust/blob/49b7004d7565de98a24b49657443b19c03365a8f/bin/ssserver.rs#L304

所以就好奇，为什么可以直接用 `new_unchecked` 函数。

tokio::pin!用的 new_unchecked 并不在乎到底Pin的是在stack还是heap，如果指针指向stack，就是pin在stack，指向heap，就是pin在heap。

tokio::pin!有意思的点是，`let mut $x = $x` 先shadow掉原来的 $x，确保 $x 不会被别的变量持有。

最后Pin住。Pin 根本就不在乎是 stack 还是 heap，Pin只是代表不能被 move，

> pinning makes sense anywhere
> pinning just means it doesn't move

相当于，你给我 T，最后我返还给你 `Pin<&mut T>`，由于 Pin<&mut T> 限制地更严，所以安全。（从宽松到严格往往是容易的）

tokio::pin好处是，你完全可以 safely 使用 pin。 而不需要你自己的代码写 unsafe


https://discord.com/channels/500028886025895936/500336333500448798/846016916957954058

```rs
#[macro_export]
macro_rules! pin {
    ($($x:ident),*) => { $(
        // Move the value to ensure that it is owned
        // 我们是 ownership
        let mut $x = $x;
        // Shadow the original binding so that it can't be directly accessed
        // ever again.
        // 原来的 x 被我们 shadow 掉了，x 变成 Pin<T>
        #[allow(unused_mut)]
        let mut $x = unsafe {
            $crate::macros::support::Pin::new_unchecked(&mut $x)
        };
    )* };
    ($(
            let $x:ident = $init:expr;
    )*) => {
        $(
            let $x = $init;
            $crate::pin!($x);
        )*
    };
}
```

## Pin projection 和 structural Pinning

大体意思是，对于一个结构体Struct，当Struct被Pin住时，这个Pin的约束对哪个字段起作用？
如果某个字段被move并不会引起问题，那 `Pin<&mut Map>` 可以转换为 `&mut f`
如果某个字段 future 必须被Pinned，那 `Pin<&mut Map>`就是约束他(future)的，而非其他字段。

其他字段是否被move，并不会引起unsound。

```rs
struct Map<Fut, F> {
    future: Fut,
    f: Option<F>
}

// 默认每个字段都必须是Unpin，Map才会是Unpin
// 我们主动为Map实现Unpin
// 当 Fut 是Unpin时，整个结构就是Unpin
// 为什么不在乎f Unpin与否？
// 因为f是否Unpin，和是否被move，并不会导致future#poll出问题，可以忽略f
// 
impl <Fut: Unpin, F> Unpin for Map<Fut, F> {};

impl<Fut, F, T> Future for Map<Fut, F>
where   Fut: Future,
        F: FnOnce(Fut::Output) -> T
{
    unsafe_pinned!(future: Fut);
    unsafe_unpinned!(f: Option<F>);

    type Output = T;
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<T> {
        match self.as_mut().future().poll(cx) {
            Poll::Pending => Poll::Pending,
            Poll::Ready(output) => {
                let f = self.f().take().expect("take must success");
                Poll::Ready(f(output))
            } 
        }
    }
}
```

pin_project crate 就是做这个的，通过在字段上标注 `#[pin]`进行局部Pin，struct是Pin的，有 `#[pin]`的字段也是Pin的，其他字段是否被move没有问题


当字段被Pin住，成为 `Pin<&mut Field>`，就不应该将 Field 移出去。这里要说明的是，移出去不一定错，Pin从设计上说不能move而已。只要将他假设为，虽然被Pin住，但还是可以被move。


## 翻译 https://doc.rust-lang.org/nightly/core/pin/index.html#projections-and-structural-pinning

### Projections and Structural Pinning 映射和结构化Pinning

名词解释

1. projection：映射，投影。提供一个方法将 `Pin<&mut Struct>` 转换成 `Pin<&mut Field>`
2. structural pinning: 结构pinning。如果整个struct是Pin，那每个成员也都是Pin。比如现存在 `Pin<&mut Struct>`类型，则此刻 Struct 里的每个字段都必须是 `Pin<&mut FieldABCDEFG>`


当使用pinned结构时，我们如果获取方法参数中类型是 `Pin<&mut Struct>`的内部结构成员？通常办法是写一个工具函数（叫做 projection），将 `Pin<&mut Struct>`转换成对内部字段的引用。但这引用应该是 `Pin<&mut Field>` 还是 `&mut Field`？实际上当获取Enum，或容器类型(`Vec<T>`, `Box<T>`, `RefCell<T>`)内部字段的引用时，也会有上面的疑问。（这疑问涉及到可变和共享引用两者，我们只是单拎出来可变引用加以说明）。

实际上是否将某个字段的pinned projection转换为 `Pin<&mut Struct>`转换成 `Pin<&mut Field>` 还是 `&mut Field` 取决于 Struct 结构体的提供者。这种转化是有约束的，最重要的约束时一致性。Struct的每个字段两个取其一，只能被映射成为 pinned reference 或者 &mut Field。如果同时映射为上述两者，那可能是 unsound.

作为数据结构的作者，你必须决定每个字段是否能从 `Pin<&mut Struct>`转换为 `Pin<&mut Field>`。pinning的传递（即从 `Pin<&mut Struct>`转换为 `Pin<&mut Field>`）称为结构化(structural)。因为他衍随了原来外层的类型（我自己的理解，`Pin<&mut Struct>`=> `Pin<&mut Field>`，只是内层T变化，作为最外层的 `Pin<T>`不变，也即，始终是Pin的）。在下面的章节，我们分别解释两种选择的考量

> 哪两者？分别是
> 1. `Pin<&mut Struct>` => `&mut Field`，称为 not structural(Pinning is not structural for `field`)
> 2. `Pin<&mut Struct>` => `Pin<&mut Field>`，称为 structural(Pinning is structural for `field`)

#### Pinning is not structural for `field`

一个被Pin住的Struct `Pin<&mut Struct>`，其内部字段竟然不是Pinned，看着很反直觉，但通常这是最简单的选择，如果`Pin<&mut Field>`从未被创建，那绝对不可能出错！所以，如果你认为某些字段没必要有 `结构化pinning约束（structural pinning）`，你需要确保的是，你自始至终都不会创建对这些字段的 pinned reference，也即，不会创建 `Pin<&mut Field>`。

没有结构化pinning的字段可以通过映射方法，将 `Pin<&mut Struct>`转换为 `&mut Field`。

```rs
impl Struct {
    fn pin_get_field(self: Pin<&mut Self>) -> &mut Field {
        // This is okay because `field` is never considered pinned.
        unsafe { &mut self.get_unchecked_mut().field }
    }
}
```

你设置可以 `impl Unpin for Struct` 即使字段 field 不是 Unpin. 

> What that type thinks about pinning is not relevant when no Pin<&mut Field> is ever created.
> 这句话我不会翻译，但我这么理解：一定要区分开约束是编译期（是编译期，不是编译器，没打错）的施加还是不这样做一定会出错
> Pin只是编译期的约束，是为了防止出错，详细介绍看本文上部分 `Pin的作用是防止move，但如果程序员小心处理，那就不会出错。为什么还需要Pin呢？xxxx`。

> 解释，由于 field 被Pin与否都不会影响逻辑，所以其他需要被Pin的都实现了Unpin，那整个结构从原则上就是 Unpin，但编译器只会对全部字段都是Unpin的 Struct才会自动实现Unpin，所以这种情况下需要开发者手动声明 Struct 为 Unpin

#### Pinning is structural for `field`

另一个选择是决定 pinning对字段是结构化的，这意味着，如果struct被pinned，field也会。

通过这可以编写映射创建 `Pin<&mut Field>`，如下所示，field是pinned

```rs
impl Struct {
    fn pin_get_field(self: Pin<&mut Self>) -> Pin<&mut Field> {
        // This is okay because `field` is pinned where `self` is.
        unsafe { self.map_unchecked_mut(|s| &mut s.field) }
    }
}
```

然而，为了结构化pinning，需要满足几个条件：

1. 如果全部结构化field都是Unpin，那struct这容器才会是Unpin。这其实是编译器默认的行为，但Unpin是safe trait，所以编写 struct 的作者需确保不添加 `impl <T> Unpin for Struct<T>`（值得注意的是，像上面 `pin_get_field`添加将`Pin<&mut Self>` 映射为 `Pin<&mut Field>`需要unsafe代码块，但实现Unpin本身是safe的，如果你通过unsafe完成上述映射，就必须仔细考虑Unpin的添加与否。）

> 为什么要特别强调Unpin？
> Unpin会逃脱 Pin 的限制，从而获取 `&mut Field`，将 `Pin<&mut Self>`转换成 `Pin<&mut Field>`的过程中蕴含着一层假设，即Self不会被move。但对Struct实现Unpin后会使得Self 成为Unpin，Unpin可以逃离Pin的限制，这很危险（只是危险，不一定会出错，如果你足够牛逼，仔细处理各种边界case也没问题，只不过没有编译器约束你罢了）


2. struct的析构函数不能将结构化pin的字段move出去。这是[上一节](https://doc.rust-lang.org/nightly/core/pin/index.html#drop-implementation)着重陈述的一点： `drop` 接受 `&mut self`参数，但 struct和他内部的字段可能刚才被Pin住了。你必须保证 Drop 实现中不会move内部字段。尤其上面介绍的，这意味着你的struct一定不能是`#[repr(packed)]`。浏览[上一节](https://doc.rust-lang.org/nightly/core/pin/index.html#drop-implementation) 学习如何编写 drop，让编译器帮助你别意外违反 pinning.


3. 你必须确保遵守 [drop 保证*（drop guarantee）](https://doc.rust-lang.org/nightly/core/pin/index.html#drop-guarantee)。一旦你struct被pinned，在没调用struct析构函数之前，struct的那片内存必须不被覆盖，不会释放（否则就是UB，这比较好理解）。这可能会很难办，正如 `VecDeque<T>`遇到的那样： 如果容器内的元素的析构函数panic，那 `VecDeque<T>` 的析构函数调用就会失败。这违反了 `drop` 保证（drop guarantee），因为这会导致容器内元素明明析构函数未被调用，但他们的内存却被释放（deallocated）。所幸`VecDeque<T>`并没有 pinning 映射（projections），所以上述情况不会导致 `unsoundness`

4. 你一定不能在struct被pinned期间，提供任何可能导致 structural field 被move出去的方法。例如，如果 struct 包含 `Option<T>`，恰好有个take方法，其函数形参为 `fn(Pin<&mut Self>) -> Pin<&mut T>`，这 take 方法可被用来将 T 从 pinned 的 `Struct<T>` move出去

这意味着 pinning 不能用于 结构化pin住 持有数据的这个字段
> 上面这句话有些生硬。换成英文感受下
> 
> which means pinning cannot be structural for the field holding this data.
> 意思是，Struct 的字段明明被 pinning 了，结果你提供方法将 `Pin<&mut Struct<T>>` 转换为 `Option<T>`，使得 T 逃脱了 Pin 的约束，这显然是**错误**的

再举一个更复杂的，将数据从已经被 pinned 的结构 move 出去的例子。想象下如果 `RefCell<T>`有个 `fn get_pin_mut(self: Pin<&mut Self>) -> Pin<&mut T>`方法，那我们可以这么玩

```rs
fn exploit_ref_cell<T>(rc: Pin<&mut RefCell<T>>) {
    { let p = rc.as_mut().get_pin_mut(); } // 将 Pin<&mut RefCell<T>> 转换为 Pin<&mut T>
    let rc_shr: &RefCell<T> = rc.into_ref().get_ref(); // Pin<&mut T> => &RefCell<T>
    let b = rc_shr.borrow_mut(); // RefCell本身有 borrow_mut() 方法，可以通过 &self 获得 &mut self。
    // https://doc.rust-lang.org/std/cell/struct.RefCell.html#method.borrow_mut
    // RefCell借助UnsafeCell，将 &self 转为 &mut self
    let content = &mut *b; // reborrow下，最终获取 &mut T
    // 通过上述 "努力" ，将 Pin<&mut Ref<T>> 一步步转化成 &mut T，逃脱了Pin的限制
    // 获得 &mut T后，就能将 T 从 Pin<&mut RefCell<T>> move出去
}
```

**翻译完上面之后，总结一句话**

> Pin作为rust zero cost的其中一例，只是编译期的约束。而如果你足够牛逼，仔细处理各种边界case，哪怕Pin住，通过unsafe随便玩，随便move都没问题。但为了绝对正确性，我们认为在 `Pin<T>` 期间利用各种手段将 T move 出去，是**绝对错误**的

----------------------------

重点看下面这连接，最后两个作为补充即可

> https://stackoverflow.com/questions/56058494/when-is-it-safe-to-move-a-member-value-out-of-a-pinned-future


https://github.com/rust-lang/futures-rs/blob/0.3.0-alpha.15/futures-util/src/future/map.rs#L36-L45

https://doc.rust-lang.org/nightly/core/pin/index.html#projections-and-structural-pinning