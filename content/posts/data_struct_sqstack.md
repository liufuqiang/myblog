---
title: "数据结构之线性栈"
date: 2018-08-02T00:26:23+08:00
tags: ["Go","数据结构","线性栈"]

---

## 概念
栈(stack)是限定仅在表尾进行插入和删除操作的线性表。我们把允许插入和删除的一端称为栈顶 (top ，另一端称为栈底 (bottom)，不含任何数据元素的栈称为空钱。栈又称为后进先出 (LastIn FilrstOut) 的线性衰，简称LlFO 结构。

## 代码示例
```go
package main

import (
	"fmt"
)

// 最大元素个数
const STACK_SIZE = 10

// 数据节点存储的数据
type Elem interface{}

// 栈结构
type SqStack struct {
	top  int              //栈顶指针
	data [STACK_SIZE]Elem // 数据元素
}

// 弹出栈顶元素
func (stack *SqStack) Pop() Elem {
	if stack.Len() == 0 {
		return nil
	}
	elm := stack.data[stack.top-1]
	stack.data[stack.top-1] = nil
	stack.top--
	return elm
}

// 压入栈顶新元素
func (stack *SqStack) Push(e Elem) {
	if stack.Len() == STACK_SIZE {
		fmt.Println("Stack is Full")
		return
	}
	stack.data[stack.top] = e
	stack.top++
}

// 栈的长度
func (stack *SqStack) Len() int {
	return stack.top
}

// 打印栈内数据
func (stack *SqStack) Print() {
	fmt.Printf("Stack is:")
	for _, e := range stack.data {
		fmt.Printf("%+v ", e)
	}
	fmt.Println()
}

func main() {
	var stack SqStack
	stack.Push(1)
	stack.Push(2)
	stack.Push(3)
	stack.Push(4)
	stack.Print()
	fmt.Println("pop:", stack.Pop())
	fmt.Println("pop:", stack.Pop())
	stack.Print()
}
```


