---
updated: 2019-09-13
issueid: 12
tags:
- 算法与数据结构
- Leetcode
title: 最长无重复字串-leetcode3
date: 2019-09-13
---
题目来自

https://leetcode.com/problems/longest-substring-without-repeating-characters/


### 题目描述

给你一个字符串s，输出 最长且不重复的字串 的长度

比如输入 `tmsmfdut`，因为最长的不重复字串是 `smfdut`，所以输出 `6`

### 解决方案

既然是最大长度，那肯定需要记录 `max`，使用双索引，i用来遍历s， `j` 为不重复字串的起始索引，我们将字符作为 `key`，下标索引作为 `value`作为索引存放到 `hashmap`中。在后面遍历s时，如果字符存在于 `map` 中，那去 当前距离 `i - j + 1` 与 `max` 的最大值。

代码如下
```java
public int lengthOfLongestSubstring(String s) {
    // 存放 character 与 下标索引
    HashMap<Character, Integer> map = new HashMap();
    int max = 0;
    for (int i = 0, j = 0; i < s.length(); i ++) {
        if(map.containsKey(s.charAt(i))) {
            j = Math.max(j,map.get(s.charAt(i)) + 1);
        }
        map.put(s.charAt(i),i);
        max = Math.max(max, i - j + 1);
    }
    return max;
}
```

为什么 `j = Math.max(j,map.get(s.charAt(i)) + 1);`中需要 `+1`？

看下图，这时我们 j 为1，i为3，hashmap中存放着第一个m，
我们通过 `map.get(s.charAt(i))`获得的是m对应的下标1

如果 `j` 等于 `1` ，那么会直接与下标为 `3` 的 `m` 重复，所以我们需要 `+1`，将 `j` 指向 `map` 中 `m` 对应的下标索引的后一位，也就是 `s` 对应的 `2`

![image](https://user-images.githubusercontent.com/24750337/64864439-ae8e9200-d669-11e9-8774-ecec70544a14.png)


