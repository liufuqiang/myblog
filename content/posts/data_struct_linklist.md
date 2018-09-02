---
title: "数据结构之线性表链式存储（链表）"
date: 2018-08-01T23:57:48+08:00
tags: ["Go","数据结构","线性表","链式存储","链表"]
---

## 概念
为了表示每个数据元素ai与其直接后继数据元素 ai+1之间的逻辑关系，对数据元素ai来说 ， 除了存储其本身的信息之外，还需存储一个指示其直接后继的信息 (即直接后继的存储位置)。我们把存储数据元素信息的域称为数据域,把存储直接后继位置的域称为指针域。 指针域中存储的信息称做指针或链。 这两部分信息组成数据元素ai的存储映像，称为结点 (Node)。
n 个结点 (ai的存储映像) 链结成一个链表，即为线性表 (a1， a2,... , an) 的链式存储结构，因为此链表的每个结点中只包含一个指针域 ，所以叫做单链表。

## 代码示例
```go
package main

import "fmt"

// 数据元素类型，支持任意类型
type Elem interface{}

// 链表节点结构
type Node struct {
	value Elem  // 数据
	next  *Node // 下一个节点的指针
}

// 返回下一个节点的地址
func (n *Node) Next() *Node {
	return n.next
}

// 设置下一个节点的位置
func (n *Node) SetNext(node *Node) *Node {
	n.next = node
	return n
}

// 获取节点的数据值
func (n *Node) Value() interface{} {
	return n.value
}

// 链表结构
type LinkList struct {
	head   *Node // 头指针
	tail   *Node // 尾指针
	length int   // 链表长度
}

// 链表长度
func (list *LinkList) Length() int {
	return list.length
}

// 链表尾部追加数据节点
func (list *LinkList) Append(val Elem) {
	// 创建一个节点，然后把节点挂到尾部
	node := &Node{value: val}
	if list.head == nil { // 如果头部为空，头指针为空时(链表为空）直接把新节点挂到头指针上
		list.head = node
	} else { // 挂到尾部
		list.tail.SetNext(node)
	}
	list.length++
	list.tail = node
}

// 指定位置插入节点
func (list *LinkList) Insert(pos int, e Elem) {
	if pos < 0 || pos > list.length {
		fmt.Printf("Pos must between %d and %d\n", 1, list.length)
		return
	}
	node := &Node{value: e}

	if pos == 1 { //head
		node.SetNext(list.head)
		list.head = node
	} else {
		// 从头开始，一直移动到指定位置的节点上
		n := list.head
		for i := 0; i < pos-1; i++ {
			n = n.Next()
		}
		node.SetNext(n.Next())
		n.SetNext(node)
	}
	list.length++
}

// 获取指定位置的数据
func (list *LinkList) Get(pos int) interface{} {
	if pos < 0 || pos > list.length {
		fmt.Printf("Pos must between %d and %d\n", 1, list.length)
		return nil
	}

	// 移动到指定位置
	n := list.head
	for i := 0; i < pos-1; i++ {
		n = n.Next()
	}
	return n.Value()
}

// 删除指定位置的节点
func (list *LinkList) Delete(pos int) bool {
	if pos < 1 || pos > list.length {
		fmt.Printf("Pos must between %d and %d\n", 1, list.length)
		return false
	}

	n := list.head
	if pos == 1 {
		list.head = n.Next()
	} else {
		for i := 1; i < pos-1; i++ {
			n = n.Next()
		}
		n.SetNext(n.Next().Next())
	}
	list.length--
	return true
}

// 迭代链表
func (list *LinkList) Each(f func(val Elem)) {
	for n := list.head; n != nil; n = n.Next() {
		f(n)
	}
}

// 打印链表
func (list *LinkList) Print() {
	if list.head != nil {
		fmt.Printf("The LinkList is:")
		for n := list.head; n != nil; n = n.Next() {
			fmt.Printf("%+v ", n.Value())
		}
		fmt.Println()
	}
}

func main() {

	var list LinkList
	list.Append(1)
	list.Append(2)
	list.Append(3)
	list.Append(4)
	list.Print()
	fmt.Println(list.Length())
	list.Insert(2, "刘付强")
	list.Insert(4, "刘开心")
	list.Print()
	fmt.Println("len:", list.Length())
	fmt.Println(list.Get(3))
	list.Delete(3)
	list.Print()
	fmt.Println("len:", list.Length())
	list.Delete(4)
	list.Print()
	fmt.Println("len:", list.Length())
}
```
