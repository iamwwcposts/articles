---
updated: 2019-09-11
issueid: 6
tags:
title: Cgroup限制内存与节点的删除
date: 2019-09-11
---
首先不限制内存，让我们常见一个进程

```
stress --vm-bytes 200m --vm-keep -m 1
```

如下图，机器 `2G Mem`
共占用 `10%`, `200mb`

用 `top` 命令观察

![](https://raw.githubusercontent.com/iamwwc/noterepo/master/images/image_10.png)


现在利用 `Cgroup` 限制下内存

```
mkdir /sys/fs/cgroup/memory/testmem -p
cd /sys/fs/cgroup/memory/testmem
# 将当前bash pid 写入 tasks
echo $$ > tasks
echo 100m > memory.limit_in_bytes
```

然后在**当前bash**中启动

```
stress --vm-bytes 200m --vm-keep -m 1
```

另开一个shell， `top` 观察

![](https://raw.githubusercontent.com/iamwwc/noterepo/master/images/image_11.png)

可以看到，第二个进程的内存被限制到了 `100m`

### 删除 Cgroup

由于每个 Cgroup 都是完整的文件夹，所以我当时的方法是直接递归删除文件

```
rm -rf ./testm
```

结果报了成堆的如下错误。

```
rm: cannot remove 'testm/cgroup.procs': Operation not permitted
```

后来查了下

`http://blog.tinola.com/?e=21`

我们无法删除这些文件，但可以删除文件夹

```
rmdir ./testm
```

但由于当前 `bash` 的 `pid` 已经写入了 `testm/tasks`

在删除这个文件夹之前需要将pid移到 包含 testm 的文件夹的 `tasks` 当中

```
cd /sys/fs/cgroup/memory/
echo $$ >> tasks
```

最后执行删除操作

```
rmdir ./testm
```

还有一个更方便的 `cmd`

他会帮你将子 `group` 下 `tasks` 当中的 `pid` 全部移动到 `root` 当中

先创建嵌套 `cgroup`

```
cd /sys/fs/cgroup/memory/
mkdir wwc/w1
echo $$ > wwc/w1/tasks
```

如果我们直接

```
rmdir wwc/
```

会直接报错

```
rmdir: failed to remove 'wwc': Device or resource busy
```

手动移除需要我们将嵌套 `cgroup` 中的 `pid` 全部移动到 `memory/tasks`

这时就可以借助

```
cgdelete -r memory:wwc
```