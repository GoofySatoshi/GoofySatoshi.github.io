---
title: LeetCode1262.可被三整除的最大和
date: 2025-11-23 22:29:00
tags:
  - LeetCode
  - 算法
  - 数组
  - 贪心
categories:
  - LeetCode
  - 算法
---

# [1262.可被三整除的最大和](https://leetcode.cn/problems/greatest-sum-divisible-by-three/description/)
## 题目描述：
给你一个整数数组 `nums`，请你找出并返回能被三整除的元素 最大和。<br>

## 示例 1:
>**输入:** nums = [3,6,5,1,8] <br>
>**输出:** 18 <br>
>**解释:** 选出数字 3, 6, 1 和 8，它们的和是 18（可被 3 整除的最大和）。<br>

## 示例 2:

>**输入:** nums = [4]<br>
>**输出:** 0<br>
>**解释：** 4 不能被 3 整除，所以无法选出数字，返回 0。
 
## 示例 3:

>**输入:** nums = [1,2,3,4,4]<br>
>**输出:** 12<br>
>**解释：** 选出数字 1, 3, 4 以及 4，它们的和是 12（可被 3 整除的最大和）。<br>

## 提示:
- `1 <= nums.length <= 4 * 104`
- `1 <= nums[i] <= 10^4`

## 思路：
> 虽然A过了，但是我写的贪心太丑陋了，放下灵神的代码吧

## 代码：
```Java
class Solution {
    public int maxSumDivThree(int[] nums) {
        int sum = 0;
        var a1 = new ArrayList<Integer>();
        var a2 = new ArrayList<Integer>();
        for (int i = 0; i < nums.length; i++) {
            sum += nums[i];
            if (nums[i] % 3 == 1)
                a1.add(nums[i]);
            else if (nums[i] % 3 == 2)
                a2.add(nums[i]);
        }
        if (sum % 3 == 0) return sum;
        Collections.sort(a1);
        Collections.sort(a2);
        if (sum % 3 == 2) {
            var tmp = a1;
            a1 = a2;
            a2 = tmp;
        }
        int res = 0;
        if (!a1.isEmpty()) {
            res = sum - a1.get(0);
        }
        if (a2.size() >= 2) {
            res = Math.max(res, sum - a2.get(0) - a2.get(1));
        }
        return res;
    }
}
```
