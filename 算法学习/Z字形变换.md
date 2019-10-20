# Z字形变换

将一个给定字符串根据给定的行数，以从上往下、从左到右进行 Z 字形排列。

比如输入字符串为 `"LEETCODEISHIRING"` 行数为 3 时，排列如下：

```
L   C   I   R
E T O E S I I G
E   D   H   N
```

之后，你的输出需要从左往右逐行读取，产生出一个新的字符串，比如：`"LCIRETOESIIGEDHN"`。

请你实现这个将字符串进行指定行数变换的函数：

```
string convert(string s, int numRows);
```

**示例 1:**

```
输入: s = "LEETCODEISHIRING", numRows = 3
输出: "LCIRETOESIIGEDHN"
```

**示例 2:**

```
输入: s = "LEETCODEISHIRING", numRows = 4
输出: "LDREOEIIECIHNTSG"
解释:

L     D     R
E   O E   I I
E C   I H   N
T     S     G
```



## 解

在输入了参数5后可得

```
A   I   Q   Y
B  HJ  PR  XZ
C G K O S W
DF  LN  TV
E   M   U
```

得出各行字符在源字符串得索引号为

1. 0行，0, 8, 16, 24
2. 1行，1, 7,9, 15,17, 23,25
3. 2行，2, 6, 10, 14, 18, 22
4. 3行，3,5, 11,13, 19,21
5. 4行，4, 12, 20

令p=numRows×2-2，可以总结出以下规律

1. 0行， 0×p，1×p，...
2. r行， r，1×p-r，1×p+r，2×p-r，2×p+r，...
3. 最后一行， numRow-1, numRow-1+1×p，numRow-1+2×p，...

只需编程依次处理各行即可。

```go
/*
 * @lc app=leetcode.cn id=6 lang=golang
 *
 * [6] Z 字形变换
 */
func convert(s string, numRows int) string {
	if numRows == 1 || len(s) <= numRows {
		return s
	}

	res := bytes.Buffer{}
	p := numRows*2 - 2
	//第一行
	for i := 0; i < len(s); i += p {
		res.WriteByte(s[i])
	}

	for r := 1; r <= numRows-2; r++ {
		res.WriteByte(s[r])

		for k := p; k-r < len(s); k += p {
			res.WriteByte(s[k-r])
			if k+r < len(s) {
				res.WriteByte(s[k+r])
			}
		}
	}

	for i := numRows - 1; i < len(s); i += p {
		res.WriteByte(s[i])
	}

	return res.String()
}
```

 