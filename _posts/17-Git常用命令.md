---
title: Git常用命令
date: 2019-09-16
updated: 2019-09-16
issueid: 17
tags:
---
这里收集着我平时用到的Git操作 :)

### 推送到不同的分支

将本地的 `master` 分支推送到远程仓库的 `fixreadme` 分支

`origin` 代表着你的远程仓库地址

```
git push origin master:fixreadme
```

### 重新修改已经commit 的message

```
git commit --amend -m "re-commit"
```

### 删除远程branch

```
git push origin --delete iambranch
```

### 删除本地branch

安全删除，检查iambranch合并状态

```
git branch -d iambranch
```

强制删除

```
git branch -D iambranch
```

### 查看全部分支的提交 commit 状态

```
git log --all --oneline --decorate --graph
```
