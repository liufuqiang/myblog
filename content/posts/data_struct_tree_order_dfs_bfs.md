---
title: "数据结构之树的各种遍历方式"
date: 2018-08-02T15:24:49+08:00
tags: ["Go","数据结构","队列"]
---

## 概述
这里介绍了树的各种遍历方式，包括前序、中序、后续，已经他们的非递归实现，还有深度优先遍历和广度优先遍历。

## 代码示例
```go
package main

import (
	"fmt"
)

type Elem interface{}

// 数据结点
type Node struct {
	lchild *Node
	rchild *Node
	data   Elem
}

// 树结构
type Tree struct {
	Root *Node
}

// 前序遍历 - 递归版本
func (n *Node) PreOrder() {
	fmt.Printf("%+v ", n.data)
	if n.lchild != nil {
		n.lchild.PreOrder()
	}
	if n.rchild != nil {
		n.rchild.PreOrder()
	}
}

func (t *Tree) PreOrder() {
	if t.Root == nil {
		fmt.Println("Tree is Null")
		return
	}
	fmt.Printf("PreOrder:")
	t.Root.PreOrder()
}

// 实现一个简单的堆

type Stack struct {
	data []Elem
}

func (s *Stack) Pop() Elem {
	if len(s.data) == 0 {
		fmt.Println("Stack is Null")
		return nil
	}
	old := s.data
	n := len(old)
	x := old[n-1]
	s.data = old[0 : n-1]
	return x
}

func (s *Stack) IsEmpty() bool {
	return len(s.data) == 0
}

func (s *Stack) Push(e Elem) {
	s.data = append(s.data, e)
}

// 前序遍历-非递归版本
func (n *Node) PreOrder2() {
	stack := new(Stack)
	pNode := n
	for pNode != nil || !stack.IsEmpty() {
		if pNode != nil {
			fmt.Printf("%+v ", pNode.data)
			stack.Push(pNode)
			pNode = pNode.lchild
		} else {
			nd := stack.Pop()
			pNode = nd.(*Node).rchild
		}
	}
}

func (t *Tree) PreOrder2() {
	if t.Root == nil {
		fmt.Println("Tree is Null")
		return
	}
	fmt.Printf("PreOrder2:")
	t.Root.PreOrder2()
}

// 中序遍历 - 递归版本
func (n *Node) InOrder() {
	if n.lchild != nil {
		n.lchild.InOrder()
	}
	fmt.Printf("%+v ", n.data)
	if n.rchild != nil {
		n.rchild.InOrder()
	}
}

func (t *Tree) InOrder() {
	if t.Root == nil {
		fmt.Println("Tree is Null")
		return
	}
	fmt.Printf("InOrder:")
	t.Root.InOrder()
}

// 中序遍历-非递归版本
func (n *Node) InOrder2() {
	stack := new(Stack)
	pNode := n
	for pNode != nil || !stack.IsEmpty() {
		if pNode != nil {
			stack.Push(pNode)
			pNode = pNode.lchild
		} else {
			nd := stack.Pop()
			fmt.Printf("%+v ", nd.(*Node).data)
			pNode = nd.(*Node).rchild
		}
	}
}

func (t *Tree) InOrder2() {
	if t.Root == nil {
		fmt.Println("Tree is Null")
		return
	}
	fmt.Printf("InOrder2:")
	t.Root.InOrder2()
}

// 后序遍历 - 递归版本
func (n *Node) PostOrder() {
	if n.lchild != nil {
		n.lchild.PostOrder()
	}
	if n.rchild != nil {
		n.rchild.PostOrder()
	}
	fmt.Printf("%+v ", n.data)
}

func (t *Tree) PostOrder() {
	if t.Root == nil {
		fmt.Println("Tree is Null")
		return
	}
	fmt.Printf("PostOrder:")
	t.Root.PostOrder()
}

type Queue struct {
	data []Elem
}

func (q *Queue) IsEmpty() bool {
	return len(q.data) == 0
}

func (q *Queue) In(e Elem) {
	q.data = append(q.data, e)
}

func (q *Queue) Out() Elem {
	e := q.data[0]
	q.data = q.data[1:]
	return e
}

// 广度优先遍历
func (n *Node) BFS() {

	queue := new(Queue)
	queue.In(n)

	for !queue.IsEmpty() {
		node := queue.Out().(*Node)
		fmt.Printf("%+v ", node.data)
		if node.lchild != nil {
			queue.In(node.lchild)
		}
		if node.rchild != nil {
			queue.In(node.rchild)
		}
	}
}

func (t *Tree) BFS() {
	if t.Root == nil {
		fmt.Println("Tree is Null")
		return
	}
	fmt.Println("DSF: ")
	t.Root.BFS()
}

// 深度优先遍历, 同前序遍历
func (n *Node) DFS() {
	stack := new(Stack)
	pNode := n
	stack.Push(pNode)
	for !stack.IsEmpty() {
		node := stack.Pop().(*Node)
		fmt.Printf("%+v ", node.data)
		if node.rchild != nil {
			stack.Push(node.rchild)
		}
		if node.lchild != nil {
			stack.Push(node.lchild)
		}
	}
}

func (t *Tree) DFS() {
	if t.Root == nil {
		fmt.Println("Tree is Null")
		return
	}
	fmt.Println("DSF: ")
	t.Root.DFS()
}

func main() {

	tree := new(Tree)
	tree.Root = &Node{data: 5}
	tree.Root.lchild = &Node{data: 3}
	tree.Root.lchild.lchild = &Node{data: 7}
	tree.Root.lchild.rchild = &Node{data: 2}
	tree.Root.rchild = &Node{data: 4}
	tree.Root.rchild.lchild = &Node{data: 8}
	tree.Root.rchild.rchild = &Node{data: 1}

	// 前序
	tree.PreOrder()
	fmt.Println()
	tree.PreOrder2()
	fmt.Println()

	// 中序
	tree.InOrder()
	fmt.Println()
	tree.InOrder2()
	fmt.Println()

	//后序
	tree.PostOrder()
	fmt.Println()

	//BFS
	tree.BFS()
	fmt.Println()

	//DFS
	tree.DFS()
	fmt.Println()
}
```
