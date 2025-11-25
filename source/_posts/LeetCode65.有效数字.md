---
title: LeetCode65.有效数字
date: 2025-11-23 01:30:00
tags:
  - LeetCode
  - 算法
  - 字符串
categories:
  - LeetCode
  - 算法
---

# [65.有效数字](https://leetcode.cn/problems/valid-number/description/)
## 题目描述：
给定一个字符串 `s` ，返回 `s` 是否是一个 有效数字。<br>

例如，下面的都是有效数字：`2`, `0089`, `-0.1`, `+3.14`, `4.`, `-.9`, `2e10`, `-90E3`, `3e+7`, `+6e-1`, `53.5e93`, `-123.456e789`，而接下来的不是：`abc`, `1a`, `1e`, `e3`, `99e2.5`, `--6`, `-+3`, `95a54e53`。<br>

一般的，一个 有效数字 可以用以下的规则之一定义：<br>

1. 一个 整数 后面跟着一个 可选指数。<br>
2. 一个 十进制数 后面跟着一个 可选指数。<br>

一个 整数 定义为一个 可选符号 `-` 或 `+` 后面跟着 数字。<br>
一个 十进制数 定义为一个 可选符号 `-` 或 `+` 后面跟着下述规则：<br>

1. 数字 后跟着一个 小数点 `.`。<br>
2. 数字 后跟着一个 小数点 `.` 再跟着 数位。<br>
3. 一个 小数点 `. `后跟着 数位。<br>
指数 定义为指数符号 `e` 或 `E`，后面跟着一个 整数。<br>

数字 定义为一个或多个数位。<br>
## 示例 1:
>**输入:** s = "0" <br>
>**输出:** true <br>

## 示例 2:

>**输入:** s = "e"<br>
>**输出:** false<br>
 
## 示例 3:

>**输入:** s = ""<br>
>**输出:** false<br>


## 提示:
- `1 <= s.length <= 20`
- `s` 仅含英文字母（大写和小写），数字（`0-9`），加号 `+` ，减号 `-` ，或者点 `.` 。

## 思路：
> 模式匹配,需要细心点，不然容易WA,写出三种数字判断函数，从e||E做拆分，带入函数得出结果

## 代码：
```Java
class Solution {
    // 判断是否是整数
    public boolean isInteger(String s, int l, int r) {
        char ch = s.charAt(l++);
        if (ch != '-' && ch != '+' && (ch < '0' || ch > '9')) {
            return false;
        }
        int numCnt = 0;
        if (ch >= '0' && ch <= '9') {
            numCnt++;
        }
        while (l <= r) {
            char item = s.charAt(l++);
            if (item < '0' || item > '9') {
                return false;
            }
            numCnt++;
        }
        return numCnt != 0;
    }

    // 是否为十进制数
    public boolean isDecimal(String s, int l, int r) {
        char ch = s.charAt(l++);
        if (ch != '-' && ch != '+' && ch != '.' && (ch < '0' || ch > '9')) {
            return false;
        }
        int pointCnt = 0, numCnt = 0;
        if (ch == '.') {
            pointCnt++;
        }
        if (ch >= '0' && ch <= '9') {
            numCnt++;
        }
        if (l > r) {
            return false;
        }
        while (l <= r) {
            char item = s.charAt(l++);
            if (item == '.') {
                if (pointCnt == 1)
                    return false;
                else
                    pointCnt++;
                continue;
            }
            if (item == 'e' || item == 'E') {
                return numCnt != 0 && isIndex(s, l - 1, r);
            }
            if (item < '0' || item > '9') {
                return false;
            } else {
                numCnt++;
            }
        }
        return numCnt != 0;
    }

    // 是否为指数
    public boolean isIndex(String s, int l, int r) {
        char ch = s.charAt(l++);
        if (ch != 'e' && ch != 'E') {
            return false;
        }
        if (l > r) {
            return false;
        }
        return isInteger(s, l, r);
    }

    public boolean isNumber(String s) {
        int r = s.length() - 1;
        int e = Math.max(s.indexOf('e'), s.indexOf('E'));
        if (e == -1) {
            return isInteger(s, 0, r) || isDecimal(s, 0, r);
        } else {
            boolean index = isIndex(s, e, r);
            return (isInteger(s, 0, e - 1) || isDecimal(s, 0, e - 1)) && index;
        }
    }
}
```
