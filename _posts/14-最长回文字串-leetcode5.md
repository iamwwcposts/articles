---
date: 2019-09-14
updated: 2019-09-14
issueid: 14
tags:
- 算法与数据结构
- Leetcode
title: 最长回文字串-leetcode5
---
题目来自

https://leetcode.com/problems/longest-palindromic-substring/

### 题目介绍

给定一个字符串 `s` ，输出 `s` 对应的最长回文字串

比如

```
Input: s = "babad"

Output: s = "bad"
```

解释：

`bad` 是 `babad` 的最长回文字串

### 解决方案

之前碰到过类似的一道题，判断 s 是否回文，其中之一的方法是从两边开始向里扫描。但对于这道题，求最长回文字串向里扫描不适用。

我们采用中间向外扫描， 不断比较记录最大的长度， 跳过肯定是回文的重复字符序列。

然后利用双指针左右移动。

```java
public String longestPalindrome(String s) {
    if (s.length() < 2) {
        return s;
    }
    int max = 0 , start = 0;
    /*
    * i 是移动的起始点
    * 比如 s.length() == 8
    * 现在max == 6
    * i只需要到 (8 - 6 / 2 ) == 5 就可以，没必要继续向后扫描
    */
    for(int i = 0 ; i < s.length() - max / 2; i ++) {
        int j = i;
        int k = i;
        // 跳过一定是回文的重复字符
        while(k < s.length() - 1 && s.charAt(k) == s.charAt(k + 1)) {
            k ++;
        }
        // 从重复序列的左右同时开始移动
        while(j > 0  && k < s.length() - 1 && s.charAt(j - 1) == s.charAt(k + 1)) {
            j --;
            k ++;
        }
        int len = k - j + 1;
        if (max <= len){
            start = j;
            max = len;
        }
    }
    return s.substring(start, start + max);
}
```
### 再度优化

上面代码在 leetcode 上运行使用了 `18ms`，我们接下来继续优化，让它逼近 `1ms`

首先要做的是 `String` 转化为 `char[]` 直接索引，并且数组长度也存入变量，不再调用 `s.length()`


```java
public String longestPalindrome(String s) {
    if (s.length() < 2) {
        return s;
    }
    char[] chars = s.toCharArray();
    int charlen = chars.length;
    int max = 0 , start = 0;
    for(int i = 0 ; i < charlen - max / 2; i ++) {
        int k = i;
        int j = i;
        while(k < charlen - 1 && chars[k] == chars[k + 1]) {
            k ++;
        }
        while(j > 0  && k < charlen - 1 && chars[j - 1] == chars[k + 1]) {
            j --;
            k ++;
        }
        int len = k - j + 1;
        if (max <= len){
            start = j;
            max = len;
        }
    }
    return s.substring(start, start + max);
}
```

以上代码运行占用时间 `10ms`

##### 继续优化

```java
public String longestPalindrome(String s) {
    if (s.length() < 2) {
        return s;
    }
    char[] chars = s.toCharArray();
    int charlen = chars.length;
    int max = 0 , start = 0;
    for(int i = 0 ; i < charlen - max / 2; i ++) {
        int k = i;
        int j = i;
        while(k < charlen - 1 && chars[k] == chars[k + 1]) {
            k ++;
        }
        i = k;
        while(j > 0  && k < charlen - 1 && chars[j - 1] == chars[k + 1]) {
            j --;
            k ++;
        }
        int len = k - j + 1;
        if (max <= len){
            start = j;
            max = len;
        }
    }
    return s.substring(start, start + max);
}

```

这次我们就加一行代码，`i = k;`，当 `while` 搜寻完重复字符之后，那 `i` 也没必要在从这里继续检索了。

所以直接跳过重复字符，从其后面继续检索。

以上代码运行占用时间 `2ms`

优化结束 :)

### 感想

到现在也是做了好多的 `leetcode` 题目了，感觉对代码的细节控制更深入了。

比如这个代码片段

```java
while(j > 0  && k < s.length() - 1 && s.charAt(j - 1) == s.charAt(k + 1)) {
    j --;
    k ++;
}
```

如果按我之前的写法，我会写成下面这样

```java
while(j >= 0  && k < s.length() && s.charAt(j) == s.charAt(k)) {
    j --;
    k ++;
}
```

看起来没什么问题，但 `j` 可能为 `-1`，而 `k` 可能为 `s.length()`，已经索引溢出了。

为了解决这个问题，我会在后面加上更臃肿的代码

```java
if (j == -1) {
  j ++;
}

if (k == s.length()) {
  k --;
}
```

算法题写多了，感觉自己更灵光了 :)