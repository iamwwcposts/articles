---
date: 2019-09-13
updated: 2019-10-31
issueid: 10
tags:
title: 如何优雅地使用Git合并多个commits？
---
你可能有过下面的经历

自己在本地开发，由于 `Github` 配置了CI，所以需要将新的代码 push到 github 来测试

所以你的 `commit` 上会有大量无用的 `commit` 与 `commit message`

比如：
update, done, fix,update,update ...

当你测试通过之后，你想把上面的 `commit` 合并成一个显示。

如果这是你自己的仓库还好，如果是 fork 的仓库，在开发完成之后你想要 pull request，留着那么多的 commits 肯定不合适

**那如何将多个 `commits` 记录合并成一个 `commit` 并且保留 `commit` 的代码呢？**

先创建几个 `commit` 记录

```
mkdir test
cd test

git init

echo 0 > a.md
git add .
git commit -m "add 0"

echo 1 >> a.md
git add .
git commit -m "add 1"

echo 2 >> a.md
git add .
git commit -m "add 2"

echo 3 >> a.md
git add .
git commit -m "add 3"
```

现在我们看一下 `commit` 记录

```
git log
```

```
commit 338fe65506a59f02e79badce0ff2c4dd77f08f8c (HEAD -> master)
Author: iamwwc <**@gmail.com>
Date:   Thu Sep 12 22:57:19 2019 +0800

    add 3

commit e693fb400991d69005caa026b4f0f2fc8dbce5a8
Author: iamwwc <**@gmail.com>
Date:   Thu Sep 12 22:57:19 2019 +0800

    add 2

commit 278713a3b6eb0e5b72670f007bdbc3bb0a1f9aa1
Author: iamwwc <**@gmail.com>
Date:   Thu Sep 12 22:57:19 2019 +0800

    add 1

commit 3011974bb76dbc5ccc75abf9a528605c60cbb51f
Author: iamwwc <**@gmail.com>
Date:   Thu Sep 12 22:57:18 2019 +0800

    add 0
```

你现在想把 `add3`, `add2` 合并到 `add1` 并且保留全部的内容，并将这个新的commit的记录改写为 `add1,2,3`

有如下两种方式

#### Git rebase

```
git rebase -i HEAD~3
// -i 是 --interactive，交互式，Git 会打开操作窗口
// 当然，你也可以 `git rebase -i 3011974bb76dbc5ccc75abf9a528605c60cbb51f`
```

这时 `git` 会为你打开 `rebase` 的对话框

你会看到如下的内容。

```
pick 278713a add 1
pick e693fb4 add 2
pick 338fe65 add 3
```

而对应着，`git rebase` 有如下命令

```
# p, pick = use commit
# r, reword = use commit, but edit the commit message
# e, edit = use commit, but stop for amending
# s, squash = use commit, but meld into previous commit
# f, fixup = like "squash", but discard this commit's log message
# x, exec = run command (the rest of the line) using shell
# d, drop = remove commit
```

我们这里使用 `fixup`，丢弃掉 `log message`

```
r 278713a add1,2,3
f e693fb4 add 2
f 338fe65 add 3
```

关闭对话框之后进入到 `commit message` 的界面，这时填写 `add1,2,3`并关闭

再次 `git log`

```
commit 74d943e839c96fe256bd3324a6da0a5e6fbeafa1 (HEAD -> master)
Author: iamwwc <**@gmail.com>
Date:   Thu Sep 12 22:57:19 2019 +0800

    add1,2,3

commit 3011974bb76dbc5ccc75abf9a528605c60cbb51f
Author: iamwwc <**@gmail.com>
Date:   Thu Sep 12 22:57:18 2019 +0800

    add 0
```

完成修改 :)

#### Git reset --soft

`--soft` 保证你的 `commit` 不丢失，内容不会被删除，仅仅移动了 `HEAD`

我们将 HEAD 移动到 add1 之前(也就是 `add0`)

```
git reset --soft 3011974bb76dbc5ccc75abf9a528605c60cbb51f
```

使用 `git log`你会发现只剩下 `add0` 的提交

下面添加全部的文件

```
git add .
git commit -m "add1,2,3"
```

完成

之后往 GitHub 提交时由于 `commit` 的变更你会推送失败

可以 `force-with-lease` 更新
```
git push --force-with-lease
```

不推荐使用 `-force`

比如A，B一起在 `master` 分支开发，B开发完成之后将代码 `push` 到了 `master`

A在本地 `master` 分支进行了 `rebase` 操作。这时A与 `remote/master` 已经冲突了

如果A 使用 `git push --force`，则会覆盖掉 B 推送到 `master` 上的代码

`https://stackoverflow.com/a/37460330/7529562`

参见

`https://github.com/Jisuanke/tech-exp/issues/13`