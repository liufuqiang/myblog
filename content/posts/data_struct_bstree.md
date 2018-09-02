---
title: "数据结构之二叉有序树（搜索树）"
date: 2018-08-02T10:51:02+08:00
tags: ["Go","数据结构","树","二叉树","搜索树","有序树"]
---
## 概念

- **二叉树**(Binary Tree)： 是n(n>=O)个结点的有限集合，该集合或者为空集(称为空二叉树)，或者由一个根结点和两颗互不相交的、分别称为根节点的左子树和右子树组成。
- **完全二叉树** ：对一棵具有 n 个结点的二叉树按层序编号，如果编号为 i (1<=i<=n) 的结点与同样深度的满二叉树中编号为i的结点在二叉树中位置完全相同，则这棵二叉树称为完全二叉树。
-  **有序树** ： 有序树是个完全二叉树，且左子树的值小于根结点的值，右子树的值大于根结点的值。

## 代码示例

```go
package main

import "fmt"

// 本示例是有序的搜索树，即右子树 < 父节点 < 左子树， 用中序遍历进行操作

// 节点结构
type Node struct {
	Value  int
	lchild *Node
	rchild *Node
}

// 以某个节点为父节点，插入一个新节点
func (node *Node) Insert(e int) {
	n := &Node{Value: e}

	if e < node.Value { // 在左侧插入
		if node.lchild != nil { // 如果左子树存在, 递归插入
			node.lchild.Insert(e)
		} else { // 新节点就是左子树了
			node.lchild = n
		}
	} else { // 在右侧插入
		if node.rchild != nil { // 如果右子树存在， 递归插入
			node.rchild.Insert(e)
		} else { // 新节点就是右子树
			node.rchild = n
		}
	}
}

// 中序遍历，获取全部value
func (n *Node) InOrder(result *[]int) {
	// 先左子树，再根节点，再右子树
	if n.lchild != nil {
		n.lchild.InOrder(result)
	}
	*result = append(*result, n.Value)
	if n.rchild != nil {
		n.rchild.InOrder(result)
	}
}

// 搜索树结构
type BSTree struct {
	Root *Node
}

// 插入新节点
func (t *BSTree) Insert(e int) {

	if t.Root == nil {
		node := &Node{Value: e}
		t.Root = node
	} else {
		t.Root.Insert(e)
	}
}

// 中序遍历
func (t *BSTree) InOrder() []int {
	if t.Root == nil {
		return nil
	}
	var result []int
	t.Root.InOrder(&result)
	return result
}

// 找到最大的值
func (t *BSTree) FindMax() int {
	var node = t.Root
	if node == nil {
		fmt.Println("The tree is null")
		return -1
	}
	//最大值是右子树的最右节点
	for node.rchild != nil {
		node = node.rchild
	}
	return node.Value
}

// 找最小的值
func (t *BSTree) FindMin() int {
	var node = t.Root
	if node == nil {
		fmt.Println("The tree is null")
		return -1
	}
	// 最小值是左子树的最左节点
	for node.lchild != nil {
		node = node.lchild
	}
	return node.Value
}

// 搜索指定的值是否存在
func (t *BSTree) Search(val int) bool {
	var node = t.Root
	if node == nil {
		fmt.Println("The tree is null")
		return false
	}

	for {
		if node == nil { // 如果不存在子树，就是没找到
			return false
		}

		if node.Value == val { // 子树根，返回true
			return true
		}
		if val < node.Value { // 如果val小于根，到左子树找
			node = node.lchild
		} else { // 到右子树找
			node = node.rchild
		}
	}
}

func main() {

	var tree = new(BSTree)
	tree.Insert(1)
	tree.Insert(4)
	tree.Insert(2)
	tree.Insert(232)
	tree.Insert(42)
	tree.Insert(142)
	tree.Insert(62)

	fmt.Println("中序遍历:", tree.InOrder())

	fmt.Println("Max:", tree.FindMax())
	fmt.Println("Min:", tree.FindMin())

	fmt.Println("Find 43:", tree.Search(43))
	fmt.Println("Find 2:", tree.Search(2))
	fmt.Println("Find 142:", tree.Search(142))
	fmt.Println("Find 99:", tree.Search(99))
}
```