---
title: ABI（Application Binary Interface）是什么？
date: 2020-02-28
updated: 2020-02-28
issueid: 32
tags:
- 基础
---
ABI是一组决定调用转换（calling convention），放置结构的规则。

Pascal利用栈，按照与C语言相反的顺序传递参数，所以Pascal和C不能编译成相同相同的ABI。

**比如，对于函数调用参数传递，C语言参数从右向左压栈，而Pascal与其相反**

C和Pascal编译器各自的标准也隐藏着却是如此（最终编译出来的ABI不兼容）

C++编译器并不能定义“标准的”方式来管理变量，因为C++并没有定义这个标准。C++的变量管理在Windows上的编译器与GNU编译器之间并不兼容，所以这两个编译器产生的代码并不是ABI兼容的。

> https://stackoverflow.com/questions/2171177/what-is-an-application-binary-interface-abi
> *看英文能明白是什么意思，结果翻译过来那个味就变了 :)*