---
title: 一个简单的React模板以及一些React运作流程的简短分析
date: 2020-04-25
updated: 2020-04-26
issueid: 42
tags:
---
我知道你看不懂的 😂
<!--more-->
<p class="codepen" data-height="265" data-theme-id="light" data-default-tab="js,result" data-user="tryfrontend" data-slug-hash="PoPpyZw" style="height: 265px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;" data-pen-title="Template of React">
  <span>See the Pen <a href="https://codepen.io/tryfrontend/pen/PoPpyZw">
  Template of React</a> by csstry (<a href="https://codepen.io/tryfrontend">@tryfrontend</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>

<img src="https://user-images.githubusercontent.com/24750337/80272648-e086cd00-86fd-11ea-9e20-3042f25f1f94.png" width=250px height=200px>

比如React，点击一个button，从触发事件到virtual dom修改都是同步进行

<img src="https://user-images.githubusercontent.com/24750337/80306426-d34d0980-87f5-11ea-8d25-80f86e1dc213.png" width=250px height=200px>

以及Vue
<img src="https://user-images.githubusercontent.com/24750337/80306494-49ea0700-87f6-11ea-82f8-e8179cae7a74.png" width=250px height=200px>

都是触发渲染之后diff出来Node不同之后就patch这个Node，不存在diff出来差别之后异步等到下一个周期才更新，也不存在diff完node之后统一patch，都是找出来不同就直接修改，树的深度优先遍历