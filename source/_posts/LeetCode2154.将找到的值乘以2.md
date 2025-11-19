---
title: LeetCode2154. 将找到的值乘以2
date: 2025-11-19 17:15:00
tags:
  - LeetCode
  - 算法
  - 数组
  - 哈希表
  - 排序
categories:
  - LeetCode
  - 算法
---

# [2154. 将找到的值乘以2](https://leetcode.cn/problems/keep-multiplying-found-values-by-two/description/)
## 题目描述：
- 给你一个整数数组 `nums` ，另给你一个整数 `original` ，这是需要在 `nums` 中搜索的第一个数字。

- 接下来，你需要按下述步骤操作：

1. 如果在 `nums` 中找到 `original` ，将 `original` 乘以 `2` ，得到新 `original`（即，令 `original = 2 * original`）。
否则，停止这一过程。
2. 只要能在数组中找到新 `original` ，就对新 `original` 继续 重复 这一过程。
3. 返回 `original` 的 最终 值。
 

## 示例 1:
>**输入:** nums = [5,3,6,1,12], original = 3 <br>
>**输出:** 24 <br>
>**解释:**
>- 3 能在 nums 中找到。3 * 2 = 6 。
>- 6 能在 nums 中找到。6 * 2 = 12 。
>- 12 能在 nums 中找到。12 * 2 = 24 。
>- 24 不能在 nums 中找到。因此，返回 24 。

## 示例 2:

>**输入:** nums = [2,7,9], original = 4 <br>
>**输出:** 4 <br>
>**解释：**
>- 4 不能在 nums 中找到。因此，返回 4 。

 

## 提示:
- `1 <= nums.length <= 1000`
- `1 <= nums[i], original <= 1000`

## 思路：
> 解法一：排序+顺序遍历 <br>
> 解法二：HashSet + if: set::contains(original) -> original*=2;
## 代码：
```Java
class Solution {
    public int findFinalValue(int[] nums, int original) {
        Arrays.sort(nums);
        for(int i=0;i<nums.length;i++){
            if(nums[i]==original){
                original*=2;
            }
        }
        return original;
    }
}
```

```Java
class Solution {
    public int findFinalValue(int[] nums, int original) {
        Set<Integer> set = IntStream.of(nums).boxed().collect(Collectors.toSet());
        while (set.contains(original)) {
            original *= 2;
        }
        return original;
    }
}
```
