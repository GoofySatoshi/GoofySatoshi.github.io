---
title: LeetCode32.最长有效括号
date: 2025-11-23 00:29:00
tags:
  - LeetCode
  - 算法
  - 字符串
  - 栈
categories:
  - LeetCode
  - 算法
---

# [LeetCode32.最长有效括号](https://leetcode.cn/problems/longest-valid-parentheses/description/)
## 题目描述：
给你一个只包含 `(` 和 `)` 的字符串，找出最长有效（格式正确且连续）括号 子串 的长度。<br>

左右括号匹配，即每个左括号都有对应的右括号将其闭合的字符串是格式正确的，比如 `(()())`。<br>



## 示例 1:
>**输入:** s = "(()" <br>
>**输出:** 2 <br>
>**解释:** 最长有效括号子串是 "()"<br>

## 示例 2:

>**输入:** s = ")()())"<br>
>**输出:** 4<br>
>**解释：**最长有效括号子串是 "()()"
 
## 示例 3:

>**输入:** s = ""<br>
>**输出:** 0<br>


## 提示:
- `0 <= s.length <= 3 * 104`
- `s[i]` 为 `(` 或 `)`

## 思路：
> 栈存下标，每次入栈取栈顶元素看当前待入栈元素是否为`)`,且栈顶元素为`(`，若满足条件则先弹出元素，然后取`ans = Math.max(i-stack(top),ans)`
> 需要注意下边界条件，即栈为空边界

## 代码：
```Java
class Solution {
    public int longestValidParentheses(String s) {
        int ans = 0;
        Deque<Integer> stack = new ArrayDeque<>();
        for (int i = 0; i < s.length(); i++) {
            if (!stack.isEmpty() && s.charAt(i) == ')' && s.charAt(stack.peekFirst()) == '(') {
                stack.pop();
                if (stack.isEmpty()) {
                    ans = i + 1;
                } else {
                    Integer index = stack.peekFirst();
                    ans = Math.max(i - index, ans);
                }
                continue;
            }
            stack.push(i);
        }
        return ans;
    }
}
```
