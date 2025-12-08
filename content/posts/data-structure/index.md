---
title: 数据结构与算法

date: 2020-06-11T14:31:00+08:00

tag: [数据结构, 算法, 排序]

categories: 数据结构
cover: https://static.guoxy.top/img/image-20200612210809033.png
description: 一些数据结构和算法的golang 实现
---



~~人老了就需要记录一下知识点~~

![](https://static.guoxy.top/img/structAndAl.jpg)

## 排序



### O( N^2 )

#### 冒泡排序

```go
func bubbleSort(list []int) []int {
	len := len(list)

	for i := 0; i < len; i++ {
		flag := false
		for j := 0; j < len-i-1; j++ {
			if list[j+1] < list[j] {
				list[j+1], list[j] = list[j], list[j+1]
				flag = true
			}
		}
		if !flag {
			break
		}
	}
	return list
}
```

+ 原地排序
+ 稳定排序

#### 插入排序

```go
func insterSort(list []int) []int {
	len := len(list)
	for i := 1; i < len; i++ {
		tmp := list[i] //把当前要排序的变量临时拿出来，就像整理扑克的时候把要整理的牌抽出来
		j := i - 1
		for ; j >= 0; j-- {
			if list[j] > tmp {
				list[j+1] = list[j] //如果牌比手上的牌大，就像后挪一位
			} else {
				break
			}
		}
		list[j+1] = tmp
	}
	return list
}
```

+ 原地排序
+ 稳定排序

#### 希尔排序

```go
func shellSort(list []int) []int {

	len := len(list)
	group := len / 2
	for group > 0 {
		for i := group; i < len; i += group {
			tmp := list[i]
			j := i - group
			for ; j >= 0; j -= group {
				if list[j] > tmp {
					list[j+group] = list[j]
				} else {
					break
				}
			}
			list[j+group] = tmp
		}
		group /= 2
	}
	return list
}
```

+ 其中group 最高效的情况无特定解

#### 选择排序

```go
func selectSort(list []int) []int {
	len := len(list)

	for i := 0; i < len-1; i++ {
		min := list[i]
		minIdx := i
		for j := i + 1; j < len; j++ {
			if list[j] < min {
				min = list[j]
				minIdx = j
			}
		}
		list[i], list[minIdx] = list[minIdx], list[i]
	}
	return list
}
```



### O（ NlogN ）

#### 归并排序 ( 自顶向下 )

```go
var tmp []int

func mergeSort(list []int) []int {
	tmp = make([]int, len(list)) //先申请一个，这样后面就不用重复申请
	sort(list, 0, len(list)-1)
	return list
}

func merge(list []int, lo int, mid int, hi int) {
	i := lo      //左半部分的起点
	j := mid + 1 //右半部分的起点

	for k := lo; k <= hi; k++ {
		tmp[k] = list[k]
	}

	for k := lo; k <= hi; k++ {
		if i > mid {
			list[k] = tmp[j]
			j++
		} else if j > hi {
			list[k] = tmp[i]
			i++
		} else if tmp[j] < tmp[i] {
			list[k] = tmp[j]
			j++
		} else {
			list[k] = tmp[i]
			i++
		}
	}
}
func sort(list []int, lo int, hi int) {
	if hi <= lo {
		return
	}
	mid := lo + (hi-lo)/2
	sort(list, lo, mid)
	sort(list, mid+1, hi)
	merge(list, lo, mid, hi)
}
```

#### 快速排序

```go
package sorts

func QuickSort(list []int) []int {
	size := len(list)
	qsort(list, 0, size-1)
	return list

}

func partition(list []int, lo int, hi int) int {
	left := lo       //左扫描指针
	right := hi      //右扫描指针
	flag := list[lo] //作为切分的元素)
	for {
		for list[left] <= flag {
			left++
			if left == hi {
				break
			}

		}
		for list[right] >= flag {
			right--
			if right == lo {
				break
			}
		}
		if left >= right {
			break
		}
		list[left], list[right] = list[right], list[left]
	}
	list[lo], list[right] = list[right], list[lo] //将切分元素放入，此时 left >= right, right指针靠左，所以 list[right] 交换
	return right
}

func qsort(list []int, lo int, hi int) {
	if hi <= lo {
		return
	}
	j := partition(list, lo, hi)
	qsort(list, lo, j)   //递归切分左半部分
	qsort(list, j+1, hi) //递归切分右半部分
}
```

#### 堆排序



### O（n）

#### 桶排序

```go
package sorts

// 桶排序
import s "sort"

// 获取待排序数组中的最大值
func getMax(a []int) int {
	max := a[0]
	for i := 1; i < len(a); i++ {
		if a[i] > max {
			max = a[i]
		}
	}
	return max
}

func bucketSort(a []int) {
	num := len(a)
	if num <= 1 {
		return
	}
	max := getMax(a)
	buckets := make([][]int, num) // 二维切片

	index := 0
	for i := 0; i < num; i++ {
		index = a[i] * (num - 1) / max                // 桶序号
		buckets[index] = append(buckets[index], a[i]) // 加入对应的桶中
	}

	tmpPos := 0 // 标记数组位置
	for i := 0; i < num; i++ {
		bucketLen := len(buckets[i])
		if bucketLen > 0 {
			s.Ints(buckets[i]) // 桶内做排序,这里使用go内建排序包，快排+堆排+希尔结合
			copy(a[tmpPos:], buckets[i])
			tmpPos += bucketLen
		}
	}
}
```

+ 我一开始的想法是每一个桶内的排序都起一个 goroutine 进行排序，但是**造成了严重的额外性能开销**，所以放弃
+ 第二天，桶数量限制在 8个，启动8个goroutine（处理器为8核），1000万数据排序用8s，串行排序则用了22s

```go
package sorts

import (
	s "sort"
	"sync"
)

var wg sync.WaitGroup

//PROCESS 逻辑处理器数，下面会用到
const PROCESS int = 8

func bucketSort(a []int) {
	size := len(a)
	if size <= 1 {
		return
	}

	//找到最大值
	max := a[0]
	for _, v := range a {
		if v > max {
			max = v
		}
	}

	buckets := make([][]int, PROCESS) //创建桶

	idx := 0
	// 将元素添加到对应的桶中
	for _, v := range a {
		idx = v * (PROCESS - 1) / max
		buckets[idx] = append(buckets[idx], v)
	}
	for _, bucket := range buckets {
		if bucket == nil {
			continue
		}
		wg.Add(1)
		go func(bucket []int) {
			defer wg.Done()
			s.Ints(bucket)
			return
		}(bucket)
	}
	//等待所有的排序完成
	wg.Wait()
	copyIdx := 0
	for _, bucket := range buckets {
		len := len(bucket)
		copy(a[copyIdx:], bucket)
		copyIdx = copyIdx + len
	}
	return
}
```

#### 计数排序

```go
package sorts

func countSort(a []int) []int {
	size := len(a)
	if size <= 1 {
		return a
	}
	//获得一个数组的最大值，函数在桶排序内容中
	max := getMax(a)
	//申请一个数组C用来做计数数组
	c := make([]int, max+1)

	//计算每个数出现的次数
	for _, v := range a {
		c[v]++
	}
	// 求 i 项 前i项和
	for i := 1; i <= max; i++ {
		c[i] = c[i] + c[i-1]
	}

	r := make([]int, size)

	for i := 0; i < size; i++ {
		idx := c[a[i]] - 1
		r[idx] = a[i]
		c[a[i]]--
	}
	return r
}
```

+ 计数排序是桶排序的变种
+ 计数排序的适用范围有限，一般用在数组 数值跨度不大，且比较集中，一个数字多次出现的的情况
+ 比如高考分数排名，用计数排序大概是最好的选择了

### Go 语言排序实现

```go
// Sort sorts data.
// It makes one call to data.Len to determine n, and O(n*log(n)) calls to
// data.Less and data.Swap. The sort is not guaranteed to be stable.
func Sort(data Interface) {
	n := data.Len()
	quickSort(data, 0, n, maxDepth(n))
}

// maxDepth returns a threshold at which quicksort should switch
// to heapsort. It returns 2*ceil(lg(n+1)).
func maxDepth(n int) int {
	var depth int
	for i := n; i > 0; i >>= 1 {
		depth++
	}
	return depth * 2
}

func quickSort(data Interface, a, b, maxDepth int) {
	for b-a > 12 { // Use ShellSort for slices <= 12 elements
		if maxDepth == 0 {
			heapSort(data, a, b)
			return
		}
		maxDepth--
		mlo, mhi := doPivot(data, a, b)
		// Avoiding recursion on the larger subproblem guarantees
		// a stack depth of at most lg(b-a).
		if mlo-a < b-mhi {
			quickSort(data, a, mlo, maxDepth)
			a = mhi // i.e., quickSort(data, mhi, b)， 相当于quickSort(data, mhi, b)
		} else {
			quickSort(data, mhi, b, maxDepth)
			b = mlo // i.e., quickSort(data, a, mlo) 相当于 quickSort(data, a, mlo)
		}
	}
	if b-a > 1 {
		// Do ShellSort pass with gap 6
		// It could be written in this simplified form cause b-a <= 12
		for i := a + 6; i < b; i++ {
			if data.Less(i, i-6) {
				data.Swap(i, i-6)
			}
		}
		insertionSort(data, a, b)
	}
}

func doPivot(data Interface, lo, hi int) (midlo, midhi int) {
	m := int(uint(lo+hi) >> 1) // Written like this to avoid integer overflow.
	if hi-lo > 40 {
		// Tukey's ``Ninther,'' median of three medians of three.
		s := (hi - lo) / 8
		medianOfThree(data, lo, lo+s, lo+2*s)
		medianOfThree(data, m, m-s, m+s)
		medianOfThree(data, hi-1, hi-1-s, hi-1-2*s)
	}
	medianOfThree(data, lo, m, hi-1)

	// Invariants are:
	//	data[lo] = pivot (set up by ChoosePivot)
	//	data[lo < i < a] < pivot
	//	data[a <= i < b] <= pivot
	//	data[b <= i < c] unexamined
	//	data[c <= i < hi-1] > pivot
	//	data[hi-1] >= pivot
	pivot := lo
	a, c := lo+1, hi-1

	for ; a < c && data.Less(a, pivot); a++ {
	}
	b := a
	for {
		for ; b < c && !data.Less(pivot, b); b++ { // data[b] <= pivot
		}
		for ; b < c && data.Less(pivot, c-1); c-- { // data[c-1] > pivot
		}
		if b >= c {
			break
		}
		// data[b] > pivot; data[c-1] <= pivot
		data.Swap(b, c-1)
		b++
		c--
	}
	// If hi-c<3 then there are duplicates (by property of median of nine).
	// Let's be a bit more conservative, and set border to 5.
	protect := hi-c < 5
	if !protect && hi-c < (hi-lo)/4 {
		// Lets test some points for equality to pivot
		dups := 0
		if !data.Less(pivot, hi-1) { // data[hi-1] = pivot
			data.Swap(c, hi-1)
			c++
			dups++
		}
		if !data.Less(b-1, pivot) { // data[b-1] = pivot
			b--
			dups++
		}
		// m-lo = (hi-lo)/2 > 6
		// b-lo > (hi-lo)*3/4-1 > 8
		// ==> m < b ==> data[m] <= pivot
		if !data.Less(m, pivot) { // data[m] = pivot
			data.Swap(m, b-1)
			b--
			dups++
		}
		// if at least 2 points are equal to pivot, assume skewed distribution
		protect = dups > 1
	}
	if protect {
		// Protect against a lot of duplicates
		// Add invariant:
		//	data[a <= i < b] unexamined
		//	data[b <= i < c] = pivot
		for {
			for ; a < b && !data.Less(b-1, pivot); b-- { // data[b] == pivot
			}
			for ; a < b && data.Less(a, pivot); a++ { // data[a] < pivot
			}
			if a >= b {
				break
			}
			// data[a] == pivot; data[b-1] < pivot
			data.Swap(a, b-1)
			a++
			b--
		}
	}
	// Swap pivot into middle
	data.Swap(pivot, b-1)
	return b - 1, c
}

```

**Go 语言的排序实现**

+ 首先判断计算出 数组的深度
+ 如果数组 长度大于 12 且深度为 0 ，采用堆排序
+ 如果数组长度大于 12 且深度不为 0 ，采用快排
  + 取标志位采用三次三数取中再取中法 （也成为 Turkey 的 9数法
  + 当一轮切分完成后，会判断有没有数组中有没有超过3个 切分元素，如果有超过三个切分元素，就会把切分元素们集中到中间位置
  + 当一轮完成后，满足 
    + data[lo, b-1) 为 小于 切分元素区间
    + data[b-1, c) 切分元素区间
    + data[c, hi) 大于切分元素区间
    + 然后 `return` `b-1` , `c`
    + 下一轮就只需要 排左边区间和右边区间，中间区间都是相同元素，就不用管他了
  + 下一轮，判断 上一轮左右区间哪个更长，为保证递归深度和最大深度一致，所以对小区间采用递归遍历，大区间采用循环遍历
  + 循环
+ 长度小于12 采用 `gap` 为6的希尔排序（其实就是以 6 为 步长 进行一次交换，然后再进行插入排序）









### 二叉树

#### 二叉查找树

+ 二叉查找树的定义  

  > 一棵二叉查找树（BST） 是一棵二叉树，其中每一个节点都含有一个Comparable的键以及相关联的值，每个节点的
  >
  > 键都大于其左子树中的任意节点的键而小于右子树的任意节点的键

+ 查找算法

  + 递归查找
  + 查找最大键和最小键
  + 查找第N大的键和第N小的键
  + 范围查找

+ 插入

  + 递归插入

+ 删除

  + 删除最大键和删除最小键
  + 删除 某个键

+ 遍历

  + 前序

  ```go
  package binarytree
  
  //TreeNode 节点
  type TreeNode struct {
  	Val   int
  	Left  *TreeNode
  	Right *TreeNode
  }
  
  var ans []int
  
  //// 二叉树的先序遍历
  // 递归方法
  func preorderTraversal(root *TreeNode) []int {
  	ans = []int{}
  	helper(root)
  	return ans
  
  }
  
  func helper(root *TreeNode) {
  	if root == nil {
  		return
  	}
  	ans = append(ans, root.Val)
  	helper(root.Left)
  	helper(root.Right)
  }
  
  //非递归方法(迭代)
  
  func preorderTraversal2(root *TreeNode) []int {
  	if root == nil {
  		return nil
  	}
  	ans := make([]int, 0)
  	stack := make([]*TreeNode, 0)
  	for root != nil || len(stack) != 0 {
  		for root != nil {
  			ans = append(ans, root.Val)
  			stack = append(stack, root)
  			root = root.Left
  		}
  		//left 到底了，所以 弹出一个元素，往他的右节点走
  		//pop
  		node := stack[len(stack)-1]
  		stack = stack[:len(stack)-1]
  		root = node.Right
  	}
  	return ans
  }
  ```

  + 中序

  ```go
  package binarytree
  
  // 递归实现
  func inorderTraversal(root *TreeNode) []int {
  
  	ans = []int{}
  	inorderHelper(root)
  	return ans
  }
  
  func inorderHelper(root *TreeNode) {
  	if root == nil {
  		return
  	}
  	inorderHelper(root.Left)
  	ans = append(ans, root.Val)
  	inorderHelper(root.Right)
  }
  
  // 非递归实现
  
  func inorderTraversal2(root *TreeNode) []int {
  	ans = []int{}
  	stack := []*TreeNode{}
  	for root != nil || len(stack) != 0 {
  		for root != nil {
  			stack = append(stack, root)
  			root = root.Left
  		}
  		//pop
  		node := stack[len(stack)-1]
  		stack = stack[:len(stack)-1]
  		ans = append(ans, node.Val)
  		root = node.Right
  	}
  	return ans
  }
  ```

  + 后序

  ```go
  package binarytree
  
  //后序遍历
  
  //递归实现
  
  func postorderTraversal(root *TreeNode) []int {
  	ans = []int{}
  	if root == nil {
  		return ans
  	}
  	postorderHelper(root)
  	return ans
  
  }
  
  func postorderHelper(node *TreeNode) {
  	if node == nil {
  		return
  	}
  	postorderHelper(node.Left)
  	postorderHelper(node.Right)
  	ans = append(ans, node.Val)
  }
  
  //非递归实现
  
  func postorderTraversal2(root *TreeNode) []int {
  	answer := []int{}
  	stack := []*TreeNode{}
  	var flag *TreeNode
  	if root == nil {
  		return answer
  	}
  	for root != nil || len(stack) != 0 {
  		for root != nil {
  			stack = append(stack, root)
  			root = root.Left
  		}
  		//先返回上一个节点
  		node := stack[len(stack)-1]
  
  		// 根节点必须在右节点弹出之后，再弹出
  		if node.Right == nil || flag == node.Right {
  			//pop
  			stack = stack[:len(stack)-1]
  			answer = append(answer, node.Val)
  			flag = node
  		} else {
  			//如果上一个弹出的不是该节点的右子节点
  			//那么就跳转到它的右子节点
  			root = node.Right
  		}
  
  	}
  	return answer
  }
  ```
	

## 红黑树

```go
package redblacktree

const (
	// RED is true
	RED = true
	// BLACK is false
	BLACK = false
)

//Node 定义为红黑树的节点
type Node struct {
	Key   int
	Value interface{}
	Color bool
	Left  *Node
	Right *Node
	N     int
}

//RedBlackTree 定义一棵红黑树
type RedBlackTree struct {
	root   *Node
	length int
}

func isRed(n *Node) bool {
	return n != nil && n.Color
}

// InitRBTree 初始化一个红黑树
func InitRBTree() *RedBlackTree {
	return &RedBlackTree{}
}

func initNode(k int, v interface{}, size int, color bool) *Node {
	return &Node{
		Key:   k,
		Value: v,
		N:     size,
		Color: color,
	}
}

// Size 用于计算子树大小
func (t *RedBlackTree) Size(node *Node) int {
	if node == nil {
		return 0
	}
	return node.N
}

func (t *RedBlackTree) rotateLeft(node *Node) *Node {
	x := node.Right
	node.Right = x.Left
	x.Left = node
	x.Color = node.Color
	node.Color = RED
	x.N = node.N
	node.N = 1 + t.Size(node.Left) + t.Size(node.Right)
	return x

}

func (t *RedBlackTree) rotateRight(node *Node) *Node {
	x := node.Left
	node.Left = x.Right
	x.Right = node
	x.Color = node.Color
	node.Color = RED
	x.N = node.N
	node.N = 1 + t.Size(node.Left) + t.Size(node.Right)
	return x
}

// 颜色翻转,将两个红叶子节点变成黑色,再将自己变成红色
func (t *RedBlackTree) colorFlip(node *Node) {
	node.Color = RED
	node.Left.Color = BLACK
	node.Right.Color = BLACK
	return
}

// Put 向红黑树中插入函数
func (t *RedBlackTree) Put(key int, value interface{}) {
	t.root = t.put(t.root, key, value)
	// 红黑树根节点总是为黑色
	t.root.Color = BLACK
}

// put 用的递归辅助方法
func (t *RedBlackTree) put(node *Node, key int, value interface{}) *Node {
	if node == nil {
		return initNode(key, value, 1, RED)
	}
	cmp := key - node.Key
	switch {
	case cmp > 0:
		node.Right = t.put(node.Right, key, value)
	case cmp < 0:
		node.Left = t.put(node.Left, key, value)
	case cmp == 0:
		node.Value = value
	}

	//对树进行平衡操作
	// 右红 左不红 ,左旋
	if isRed(node.Right) && !isRed(node.Left) {
		node = t.rotateLeft(node)
	}
	if isRed(node.Left) && isRed(node.Left.Left) {
		node = t.rotateRight(node)
	}
	if isRed(node.Left) && isRed(node.Right) {
		t.colorFlip(node)
	}
	node.N = 1 + t.Size(node.Left) + t.Size(node.Right)
	return node
}

```



### 并查集

```go
package unnionfind

// UF 并查集数据结构
type UF struct {
	elements []int

	//给并查集加权用的
	//将小树挂在大树上以降低数的高度
	size []int
}

//IsContained 判断pq 两个元素是否联通
func (u *UF) IsContained(p, q int) bool {
	//如果这两个节点的根节点是相同的就表示是联通的
	return u.root(p) == u.root(q)
}

// Union 函数用来联通两个节点
func (u *UF) Union(p, q int) {
	pRoot := u.root(p)
	qRoot := u.root(q)
	if pRoot == qRoot {
		return
	}

	if u.size[pRoot] < u.size[qRoot] {
		//如果 p 的树比 q 的树小,那么就把 p接在q上
		u.elements[pRoot] = qRoot
		u.size[qRoot] += u.size[pRoot]
	} else {
		u.elements[qRoot] = pRoot
		u.size[pRoot] = u.size[qRoot]
	}
	return

}

func (u *UF) root(num int) int {
	//这样的写法每次只能 回溯 一层
	// for num != u.elements[num] {
	// 	num = u.elements[num]
	// }
	// 路径压缩
	// 将路径上的每个节点都指向他的祖父节点
	for num != u.elements[num] {
		// u.elements[num] 表示num的父节点
		// 如果 num 的父节点 不是num
		// 就将 num 的父节点设成 num 当前父节点的父节点 -- 祖父节点
		u.elements[num] = u.elements[u.elements[num]]
		// 进入num 的 父节点继续寻找
		num = u.elements[num]
	}
	return num
}
```

