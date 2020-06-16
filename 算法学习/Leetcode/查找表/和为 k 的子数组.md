# 题目描述

给定一个整数数组和一个整数 **k，**你需要找到该数组中和为 **k** 的连续的子数组的个数。

**示例 1 :**

```
输入:nums = [1,1,1], k = 2
输出: 2 , [1,1] 与 [1,1] 为两种不同的情况。
```

**说明 :**

1.  数组的长度为 [1, 20,000]。
2.  数组中元素的范围是 [-1000, 1000] ，且整数 **k** 的范围是 [-1e7, 1e7]。





# 解题思路

## 枚举

首先我们可以使用枚举的方法来做，以 `i` 为结尾和为 `k` 的数，我们只需要统计符合条件的下标 `j` 的数量，其中 `0<=j<=i` 且 `[j..i]` 子数组和刚好为 `k`

代码

```go
func subarraySum(nums []int, k int) int {
    count := 0
    for i := 0; i < len(nums); i++ {
        sum := 0
        for j := j; j >= 0; j-- {
            sum += nums[j]
            if sum == k {
                count++
            }
        }
    }
    return count
}
```

时间复杂度 $O(n^2)$

空间复杂度 $O(1)$

## 优化（前缀和➕哈希表）

通过方法一可以引入前缀和的概念，通过数据结构优化将复杂度降低到 $O(n)$。

定义前缀和：

*   从 `0` 项到当前项的总和
*   即 `pre[i] = pre[i-1]+nums[i]=nums[i]+num[i-1]+...+nums[0]`
*   因为 `nums[i] = pre[i]-pre[i-1]`
*   所以有 `nums` 数组 `[j..i]` 项的和为 `pre[i] - pre[j - 1]`

所以从 `[j..i]` 这个子数组和为 `k` 可以等于
$$
pre[i]-pre[j-1]==k
$$
找到下标 `j` 只需满足
$$
pre[j-1]==pre[i]-k
$$
我们可以定义以下哈希表

*   键：前缀和
*   值：该前缀和出现了几次

接着我们遍历数组

1.  求当前下标的前缀和，如果未存入，则存入，初始值为 1
2.  如果存入，则值 加 1
3.  如果发现 map 中存在当前键值对的键`key` 为当前前缀和 - k
4.  则将 `之前求出的前缀和` 出现次数，累加给 count 计数器





代码

```go
func subarraySum(nums []int, k int) int {
    count := 0
    sumMap := make(map[int]int, 0)
    sum := 0
    sumMap[0] = 1
    for _,  v := range nums {
        sum += v
        if sumMap[sum - k] > 0 {
            count += sumMap[sum - k]
        }
        sumMap[sum]++
    }
    return count
}
```

