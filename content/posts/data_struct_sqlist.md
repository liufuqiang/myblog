---
title: "数据结构之线性表顺序存储"
date: 2018-08-01T23:29:15+08:00
tags: ["Go","数据结构","线性表"]
---
### 概念描述
线性表的顺序存储结构，指的是用一段地址连续的存储单元依次存储线性表的数据元素 。

### 代码示例

```go
package main

import "fmt"

type Elem interface{} // 可以存储任意类型

// 线性表结构
type SqList struct {
	len     int    // 数据长度
	maxSize int    // 表最大长度
	data    []Elem // 数据表
}

// 初始化一个线性表
func New(num int) *SqList {
	return &SqList{maxSize: num, len: 0, data: make([]Elem, num)}
}

// 判断线性表是否为空
func (list *SqList) IsEmpty() bool {
	return list.len == 0
}

// 判断线性表是否满了
func (list *SqList) IsFull() bool {
	return list.len == list.maxSize
}

// 在指定位置插入数据（从插入点之后的元素依次后移）
func (list *SqList) Insert(pos int, e Elem) bool {
	if list.IsFull() {
		fmt.Println("The List is Full,Can't Insert any Elem")
		return false
	}
	//0,1,...,(),...len-1, len
	for i := list.len; i >= pos-1; i-- {
		list.data[i] = list.data[i-1]
	}
	list.data[pos-1] = e
	list.len++
	return true
}

// 删除指定位置的数据（ 删除后，后边元素依次前移）
func (list *SqList) Delete(pos int) bool {
	if pos < 1 || pos > list.len {
		fmt.Println("I must between 1 and ", list.len)
		return false
	}
	// 0,1,...i-1,i,...len-1
	for i := pos - 1; i <= list.len-1; i++ {
		list.data[i] = list.data[i+1]
	}
	list.data[list.len-1] = 0
	list.len--
	return true
}

// 获取指定位置的数据
func (list *SqList) GetElem(pos int) Elem {
	if pos < 1 || pos > list.len {
		fmt.Println("I must between 1 and ", list.len)
		return nil
	}
	return list.data[pos-1]
}

// 线性表尾部追加数据
func (list *SqList) Append(e Elem) bool {
	if list.IsFull() {
		return false
	}
	list.data[list.len] = e
	list.len++
	return true
}

// 打印当前数据
func (list *SqList) Print() {
	fmt.Printf("SqList :")
	for _, e := range list.data {
		fmt.Printf("%+v ", e)
	}
	fmt.Println()
}

func main() {

	sq := New(10)
	sq.Append(99)
	sq.Append(999)
	sq.Append(9999)
	sq.Append(99999)
	sq.Print()
	sq.Insert(4, 888)
	sq.Print()
	sq.Delete(2)
	sq.Print()
	fmt.Println("GetElem(3)=", sq.GetElem(3)) // 888

}

```


