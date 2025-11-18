---
title: LeetCode717. 1 比特与 2 比特字符
date: 2025-05-04 03:30:00
tags:
  - LeetCode
  - 算法
  - 数组
categories:
  - LeetCode
  - 算法
---

# [717. 1 比特与 2 比特字符](https://leetcode.cn/problems/1-bit-and-2-bit-characters/description/)
## 题目描述：
有两种特殊字符：

- 第一种字符可以用一比特 0 表示<br>
- 第二种字符可以用两比特（10 或 11）表示<br>
给你一个以 0 结尾的二进制数组 bits ，如果最后一个字符必须是一个一比特字符，则返回 `true` 。
 

## 示例 1:
>**输入:** bits = [1, 0, 0] <br>
>**输出:** true <br>
>**解释:** 唯一的解码方式是将其解析为一个两比特字符和一个一比特字符。所以最后一个字符是一比特字符。

## 示例 2:

>**输入:** bits = [1,1,1,0]<br>
>**输出:** false<br>
>**解释：** 唯一的解码方式是将其解析为两比特字符和两比特字符。所以最后一个字符不是一比特字符。
 

## 提示:
- `1 <= bits.length <= 1000`
- `bits[i] 为 0 或 1`

## 思路：
> 思维题,倒序判断最后一段1奇偶性,最后一个0被占用了，一旦构成01就是不合法的<br>
> 最后一段1为奇数则一定不合法<br>
> 最后一段1为偶数则一定合法<br>
> 这个可以枚举验证一下

## 代码：
```Java
class Solution {
    public boolean isOneBitCharacter(int[] bits) {
        int n = bits.length;
        if (bits[n - 1] == 1)
            return false;
        int r = bits.length - 2;
        int cnt = 0;
        while (r > -1 && bits[r] == 1) {
            cnt++;
            r--;
        }
        return cnt % 2 == 0;
    }
}
```
