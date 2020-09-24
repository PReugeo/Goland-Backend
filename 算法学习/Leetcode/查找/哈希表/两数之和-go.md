# 两数之和

给定 nums = [2, 7, 11, 15], target = 9
因为 nums[0] + nums[1] = 2 + 7 = 9
所以返回 [0, 1]


## go语言解法
```go
func twoSum(nums []int, target int) []int {
    m := make(map[int]int)
    var result []int
    for i := 0; i < len(nums); i++ {
        value := target - nums[i]
        if _, ok := m[value]; ok {
            return []int{m[value], i}
        }
        m[nums[i]] = i
    }
    return result
}
```
时间复杂度O(n)

## 分析:
   遍历 nums 将nums中数字存入HashMap的key中其数组下标存入value中,继续遍历若找到HashMap中有与nums[i]相加为target值的数则返回map中存入的i与当前i

