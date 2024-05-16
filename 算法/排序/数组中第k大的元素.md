# 题目描述

在未排序的数组中找到第 k 个最大的元素。请注意，你需要找的是数组排序后的第 k 个最大的元素，而不是第 k 个不同的元素。

示例 1:

>   输入: [3,2,1,5,6,4] 和 k = 2
>   输出: 5

示例 2:

>   输入: [3,2,3,1,2,4,5,5,6] 和 k = 4
>   输出: 4

说明：

你可以假设 k 总是有效的，且 1 ≤ k ≤ 数组的长度。

# 解题思路

## 1.快排切分思路

快排的 partition 操作，会使得，该标定点（privot）呆在排序后的最终位置，即：

*   对于某个索引 `j`，`nums[j]` 已经排定，即 `nums[j]` 经过 partition（切分）操作以后会放置在它 “最终应该放置的地方”；
*   `nums[left]` 到 `nums[j - 1]` 中的所有元素都不大于 `nums[j]`；
*   `nums[j + 1]` 到 `nums[right]` 中的所有元素都不小于 `nums[j]`。

通过切分操作，每次都能缩小搜索范围，切分过程可以不借助额外的数组空间，仅通过交换数组元素实现。

```go
func findKthLargest(nums []int, k int) int {
    pos := partition(nums, 0, len(nums) - 1)

    if pos < len(nums) - k {
        return findKthLargest(nums[pos + 1:], k)
    }

    if pos > len(nums) - k {
        return findKthLargest(nums[:pos], k - (len(nums) - pos))
    }

    return nums[pos]
}

// partition 操作
func partition(nums []int, start, end int) int {
    // 随机初始化第一个元素，防止递归树过深 
    if end > start {
        randomIndex := start + 1 + rand.Intn(end - start)
        nums[start], nums[randomIndex] = nums[randomIndex], nums[start]
    }

    pos, l, r := nums[start], start, end

    for l < r {
        // 双指针分别从左右扫描，将大于 pos 的值放在 pos 右边，小于 pos 的值放在左边
        for nums[r] >= pos && l < r {
            r--
        }
        for nums[l] <= pos && l < r {
            l++
        }
        
        if l > r {
            break
        }
        
        nums[l], nums[r] = nums[r], nums[l]
    }
	// 将 pos 值放在有序后的位置上
    nums[start], nums[l] = nums[l], nums[start]

    return l
}   
```

## 2. 堆排序思路

题目要求寻找第 `K` 大元素，其实就是排序后，后半部分最小的元素，因此，可以维护一个有 `K` 个元素的最小堆：

1.  如果堆不满，直接添加元素
2.  堆满时，若新读到的树小于等于堆顶就不是我们的需要的元素，只有新元素大于堆顶时，才将堆顶拿出，然后放入新读到的数，然后用堆自己去调整内部结构

假设数组有 `len` 个元素。

思路1：把 `len` 个元素都放入一个最小堆中，然后再 `pop()` 出 `len - k` 个元素，此时最小堆只剩下 `k` 个元素，堆顶元素就是数组中的第 `k` 个最大元素。

思路2：把 `len` 个元素都放入一个最大堆中，然后再 pop() 出 `k - 1` 个元素，因为前 `k - 1` 大的元素都被弹出了，此时最大堆的堆顶元素就是数组中的第 `k` 个最大元素。



```go
func findKthLargest(nums []int, k int) int {
    nums = heapSort(nums)
    return nums[k - 1]
}

func heapSort(nums []int) []int {
    lens := len(nums)
    // 建堆， lens/2 后面都是叶子节点，不需要调整 down()
    for i := lens / 2; i >= 0; i-- {
        down(nums, i, lens)
    }

    // 将小根堆堆顶排到切片末尾（降序）
    for i := lens - 1; i >= 0; i-- {
        nums[0], nums[i] = nums[i], nums[0]
        lens--
        down(nums, 0, lens)
    }

    return nums
}   

// 小根堆
func down(nums []int, i, lens int) {
    // i 父节点
    min := i
    // 左右孩子节点
    l, r := 2 * i + 1, 2 * i + 2

    if l < lens && nums[l] < nums[min] {
        min = l
    }
    if r < lens && nums[r] < nums[min] {
        min = r
    }
    
    // 若最小值不是父节点了，那么把最小的节点与父节点交换
    // 变换后可能会影响到该父节点的子节点的子节点，所以还需要继续往下对比
    if min != i {
        nums[min], nums[i] = nums[i], nums[min]
        down(nums, min, lens)
    }

}
```

