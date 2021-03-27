package main

import "fmt"

func lengthOfLongestSubstring(s string) int {
	fmt.Println(s)
	// 哈希集合，记录每个字符是否出现过
	m := map[string]int{}
	n := len(s)
	// 右指针，初始值为 -1，相当于我们在字符串的左边界的左侧，还没有开始移动
	rk, ans := -1, 0
	for i := 0; i < n; i++ {
		if rk+1 < n {
			fmt.Println("左指针处:", string(s[i]), "--右指针处s[rk+1]:", string(s[rk+1]), "滑动窗口所在:", string(s[i:rk+1]), "--m:", m)
		}
		if i > 0 {
			// 左指针向右移动一格，移除一个字符
			delete(m, string(s[i-1]))
		}
		for rk+1 < n && m[string(s[rk+1])] == 0 {
			// 不断地移动右指针
			m[string(s[rk+1])]++
			rk++
			fmt.Println("右移右指针后rk+1:", rk+1, "滑动窗口所在:", string(s[i:rk+1]), "--m:", m)
		}
		// 第 i 到 rk 个字符是一个极长的无重复字符子串
		ans = max(ans, rk-i+1)
	}
	return ans
}

func max(x, y int) int {
	if x < y {
		return y
	}
	return x
}

func main() {
	len := lengthOfLongestSubstring("abcabcbb")
	fmt.Println("len:", len)
}
