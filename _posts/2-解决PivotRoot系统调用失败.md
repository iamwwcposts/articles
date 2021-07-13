---
title: 解决PivotRoot系统调用失败
date: 2019-09-11
updated: 2019-09-11
issueid: 2
tags:
---
### pivot_root 介绍

当我们fork新的进程，子进程会使用父进程的文件系统。

但如果我们想要把子进程的 `/` 文件系统修改成 `/var/run/wwcdocker/mnt/balabala` 怎么办呢？

这时候就要使用 `[pivot_root](https://linux.die.net/man/2/pivot_root)` 了

`int pivot_root(const char *new_root, const char *put_old);`

它的作用是将进程的 `/` 更改为 `new_root`,原 `/` 存放到 `put_old` 文件夹下。


### 遇到的问题

[存在问题的commit](https://github.com/iamwwc/wwcdocker/tree/5490793ea136bdffabb297b169197ea41fd6b4ec)

我想使用 `PivotRoot` 来修改容器进程的根文件系统路径。

但每次进行[系统调用](https://github.com/iamwwc/wwcdocker/blob/5490793ea136bdffabb297b169197ea41fd6b4ec/container/process.go#L100)，总会出 `Invalid arguments` 错误

Docker runC 下有这么[一段话](https://github.com/opencontainers/runc/blob/bbb17efcb4c0ab986407812a31ba333a7450064c/libcontainer/rootfs_linux.go#L646)

```
// Make parent mount PRIVATE if it was shared. It is needed for two
// reasons. First of all pivot_root() will fail if parent mount is
// shared. Secondly when we bind mount rootfs it will propagate to
// parent namespace and we don't want that to happen.
```

我们需要将父 root 修改为 PRIVATE

更多请参考

https://github.com/iamwwc/wwcdocker/issues/3

我在这个 issue 上有详细介绍