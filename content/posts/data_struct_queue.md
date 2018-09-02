---
title: "数据结构之队列"
date: 2018-08-02T07:12:55+08:00
tags: ["Go","数据结构","队列"]
---

## 概念
队列(queue)是只允许在-端进行插入操作，而在另一端进行删除操作的线性表。队列是一种先进先出 (First IN First Out) 的线性表，简称 FIFO。允许插入的一端称为队尾，允许删除的一端称为队头。

## 代码示例
```go
package main

import "fmt"

// 数据类型，支持任意类型
type Elem interface{}

// 队列结构定义
type Queue struct {
	data   []Elem
	length int
}

// 新建队列
func NewQueue() *Queue {
	return &Queue{}
}

// 队列长度
func (q *Queue) Len() int {
	return q.length
}

// 判断为空
func (q *Queue) IsEmpty() bool {
	return q.length == 0
}

// 入队列
func (q *Queue) InQueue(e Elem) {
	q.data = append(q.data, e)
	q.length++
}

// 出队列
func (q *Queue) OutQueue() Elem {
	if q.IsEmpty() {
		fmt.Println("The queue is empty")
		return nil
	}
	e := q.data[0]
	q.data = q.data[1:]
	q.length--
	return e
}

// 打印队列数据
func (q *Queue) Print() {
	if !q.IsEmpty() {
		fmt.Printf("Queue:")
		for _, e := range q.data {
			fmt.Printf("%+v ", e)
		}
	}
	fmt.Println()
}

func main() {
	q := NewQueue()
	q.InQueue(3)
	q.InQueue(6)
	q.InQueue(0)
	q.InQueue(8)
	q.Print()
	fmt.Println("Queue Len:", q.Len())
	fmt.Println("Pop:", q.OutQueue())
	fmt.Println("Pop:", q.OutQueue())
	fmt.Println("Pop:", q.OutQueue())
	q.InQueue(9)
	q.Print()
	fmt.Println("Queue Len:", q.Len())

}

```