---
title: 罗马数字求和-leetcode13
date: 2019-09-17
updated: 2019-09-17
issueid: 18
tags:
- Leetcode
---
题目来源

https://leetcode.com/problems/roman-to-integer/

### 题目描述

给定一个字符串 s，计算得出 s 对应的值。

```
Input: "MCMXCIV"
Output: 1994
Explanation: M = 1000, CM = 900, XC = 90 and IV = 4.
```

### 题目分析

罗马数字计算最关键的是

1. 通常数字大小从左到右依次递减。
2. 如果 s[i] < s[i + 1]，说明这两个数字可以组合在一起，且值为 s[i + 1] - s[i]。
3. 如果 s[i] >= s[i + 1]，则值为 s[i]

### 实现

````java
class Solution {
    public int romanToInt(String s) {
        char[] arrs = s.toCharArray();
        int result = 0;
        HashMap<Character,Integer> map = new HashMap<>();
        map.put('I',1);
        map.put('V',5);
        map.put('X',10);
        map.put('L',50);
        map.put('C',100);
        map.put('D',500);
        map.put('M',1000);
        for(int i = 0 ; i < arrs.length; i ++) {
            int current = map.get(arrs[i]);
            // 这里需要处理数组索引溢出
            // '1'表明溢出，我们用 0 代替
            int next = map.getOrDefault(i < arrs.length - 1 ? arrs[i + 1] : '1',0);
            if (next > current) {
                result += next - current;
                i += 1;
                continue;
            }
            result += current;
        }
        return result;
    }
}
```