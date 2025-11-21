---
title: LeetCode1930.长度为3的不同回文子序列
date: 2025-11-21 22:56:00
tags:
  - LeetCode
  - 算法
  - 字符串
  - 枚举
  - 哈希表
categories:
  - LeetCode
  - 算法
---

# [1930.长度为3的不同回文子序列](https://leetcode.cn/problems/unique-length-3-palindromic-subsequences/description/)
## 题目描述：
给你一个字符串 `s` ，返回 `s` 中 长度为 `3` 的不同回文子序列 的个数。<br>

即便存在多种方法来构建相同的子序列，但相同的子序列只计数一次。<br>

回文 是正着读和反着读一样的字符串。<br>

子序列 是由原字符串删除其中部分字符（也可以不删除）且不改变剩余字符之间相对顺序形成的一个新字符串。<br>

- 例如，`"ace"` 是 `"abcde"` 的一个子序列。

## 示例 1:
>**输入:** s = "aabca" <br>
>**输出:** 3 <br>
>**解释:** 长度为 3 的 3 个回文子序列分别是：<br>
> - "aba" ("aabca" 的子序列)
> - "aaa" ("aabca" 的子序列)
> - "aca" ("aabca" 的子序列)

## 示例 2:

>**输入:** s = "adc"<br>
>**输出:** 0<br>
>**解释：**"adc" 不存在长度为 3 的回文子序列。
 
## 示例 3:

>**输入:** s = "bbcbaba"<br>
>**输出:** 4<br>
>**解释：** 长度为 3 的 4 个回文子序列分别是：
> - "bbb" ("bbcbaba" 的子序列)
> - "bcb" ("bbcbaba" 的子序列)
> - "bab" ("bbcbaba" 的子序列)
> - "aba" ("bbcbaba" 的子序列)

## 提示:
- `3 <= s.length <= 105`
- `s 仅由小写英文字母组成`

## 思路：
> 长度为3 ,从`a`-`z`枚举左端点，枚举左端点等同于枚举右端点，然后从两个端点中间枚举其他字母累加求和

## 代码：
```Java
class Solution {
    public int countPalindromicSubsequence(String s) {
        int res = 0;
        for (char point = 'a'; point <= 'z'; point++) {
            int L = s.indexOf(point);
            int R = s.lastIndexOf(point);
            if (L != R) {
                boolean[] set = new boolean[26];
                for (int M = L + 1; M < R; M++) {
                    int index = s.charAt(M) - 'a';
                    if (set[index])continue;
                    set[index] = true;
                    res++;
                }
            }
        }
        return res;

    }
}
```
