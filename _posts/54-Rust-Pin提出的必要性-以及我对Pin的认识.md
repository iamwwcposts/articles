---
title: Rust-Pin提出的必要性-以及我对Pin的认识
date: 2021-06-08
updated: 2021-06-08
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

rust中默认类型都是Unpin的，也即，可以move

当需要开发自引用结构时，可以将 struct 某个自引用字段用Pin包裹，这样struct就是 !Unpin，想要Pin住这种结构常规用 `Pin::new(x)`不行了。必须 Box::pin(x);

典型的自引用结构是 linked_list

https://github.com/tokio-rs/tokio/blob/6b9bdd5ca25bd3f30589506de911db73f7dbf8b4/tokio/src/time/driver/entry.rs#L299


TimerEntry -> TimerShared -> TimerSharedPadded#pointers -> LinkedList -> TimerShared

由于TimerShared结构间接自引用，所以整个链条都需要通过PhantomPined 来 !Unpin

---------------------

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

tokio::pin!用的new_unchecked 并不在乎到底Pin的是在stack还是heap，如果指针指向stack，就是pin在stack，指向heap，就是pin在heap。

tokio::pin有意思的点是，`let mut $x = $x` 先 move 一下，确保 x moveable ，也即拥有ownership

然后Pin住。Pin 根本就不在乎是stack 还是 heap，Pin只是代表不能被 move，

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