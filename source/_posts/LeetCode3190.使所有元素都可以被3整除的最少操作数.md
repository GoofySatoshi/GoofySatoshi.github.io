---
title: LeetCode3190.使所有元素都可以被3整除的最少操作数
date: 2025-11-22 23:44:00
tags:
  - LeetCode
  - 算法
  - 数组
categories:
  - LeetCode
  - 算法
---

# [3190.使所有元素都可以被3整除的最少操作数](https://leetcode.cn/problems/find-minimum-operations-to-make-all-elements-divisible-by-three/description/)
## 题目描述：
给你一个整数数组 `nums` 。一次操作中，你可以将 `nums` 中的 任意 一个元素增加或者减少 `1` 。

请你返回将 `nums` 中所有元素都可以被 `3` 整除的 最少 操作次数。

## 示例 1:
>**输入:** nums = [1,2,3,4] <br>
>**输出:** 3 <br>
>**解释:** 通过以下 3 个操作，数组中的所有元素都可以被 3 整除：<br>
> - 将 1 减少 1 。
> - 将 2 增加 1 。
> - 将 4 减少 1 。

## 示例 2:

>**输入:** nums = [3,6,9]<br>
>**输出:** 0<br>

 


## 提示:
- `1 <= nums.length <= 50`
- `1 <= nums[i] <= 50`

## 思路：
> 暴力破解就好，循环代替思考

## 代码：
```Java
class Solution {
    public int minimumOperations(int[] nums) {
        return Arrays.stream(nums)
                .map(num -> {
                    return num % 3 == 0 ? 0 : 1;
                })
                .sum();
    }
}
```
