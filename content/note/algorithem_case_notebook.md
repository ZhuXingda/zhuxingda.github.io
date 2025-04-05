---
title: 【leetcode】一些常见传统算法的 Golang 实现
date: 2024-10-12
tags:
  - "algorithm"
  - "leetcode"
categories:
  - "Algorithem"
---
记录一些 leetcode 上常见算法的 Golang 实现。
<!--more-->
## 数组
#### 快排
1. 取数组第一位作为 pivot
2. 从左右向中间遍历数组，右边找到一个比 pivot 小的数时，和 pivot swap，左边找到一个比 pivot 大的数时，和 pivot swap
3. 直到左右相遇结束 swap，从而把数组分成两个 partition，左边的 partition 都小于等于 pivot，右边的 partition 都大于等于 pivot
4. 递归对两个 partition 进行同样的操作，直到 partition 为空
```go
func quickSort(arr []int) {
	sort(arr, 0, len(arr)-1)
}

func sort(arr []int, left, right int) {
	if left >= right {
		return
	}
	p := left
	i, j := left, right
	for i < j {
		for arr[p] <= arr[j] && i < j {
			j--
		}
		if i < j && arr[p] > arr[j] {
			swap(arr, i, j)
			p = j
		}
		for arr[p] >= arr[i] && i < j {
			i++
		}
		if i < j && arr[p] < arr[i] {
			swap(arr, i, j)
			p = i
		}
	}
	sort(arr, left, p-1)
	sort(arr, p+1, right)
}

func swap(arr []int, i, j int) {
	tmp := arr[i]
	arr[i] = arr[j]
	arr[j] = tmp
}
```
## 树
#### 大顶堆/小顶堆
#### 红黑树
## 动态规划