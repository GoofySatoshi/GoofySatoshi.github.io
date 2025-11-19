---
title: LeetCode757.设置交集大小至少为2
date: 2025-11-19 00:56:00
tags:
  - LeetCode
  - 算法
  - 数组
  - 贪心
  - 排序
categories:
  - LeetCode
  - 算法
---

# [757.设置交集大小至少为2](https://leetcode.cn/problems/set-intersection-size-at-least-two/description/)
## 题目描述：
给你一个二维整数数组 `intervals` ，其中 `intervals[i] = [starti, endi]` 表示从 `starti` 到 `endi` 的所有整数，包括 `starti` 和 `endi` 。

包含集合 是一个名为 `nums` 的数组，并满足 `intervals` 中的每个区间都 至少 有 两个 整数在 `nums` 中。

- 例如，如果 `intervals = [[1,3], [3,7], [8,9]]` ，那么 `[1,2,4,7,8,9]` 和 `[2,3,4,8,9]` 都符合 包含集合 的定义。<br>
 返回包含集合可能的最小大小。

## 示例 1:
>**输入:** intervals = [[1,3],[3,7],[8,9]] <br>
>**输出:** 5 <br>
>**解释:** nums = [2, 3, 4, 8, 9].
>可以证明不存在元素数量为 4 的包含集合。

## 示例 2:

>**输入:** intervals = [[1,3],[1,4],[2,5],[3,5]]<br>
>**输出:** 3<br>
>**解释：** nums = [2, 3, 4].
>可以证明不存在元素数量为 2 的包含集合。 
 
## 示例 3:

>**输入:** intervals = [[1,2],[2,3],[2,4],[4,5]]<br>
>**输出:** 5<br>
>**解释：** nums = [1, 2, 3, 4, 5].
>可以证明不存在元素数量为 4 的包含集合。 


## 提示:
- `1 <= intervals.length <= 3000`
- `intervals[i].length == 2`
- `0 <= starti < endi <= 108`

## 思路：
> 排序贪心，先将数组排序 排序优先级 右区间端点增序，左区间端点降序<br>
> 枚举状态，累加求个和返回结果<br>
> 最难理解的应该是排序的思路了，按 “右端点升序，右端点相同时左端点降序” 排序，本质是为了让选择的元素尽可能覆盖更多后续区间

## 代码：
```Java
class Solution {
    public int intersectionSizeTwo(int[][] intervals) {
        Arrays.sort(intervals,
                (u, v) -> {
                    if (u[1] != v[1]) {
                        return u[1] - v[1]; // 右端点升序
                    } else {
                        return v[0] - u[0]; // 右端点相同则左端点降序
                    }
                });
        int l = intervals[0][1] - 1, r = intervals[0][1], res = 2;
        for (int i = 0; i < intervals.length; i++) {
            int L = intervals[i][0];
            int R = intervals[i][1];
            if (l >= L && r <= R) {
                continue;
            } else if (r < L) {
                l = R - 1;
                r = R;
                res += 2;
            } else if ((r == L) || (r <= R && l < L)) {
                l = r;
                r = R;
                res++;
            }
        }
        return res;
    }
}
```
