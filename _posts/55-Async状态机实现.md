---
issueid: 55
tags:
- Rust
title: Async状态机实现
date: 2021-06-18
updated: 2021-06-18
---
## Rust中的状态机实现

我概括一下，引入 Future 必须解决自引用被swap的问题

Pin通过 提供 &Self 来让编译器进行入参检查，mem::swap(& mut self) 需要 mut 类型，所以编译器类型检查失败而退出

[](https://cloud.tencent.com/developer/article/1628311)

https://rust-lang.github.io/async-book/04_pinning/01_chapter.html

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
https://discord.com/channels/500028886025895936/500336333500448798/854955299522084914

由于保存变量状态， poll 中获得这状态通过变量引用访问，所以struct往往会保存变量的引用，这就使得 future 不能被移动，所以编译器会为生成的 future 状态机增加 PhantomPinned，使得 future 是 !Unpin

https://os.phil-opp.com/async-await/#pinning-and-futures

也正是由于 future !Unpin，为了再次 poll 这个future，必须重新 Pin 住

所以 shadowsocks-rust 这里重新Pin住

https://github.com/willdeeper/shadowsocks-rust/blob/6ff4f5b04b8e22d36cd54477cfcadf8cb403de21/bin/sslocal.rs#L463

虽然看着 server 是 stack local，但 async 会将其保存到 struct，确保pin的整个过程中server始终存在。

https://os.phil-opp.com/async-await/#stack-pinning-and-pin-mut-t

> 实际上，我并没有找到具体文档说，rustc实现的 future 一定会 opt out Unpin，使其 !Unpin。
> 但根据我的理解，既然 future 翻译成的状态机是自引用结构，那就不允许move。自然就需要为其 !Unpin
> 从下面的回答可证明是自引用结构
> https://users.rust-lang.org/t/is-it-possible-to-de-sugar-async-with-normal-rust-syntax/37789/3
> 至于说的 !Unpin 是我的推论，不过我认为我推论是正确的。

----------------
https://os.phil-opp.com/async-await/#saving-state-1

> Language-supported implementations of cooperative tasks are often even able to backup up the required parts of the call stack before pausing. As an example, **Rust's async/await implementation stores all local variables that are still needed in an automatically generated struct**
-----------

Rust没找到在线编辑器看编译后的代码，但stackless理论中，js和rust最接近，后面不不明白，可以看js编译后是如何利用函数闭包保存状态的

> https://babeljs.io/repl#?browsers=ie%206%0A&build=&builtIns=false&corejs=3.6&spec=false&loose=false&code_lz=NwCghgzgngdgxgAgGYFd4BcCWB7GCQCUCA3gFACQqGOeEANgKYMAOIAthEWQj7wE4N0KPnhgMA7ggAKfbG0wQGIARGx0AbgwQBeAHwJF6ACqY2DbCnTKGqjQwA0CDgQLBSvBAF9S73pFiIVHBYuAgAJgxs2IQkvh5wuLYMAHR02ADmIABEHOkAjFmucbwJMBDoCGA6CACsbh48jBUARtVg4mCYFfRMrHkADINFDQgA9KMIgBexlR1dCICo-oAUroBhcmDFPOMIgNDugGj-gIvRgF-KgCFugMdygIAegEAMVYDVEYC3qYCgyoCAOoAU6oC_ioAOpoAA5oBQcoBCOutjCaATfjAOwWgHvlQCncoBd-UAGtqANqdAIAGgEP5QD2BgDNoBlfQ-gFjFQCE1tdAMAxi3RE0AWPKAfdjAPFpBnEXTgAAsEIBt-MApoqAaDlAABygCo5REkhDPQAuplzKtB4AgPoBaOUAMP-AXzDAGyOgCxNXFEmadCqAf3lAOn6_jF5TAcAA1oAV-MAe2p8wBi8ojANOagAm_QC_Cc8giEYEcHhLYXy2YAjdKBgFPlQC70YBAyMADdGAeXk-TrEHrDYA3PUAsHKAejNAIhGgBu5QAceoAX6KOHyJgAgLKWATgtADTmgEYdRM54kjTb7QAWaqTAHYeUP2x3OsLe7wegEAGBgwFBsTvoPgoYIA0pJVIZbK5ABMAC4EFkEABqSrDXjeYrtNXhSLRIqeAiEYBAA&debug=false&forceAllTransforms=true&shippedProposals=false&circleciRepo=&evaluate=true&fileSize=false&timeTravel=false&sourceType=module&lineWrap=true&presets=es2017%2Ces2015-loose&prettier=false&targets=&version=7.14.1&externalPlugins=


## 为什么说 Pin 是零开销？

Pin! 的内容会分配到堆上防止移动，对于自引用结构为了防止移动一定会内存**分配到堆上**，零开销的意思是不会进行**多余的**堆分配。

mem::swap 要求 参数为 `&mut T`，只要我们保证不泄漏出 mut 就可保证用户代码不会 move 内存造成**不安全编程**

除非显式 !Unpin， 否则编译器会自动实现Unpin，也就是Rust里默认变量是可以移动的

比如下面这代码
https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=636b71d4acef346a9ce810b735908cb1

https://stackoverflow.com/a/49917903/7529562

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


