---
tags:
- Rust
title: Async状态机实现
date: 2021-06-18
updated: 2023-12-19
issueid: 55
---
## Rust中的状态机实现

我概括一下，引入 Future 必须解决自引用被swap的问题

Pin通过 提供 &Self 来让编译器进行入参检查，mem::swap(& mut self) 需要 mut 类型，所以编译器类型检查失败而退出

[](https://cloud.tencent.com/developer/article/1628311)

<https://rust-lang.github.io/async-book/04_pinning/01_chapter.html>

Future 任务会跨越线程执行，我们知道Future可以编译成状态机yield执行，那在从Future Pending 到 Done 的过程中需要将全部的状态统一保存起来。

这里最好的办法是 struct，也即 async 中所有的变量都在 struct 上分配，并通过self引用。

整个 Future 执行期间全部的变量都通过 enum 进行保存。

## Future 是如何保存执行状态的？

编译之前

```rs
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

struct Delay {
    when: Instant,
}

impl Future for Delay {
    type Output = &'static str;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<&'static str>
    {
        if Instant::now() >= self.when {
            println!("Hello world");
            Poll::Ready("done")
        } else {
            // Ignore this line for now.
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

#[tokio::main]
async fn main() {
    let when = Instant::now() + Duration::from_millis(10);
    let future = Delay { when };

    let out = future.await;
    assert_eq!(out, "done");
}
```

编译之后

```rs
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};
// 每次返回的Future都会存放到 enum 结构体中。这样在Future未结束之前Future会在 enum 持久化
enum MainFuture {
    // Initialized, never polled
    State0,
    // Waiting on `Delay`, i.e. the `future.await` line.
    State1(Delay),
    // The future has completed.
    Terminated,
}

impl Future for MainFuture {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<()>
    {
        use MainFuture::*;

        loop {
            match *self {
                State0 => {
                    // 这些代码并不会执行多次
                    // https://discord.com/channels/500028886025895936/500336333500448798/854955299522084914
                    let when = Instant::now() +
                        Duration::from_millis(10);
                        // 看编译前的代码
                        // 由于when在.await使用，编译器自动将when保存到enum
                        // 所以不用担心async function调用返回后stack里的变量没了
                        // 编译器会分析出来，帮你处理
                    let future = Delay { when };
                    *self = State1(future);
                }
                State1(ref mut my_future) => {
                    match Pin::new(my_future).poll(cx) {
                        Poll::Ready(out) => {
                            assert_eq!(out, "done");
                            *self = Terminated;
                            return Poll::Ready(());
                        }
                        Poll::Pending => {
                            return Poll::Pending;
                        }
                    }
                }
                Terminated => {
                    panic!("future polled after completion")
                }
            }
        }
    }
}
```

比如如下async代码块

```rs
async {
    let mut x = [0; 128];
    let read_into_buf_fut = read_into_buf(&mut x);
    read_into_buf_fut.await;
    println!("{:?}", x);
}
```

下面是desugar的代码

```rs
struct AsyncFuture {
    x: [u8; 128],
    read_into_buf_fut: ReadIntoBuf
    state: State,
}

enum State { 
    Start, 
    AwaitingReadIntoBuf, 
    Done
}

struct ReadIntoBuf {
    buf: &'a mut [u8] // 可能被实现为原始指针，这儿只是为了方便描述
}
```

会被rustc编译成Future对象，并且跨.await的变量被封装到struct保存，通过生成state维护状态机的状态

并不是全部的local variable 都会被保存，rustc只会保存await需要的变量，这些变量特点是await返回后就会被销毁，所以需要保存到struct

<https://discord.com/channels/500028886025895936/500336333500448798/854955299522084914>

由于保存变量状态， poll 中获得这状态通过变量引用访问，所以struct往往会保存变量的引用，这就使得 future 不能被移动，所以编译器会为生成的 future 状态机增加 PhantomPinned，使得 future 是 !Unpin

<https://os.phil-opp.com/async-await/#pinning-and-futures>

<https://www.reddit.com/r/learnrust/comments/fkvr15/why_cant_an_async_block_or_fn_implement_unpin/>

也正是由于 future !Unpin，为了再次 poll 这个future，必须重新 Pin 住

所以 shadowsocks-rust 这里重新Pin住

<https://github.com/willdeeper/shadowsocks-rust/blob/6ff4f5b04b8e22d36cd54477cfcadf8cb403de21/bin/sslocal.rs#L463>

虽然看着 server 是 stack local，但 async 会将其保存到 struct，确保pin的整个过程中server始终存在。

<https://os.phil-opp.com/async-await/#stack-pinning-and-pin-mut-t>

Future 可能会出现自引用结构，比如 `https://os.phil-opp.com/async-await/#self-referential-structs`，在`await`之后获取了array的引用。这种情况下future move会出问题

<https://os.phil-opp.com/async-await/#pinning-and-futures>

> The reason that this method takes self: Pin<&mut Self> instead of the normal &mut self is that future instances created from async/await are often self-referential, as we saw above. By wrapping Self into Pin and letting the compiler opt-out of Unpin for self-referential futures generated from async/await, it is guaranteed that the futures are not moved in memory between poll calls
----------------
<https://os.phil-opp.com/async-await/#saving-state-1>

> Language-supported implementations of cooperative tasks are often even able to backup up the required parts of the call stack before pausing. As an example, **Rust's async/await implementation stores all local variables that are still needed in an automatically generated struct**
-----------

Rust没找到在线编辑器看编译后的代码，但stackless理论中，js和rust最接近，后面不不明白，可以看js编译后是如何利用函数闭包保存状态的

[js async编译后的状态机](https://babeljs.io/repl#?browsers=defaults%2C%20not%20ie%2011%2C%20not%20ie_mob%2011&build=&builtIns=false&corejs=3.21&spec=false&loose=false&code_lz=NwCghgzgngdgxgAgGYFd4BcCWB7GCQCUCA3gFACQqGOeEANgKYMAOIAthEWQj7wE4N0KPnhgMA7ggAKfbG0wQGIARGx0AbgwQBeAHwJF6ACqY2DbCnTKGqjQwA0CDgQLBSvBAF9S73pFiIVHBYuAgAJgxs2IQkvh5wuLYMAHR02ADmIABEHOkAjFmucbwJMBDoCGA6CACsbh48APSNCICb8YAziYD0ZmDiYJjogNJygLJKxTyMFQBG1T19FfRMrHkADCtFDQjNbV0z_QOAcCo-65uAF7GVvf0IgKj6gBSugGFyYKMbLYDQ7oBo_oCL0YBfioAhboDHcoCAHoAgBiqgGqIwC3qYBQZUAgDqACnVAL-KgAdTQAA5oAoOUAQjqPTatQDsFoB75UAp3KAXflABragDanQCABoBD-UA9gaYlqAZX1EYBYxUAhNYgwDAMdc6QhAFjygH3YwDxaQZxP04AALBCAbfjAKaKgGg5QAAcoAqOQp3JhgBdTRWVaDwBCIwC0coAYf8AvmGANkdAFiaLM5Z1mCEA_vKAdP1_DrymA4ABrQAr8YA9tW5gDF5CmAac1ABN-gF-EmFBEIwb6QvUk7mywBG6a1AKfKgF3owCBkYAG6MA8vLcp2IF3uwBueoBYOU6gEQjQA3coAOPUAL9HfRGcwAQFgbAJwWgBpzQCMOmXG1yji0PoALNR5gDsPQkfH4AknwhGQwCADAwYCg2HP0HwUMFHqUkqkMtlcgAmABcCCyCAA1JU1rxvMUdhUIlFCG5PARn0A&debug=false&forceAllTransforms=false&modules=false&shippedProposals=false&circleciRepo=&evaluate=false&fileSize=false&timeTravel=false&sourceType=module&lineWrap=true&presets=react%2Cstage-2&prettier=false&targets=&version=7.23.6&externalPlugins=%40babel%2Fplugin-transform-regenerator%407.23.3&assumptions=%7B%7D)

## 为什么说 Pin 是零开销？

Pin! 的内容会分配到堆上防止移动，对于自引用结构为了防止移动一定会内存**分配到堆上**，零开销的意思是不会进行**多余的**堆分配。

mem::swap 要求 参数为 `&mut T`，只要我们保证不泄漏出 mut 就可保证用户代码不会 move 内存造成**不安全编程**

除非显式 !Unpin， 否则编译器会自动实现Unpin，也就是Rust里默认变量是可以移动的

比如下面这代码
<https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=636b71d4acef346a9ce810b735908cb1>

<https://stackoverflow.com/a/49917903/7529562>

## Javascript Promise 和 Rust Future 的不同

JavaScript 的 Promise 调用可能会返回新的 Promise，而Rust的Future只会返回 Pending 或 Ready，并不会返回新的 Future

Future在Rust中是块写法

```rs
thread::spawn(async {

});
```

这 async 编译为 Future，Future::poll返回 `Pending / Ready`，想孵化新的异步任务需调用运行时提供的孵化方法，而不是通过返回 Future

```rs
tokio::spawn(async {
    // spawn 孵化新的Future
    let join_hander = tokio::spawn(async {

    });
    // future#poll 会先返回 join_hander 的完成状态
    join_hander.await?;
});
```

## JS中的状态机

我将用 Javascript 来演示 async 如何实现。 async 原理类似 C# 的 async Task，Rust中的 ?. Future

async 本质是依赖 回调函数

```js
async function fetchDataFromRemote(path) {
    const url = 'https://' + path;
    const r1 = await api.get(url)
    const r2 = await api.getFrom(url)
    return [r1,r2]
}

const data = await fetchDataFromRemote('api.github.com/repos/tokio-rs/mini-redis/languages')
```

```js
fetchDateFromRemote() {
    const url = 'https://' + path;
    let state = 0;
    let finalState = 2;
    let waitFor = null
    switch(state) {
        case 0: {
            waitFor = api.get(url);
            return waitFor;
        }
    }
}
```

又或者下面从babel拷贝过来的代码

```js
async function getActive(keyId: string) {
    if (typeof itemId === 'string') {
        const r = await (await api.items.get(keyId, itemId)).data;
        setDefault(r.value);
        setFields({
            key: 'value',
            value: r.value
        });
        return;
    }
    const k = await (await api.keys.get(keyId)).data;
    if (Number.parseInt(k?.active) !== -1) {
        const active_item = await (await api.items.get(keyId, itemId)).data
        
        setDefault(active_item.value);
        setFields({
            key: 'value',
            value: active_item.value
        });
    }
}
```

```js
function getActive(keyId) {
            return __awaiter(this, void 0, void 0, function () {
                var r, k, active_item;
                return __generator(this, function (_a) {
                    switch (_a.label) {
                        case 0:
                            if (!(typeof itemId === 'string')) return [3 /*break*/, 3];
                            return [4 /*yield*/, _api__WEBPACK_IMPORTED_MODULE_5__["default"].items.get(keyId, itemId)];
                        case 1: return [4 /*yield*/, (_a.sent()).data];
                        case 2:
                            r = _a.sent();
                            setDefault(r.value);
                            setFields({
                                key: 'value',
                                value: r.value
                            });
                            return [2 /*return*/];
                        case 3: return [4 /*yield*/, _api__WEBPACK_IMPORTED_MODULE_5__["default"].keys.get(keyId)];
                        case 4: return [4 /*yield*/, (_a.sent()).data];
                        case 5:
                            k = _a.sent();
                            debugger;
                            if (!(Number.parseInt(k === null || k === void 0 ? void 0 : k.active) !== -1)) return [3 /*break*/, 8];
                            return [4 /*yield*/, _api__WEBPACK_IMPORTED_MODULE_5__["default"].items.get(keyId, itemId)];
                        case 6: return [4 /*yield*/, (_a.sent()).data];
                        case 7:
                            active_item = _a.sent();
                            setDefault(active_item.value);
                            setFields({
                                key: 'value',
                                value: active_item.value
                            });
                            _a.label = 8;
                        case 8: return [2 /*return*/];
                    }
                });
            });
        }
```
