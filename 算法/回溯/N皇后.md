

# 问题

*n* 皇后问题研究的是如何将 *n* 个皇后放置在 *n*×*n* 的棋盘上，并且使皇后彼此之间不能相互攻击。

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/10/12/8-queens.png)

上图为 8 皇后问题的一种解法。

给定一个整数 *n*，返回所有不同的 *n* 皇后问题的解决方案。

每一种解法包含一个明确的 *n* 皇后问题的棋子放置方案，该方案中 `'Q'` 和 `'.'` 分别代表了皇后和空位。

**示例：**

```txt
输入：4
输出：[
 [".Q..",  // 解法 1
  "...Q",
  "Q...",
  "..Q."],

 ["..Q.",  // 解法 2
  "Q...",
  "...Q",
  ".Q.."]
]
解释: 4 皇后问题存在两个不同的解法。
```

**提示：**

- 皇后彼此不能相互攻击，也就是说：任何两个皇后都不能处于同一条横行、纵行或斜线上。

# 题解



使用 DFS 尝试所有可能性然后进行剪枝，对棋盘每个位置都进行搜索查看是否可以放下一个皇后

直接套用回溯套路框架

```go
result = []
func backtrack(路径，选择列表) {
	if 满足结束条件 {
		result.add(路径)
	}
	return

	for 选择 in 选择列表 {
		做选择
		backtrack(路径，选择列表)
		撤销选择
	}
}
```

```go
package backtrack
// 全局声明返回值
var result [][]string

func solveNQueen(n int) [][]string {
	result = [][]string{}
	board := make([][]bool, n)
	for i := 0; i < n; i++ {
		board[i] = make([]bool, n)
	}
	NQueenBacktrack(board, [][]byte{})
	return result
}

// NQueenBacktrack 回溯剪枝遍历所有情况
func NQueenBacktrack(board [][]bool, path [][]byte) {
    // 到达终点将其可能行加入到 result 中
	if len(path) == len(board) {
		t := make([]string, len(path))
		for k, bs := range path {
			t[k] = string(bs)
		}
		result = append(result, t)
	}

	for i := 0; i < len(board); i++ {
        // 剪枝
		if !isValid(board, len(path), i) {
			continue
		}
        // 初始化棋盘
		bs := make([]byte, len(board))
		for j := 0; j < len(board); j++ {
			bs[j] = '.'
		}
        // 标记并回溯搜索
		bs[i] = 'Q'
		board[len(path)][i] = true
		path = append(path, bs)
		NQueenBacktrack(board, path)
		path = path[:len(path)-1]
		board[len(path)][i] = false
	}
}

// isValid 检测能否在 board[row][col] 放置皇后
// 皇后不可以同时在上下左右对角线存在
func isValid(board [][]bool, row, col int) bool {
	// 检测行是否有皇后冲突
	for i := 0; i < row; i++ {
		if board[i][col] == true {
			return false
		}
	}
	// 检测列是否有皇后冲突
	for j := 0; j < len(board); j++ {
		if board[row][j] == true {
			return false
		}
	}
	// 检测左上角
	for i, j := row, col; i >= 0 && j >= 0; i, j = i-1, j-1 {
		if board[i][j] == true {
			return false
		}
	}
	// 检测右上角
	for i, j := row, col; i >= 0 && j < len(board); i, j = i-1, j+1 {
		if board[i][j] == true {
			return false
		}
	}
	return true
}

```



