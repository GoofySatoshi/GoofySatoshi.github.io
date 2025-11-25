---
title: LeetCode1018.可被5整除的二进制
date: 2025-11-24 22:56:00
tags:
  - LeetCode
  - 算法
  - 位运算
  - 数组
categories:
  - LeetCode
  - 算法
---

# [1018.可被5整除的二进制](https://leetcode.cn/problems/binary-prefix-divisible-by-5/description/)
## 题目描述：
给定一个二进制数组 `nums` ( 索引从`0`开始 )。<br>

我们将`xi` 定义为其二进制表示形式为子数组 `nums[0..i]` (从最高有效位到最低有效位)。<br>

例如，如果 `nums =[1,0,1]` ，那么 `x0 = 1`, `x1 = 2`, 和 `x2 = 5`。<br>
返回布尔值列表 `answer`，只有当 `xi` 可以被 `5` 整除时，答案 `answer[i]` 为 `true`，否则为 `false`。<br>

## 示例 1:
>**输入:** nums = [0,1,1] <br>
>**输出:** [true,false,false] <br>
>**解释:** 输入数字为 0, 01, 011；也就是十进制中的 0, 1, 3 。只有第一个数可以被 5 整除，因此 answer[0] 为 true 。<br>


## 示例 2:

>**输入:** nums = [1,1,1]<br>
>**输出:** [false,false,false]<br>
 
## 提示:
- `1 <= nums.length <= 105`
- `nums[i]` 仅为 `0` 或 `1`

## 思路：
> 循环遍历左移一位取余`5`，为了避免溢出对5取余

## 代码：
```Java
class Solution {
    public List<Boolean> prefixesDivBy5(int[] nums) {
        int n = nums.length, cnt = 0;
        List<Boolean> res = new ArrayList<>(n);
        for (int i = 0; i < n; i++) {
            cnt = ((cnt << 1) + nums[i]) % 5;
            res.add(cnt == 0);
        }
        return res;
    }
}
```
