---
date: 2020-03-03
updated: 2020-03-03
issueid: 33
tags:
- 杂七杂八
title: Ubuntu如何开启自挂在 Vmware shared folder?
---
我想要将Windows整个D盘挂载到 Ubuntu 的 `/root/d` 下，使用 `vmhgfs` 一直提示我 `Error: cannot canonicalize mount point: No such file or directory`，查了好久，之后发现 `vmhgfs`已经 `out of date`，对于我 Ubuntu 18.04版本应该使用 `vmhgfs-fuse`

下面这个回答解决了问题

https://askubuntu.com/a/1051620/956906

另外参考 

https://github.com/vmware/open-vm-tools/issues/248