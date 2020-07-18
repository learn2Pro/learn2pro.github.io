---
title: 什么是马拉车(manacher's)算法
date: 2020-07-18 09:06:41
tags:
- Algorithm
categories:
- Technology
---

## 背景
在计算机世界里，有一类问题叫寻找回文子串，英文叫`Palindrome`，这个问题在很长一段时间，人们都没找到比较高效的算法
，时间复杂度都在`O(n^2)`，下面来看看什么是回问字符串：
![回文串](/image/manacher/palindrome_substr.svg)
可以看到整个字符串是对称的，将`sub`反转过来还是等于它自身，字符串的part0和part1是对称的，基于这个特点，可以写出
一个`O(n^2)`的算法；
## Intuition
直接比较前后部分，但是要判断奇偶的情况
```python
def findPalindromeNum(s:str)->int:
    if not s : return 0
    size,ans = len(s),0
    def helper(start,end):
        while start>=0 and end<size and s[start]==s[end]:
            ans += 1
            start -= 1
            end += 1
    for i in range(size):
        helper(i,i)
    for i in range(size):
        helper(i,i+1)
    return ans
```
这个算法有两层循环，外层是O(n),内层最坏情况也是O(n),所以整个算法复杂度是O(n^2),里面一个很大的问题，我们进行了很多
重复的比较，这时候就要引入manacher's算法了，他能够复用我以前比较的结果
## manacher's
* 一开始想这个问题我也试过动态规划的思想，我的想法是利用二位数组dp[i][j] = dp[i+1][j-1] + 2 if str[i] == str[j]；利用以前
判断过的回文串来迭代，但是整个算法复杂度还是O(n^2),因为我dp[i][j]无法利用dp[i][j-1]的信息，manacher用了另外一个思路来解决这个问题；
* 算法中有几个变量需要定义:
  * center,当前最长回文子串的中心坐标index
  * right,当前最长回文子串的右边界坐标index
  * radius数组,记录字符串每一个位置的最长回文子串半径
  * mirror_i,我当前位置处于i，以center为中心，我的对称位置为mirror_i=2*center-i

![马拉车](/image/manacher/manacher_alg.svg)
一句话解释：当我要计算i位置的回文子串时，我会去看mirror_i位置的回文子串长度，因为mirror_i的回文子串会根据center对称
到i位置，所以mirror_i半径以内的字符都是相等的，迭代时会跳过这一部分；这里有一个问题是需要考虑边界right，因为如果mirror_i
的半径超过了mirror_i,我们只能起始点卡在right，因为如果超过right部分是对称的，当前center的半径肯定不是right这个位置，算法
实现如下：
```python
def findPalindromeNum(s:str)->int:
    def manachers(input):
        # 屏蔽奇偶问题
        input = '!#'+'#'.join(input)+'#$'
        center = right = 0
        radius = [0] * len(input)
        for i in range(len(input)):
            if i < right:
                # 获取当前要进行迭代的起始半径
                radius[i] = min(radius[2*center - i],right - i)
            # 迭代进行比较
            while input[i-radius[i]-1] == input[i+radius[i]+1]:
                radius[i] += 1
            # 如果当前回文子串边界大于右边界，则更新
            if i+radius[i] > right:
                center,right = i,i + radius[i]
        return sum((size+1)//2 for size in radius)
    return manachers(s)
```
因为算法实现不是每次都跳到0开始搜索，而是跳到了mirror的radius上开始搜索，可以证明整个算法时间复杂度是O(n)的；


## Reference
* [manancher's algorithm](https://dl.acm.org/doi/10.1145/321892.321896)