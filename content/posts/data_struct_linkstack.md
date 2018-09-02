---
date: 2018-08-02T00:36:58+08:00
title: "数据结构之链栈"
tags: ["Go","数据结构","链接栈"]

---

## 概念
所谓链栈，就是栈的链式存储。

## 代码示例
```go
package main

import "fmt"

// 元素类型，支持任意元素
type Elem interface{}

// 节点的定义
type Node struct {
	data Elem
	next *Node
}

func (n *Node) Next() *Node {
	return n.next
}

func (n *Node) SetNext(node *Node) {
	n.next = node
}

func (n *Node) Value() Elem {
	return n.data
}

// 链栈
type LinkStack struct {
	top    *Node
	length int
}

func (ls *LinkStack) Pop() Elem {
	if ls.Len() == 0 {
		fmt.Println("No Element in stack.")
		return nil
	}
	defer func() {
		ls.top = ls.top.Next()
		ls.length--
	}()

	return ls.top.Value()
}

func (ls *LinkStack) Push(e Elem) {
	node := &Node{data: e}

	node.SetNext(ls.top)
	ls.top = node
	ls.length++

}

func (ls *LinkStack) Len() int {
	return ls.length
}

func (ls *LinkStack) Each(f func(Elem)) {
	n := ls.top
	for i := 0; i < ls.Len(); i++ {
		f(n)
		n.Next()
	}
}

func (ls *LinkStack) Print() {

	if ls.top != nil {
		fmt.Println("The LinkStack is:")
		for node := ls.top; node != nil; node = node.Next() {
			fmt.Printf("%+v ", node.Value())
		}
		fmt.Println()
	}
}

func main() {

	var stack LinkStack
	stack.Push(1)
	stack.Push(2)
	stack.Push(3)
	stack.Push(4)

	stack.Print()
	fmt.Println("now len:", stack.Len())

	fmt.Println("pop:", stack.Pop())
	fmt.Println("pop:", stack.Pop())
	stack.Print()
	fmt.Println("now len:", stack.Len())

}
```