---
updated: 2019-09-11
issueid: 1
tags:
title: Travis配置Github集成发布
date: 2019-09-11
---
#### 写在前面

首先需要明确几个概念

<!--more-->

1. Tag与只与commit关联，与你在哪个分支给这个 commit 打tag 无关

下面是我的配置文件
```yml
language: go

go:
- 1.12
env:
  - GO111MODULE=on
script:
  - go build .
  
# build not trigger when I tag a commit, push it branch
# remove branch.only
# https://stackoverflow.com/questions/30156581/travis-ci-skipping-deployment-although-commit-is-tagged

notifications:
  email:
  # 只有失败才会邮件通知
    on_success: never
    on_failure: always

before_deploy:
  # 这两个环境变量已经提前写入 travis 中
  - git config --local user.name "${GIT_USER_NAME}"
  - git config --local user.email "${GIT_USER_EMAIL}"
  - git tag

deploy:
  # 发布到github release
  provider: releases
  # 去 https://github.com/settings/tokens 申请 token，写入 travis 环境变量，最后这里引用
  api_key: $GITHUB_TOKEN
  # 打包出来的 wwcdocker 存放在根目录，所以这里直接上传
  file: "wwcdocker"
  skip_cleanup: true
  # 只有 commit 中有tag时才会触发
  on:
    tags: true
```

在Github中，你上传的tag会同时出现在 releases 与 tags 面板

### Github 中 releases 与 tags 面板的区别

https://stackoverflow.com/questions/28496319/github-a-tag-but-not-a-release

而对于 `latest` 的定义。只有你在这个 `tag` 对应的 `commit` 发布的文件。
也即 `travis` 向某个 `commit` 提交了额外的文件，而这时 `commit` 最新，`github` 自动让其成为 `latest release`

如果你想 在 提交到 master 分支且有 tag 时触发构建，你大概会使用如下的配置

```yml
branches:
  only:
    - master

deploy:
  on:
    tags: true
```
但最后的结果不会如你所愿。

https://stackoverflow.com/questions/30156581/travis-ci-skipping-deployment-although-commit-is-tagged

Travis 区分如下几个条件

1. 由 分支 触发构建
2. 由 tag 触发构建

所以你的推送会分别触发两个条件，而不会等两个条件都满足才触发。


所以我在后面的 commit 中移除了 `branches.only.master`

https://github.com/iamwwc/wwcdocker/commit/b244db052a2bc2becc80cbe8bcb5afa76b84de64

