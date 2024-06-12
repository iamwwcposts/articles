---
title: 两个链表相加-leetcode2
date: 2019-09-13
updated: 2019-09-13
issueid: 11
tags:
- 算法与数据结构
- Leetcode
---
[原题在这里](https://leetcode.com/problems/add-two-numbers/)

### 介绍

给你两个非空单向链表 `l1, l2`，将链表按照 `个十百千` 的顺序相加。

比如

```
输入： (1 -> 2 -> 3) + (4 -> 5 -> 6)

返回： (5 -> 7 -> 9)
```

这里最需要注意的是进位。

### 实现

```java
public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
    int carry = 0;
    ListNode origin = new ListNode(0), tmp = origin;

    // 遍历最大长度的链表
    while(!(l1 == null && l2 == null && carry == 0)) {
        // 如果 l1 或者 l2 为 null，那么值始终为 0
        int v1 = l1 == null ? 0 : l1.val;
        int v2 = l2 == null ? 0 : l2.val;
        int num = v1 + v2 + carry;
        tmp.next = new ListNode(num % 10);
        // 确定是否进位
        carry = num / 10;
        carry = num / 10;
        tmp = tmp.next;
        if (l1 != null) l1 = l1.next;
        if (l2 != null) l2 = l2.next;
    }
    return origin.next;
}
```

这道题和我原来的思路不同点在于

我往往会去遍历最小的链表，然后 `if` 分开处理

```java
while(l1 != null && l2 != null) {
  
}

if (l1 == null ){
  
}
if (l2 == null ){

}

```

却没有想到遍历最长链表，如果短链表为 `null`，那直接返回0就可以

写算法往往带来的是新的思考方式