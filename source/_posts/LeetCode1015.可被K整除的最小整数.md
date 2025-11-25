---
title: LeetCode1015.可被K整除的最小整数
date: 2025-11-25 20:30:00
tags:
  - LeetCode
  - 算法
  - 哈希表
  - 数学
categories:
  - LeetCode
  - 算法
---

# [1015.可被K整除的最小整数](https://leetcode.cn/problems/smallest-integer-divisible-by-k/description/)
## 题目描述：
给定正整数 `k` ，你需要找出可以被 `k` 整除的、仅包含数字 `1` 的最 小 正整数 `n` 的长度。<br>

返回 `n` 的长度。如果不存在这样的 `n` ，就返回`-1`。<br>

注意： `n` 可能不符合 `64` 位带符号整数。<br>

## 示例 1:
>**输入:** k = 1 <br>
>**输出:** 1 <br>
>**解释:** 最小的答案是 n = 1，其长度为 1。<br>


## 示例 2:

>**输入:** k = 2<br>
>**输出:** -1<br>
>**解释:** 不存在可被 2 整除的正整数 n 。<br>


## 示例 3:

>**输入:** k = 3<br>
>**输出:** 3<br>
 >**解释:** 最小的答案是 n = 111，其长度为 3。<br>
## 提示:
- `1 <= k <= 10^5`

## 思路：
> 循环遍历乘`10`取余`k`，为了避免溢出对`k`取余,对`k`取余不会影响结果

## 代码：
```Java
class Solution {
    public int smallestRepunitDivByK(int k) {
        int res = 0;
        for (int i = 1; i < 100000; i++) {
            res = (res * 10 + 1) % k;
            if (res == 0) {
                return i;
            }
        }
        return -1;
    }
}
```
