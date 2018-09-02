---
title: "数据结构之实战-用搜索树实现一个ipquery服务"
date: 2018-08-02T11:22:03+08:00
tags: ["Go","数据结构","树","二叉树","搜索树","ipquery"]
---

## 介绍
ipquery是比较常用的根据ip查询归属地、运营商等信息的服务，服务需要一份ip库作为基础数据，一般的实现方法是把ip库按照ip段的有序排序后，用二分查找的方法实现。 我们今天利用下二叉树的有序树，来实现下ipquery服务。
需要注意的是， 由于有序树的问题，我们需要满足满二叉树，而树的高度越大就会导致树的操作会越来越差，所以我们一定要控制好树的高度。而传统的ip库文件里都是对ip做了有序排列的，如果我们之间用有序的ip库来构建有序树的话，会导致把树构建成一个“斜树”（递增的ip库会构建成一个右斜树，高度就是ip库的个数-1，而ip库一般都是十万级的个数，这是非常恐怖的）。因此，我们需要把ip库文件内容进行随机打散，这样会大大降低树的高度。如下是一个简单的打散文件的小技巧：
```bash
cat ip.txt.old|awk '{print rand(),$0}'|sort -k1n|awk  '{gsub($1FS,""); print $0}' > ip.txt
```

## 代码示例
```go
package main

import (
	"bytes"
	"encoding/binary"
	"fmt"
	"io/ioutil"
	"net"
	"strconv"
	"strings"

	"github.com/gin-gonic/gin"
)

// ip.txt 格式为: 4034607104	4034623487	未知	未知	未知	未知	保留地址	未知
// ip.txt 最好是比较乱序的，这样构建的树相对深度小一些。 如果本文件是有序的，会形成一个斜树，随着深度增加插入速度越来越慢
// 一个打散有序文件的小技巧: cat ip.txt.old|awk '{print rand(),$0}'|sort -k1n|awk  '{gsub($1FS,""); print $0}' > ip.txt

// 数据节点
type Node struct {
	begin  int    // IP段的起始
	end    int    // IP段的结束
	desc   string // IP属性描述信息
	Lchild *Node
	Rchild *Node
}

// 基于当前结点插入新节点（满足有序二叉树特点）
func (node *Node) Insert(begin, end int, desc string) {
	if end < node.begin {
		if node.Lchild == nil {
			node.Lchild = &Node{begin: begin, end: end, desc: desc}
		} else {
			node.Lchild.Insert(begin, end, desc)
		}
	}

	if begin > node.end {
		if node.Rchild == nil {
			node.Rchild = &Node{begin: begin, end: end, desc: desc}
		} else {
			node.Rchild.Insert(begin, end, desc)
		}
	}
}

// 二叉搜索树结构
type BSTree struct {
	Root *Node
}

// 插入新节点
func (t *BSTree) Insert(begin, end int, desc string) {
	if t.Root == nil {
		t.Root = &Node{begin: begin, end: end, desc: desc}
	} else {
		t.Root.Insert(begin, end, desc)
	}
}

// 在二叉树中搜索ip的记录
func (t *BSTree) Search(iplong int) string {
	var node = t.Root
	if node == nil {
		fmt.Println("The tree is null")
		return ""
	}

	for {
		if node == nil {
			fmt.Println("can't find ip:", iplong)
			return ""
		}

		// 先序方式
		if iplong >= node.begin && iplong <= node.end {
			return node.desc
		}
		if iplong < node.begin {
			node = node.Lchild
		}
		if iplong > node.end {
			node = node.Rchild
		}
	}
}

// 定义个全局变量，ip树对象
var ipTree = new(BSTree)

// 初始化ip树
func initIPData() {
	data, err := ioutil.ReadFile("./ip.txt")
	if err != nil {
		fmt.Println("read ipdata file faild,", err)
		return
	}

	for _, line := range strings.Split(string(data), "\n") {
		line := strings.TrimSpace(line)
		if line == "" {
			continue
		}
		cols := strings.Split(line, "\t")
		begin, _ := strconv.Atoi(cols[0])
		end, _ := strconv.Atoi(cols[1])
		desc := strings.Join(cols[2:], "\t")
		ipTree.Insert(begin, end, desc)
	}
}

func main() {

	initIPData()
	router := gin.Default()
	router.GET("/ipquery", ipquery)
	router.Run(":1234")
}

func ipquery(c *gin.Context) {
	ip := c.DefaultQuery("ip", "182.118.20.208")
	iplong := ip2Long(ip)
	ipresult := ipTree.Search(int(iplong))
	//fmt.Printf("Search %s(%d) is %s:", ip, iplong, ipresult)
	c.JSON(200, gin.H{"data": ipresult, "ip": ip, "iplong": iplong})
}

// ip 从字符串转整型
func ip2Long(ip string) uint32 {
	var long uint32
	binary.Read(bytes.NewBuffer(net.ParseIP(ip).To4()), binary.BigEndian, &long)
	return long
}
```

这样一个非常简单高效的ipquery服务就搞定了，如下测试下：
```bash
curl 'http://localhost:1234/ipquery?ip=103.213.248.89'
{"data":"中国\t香港特别行政区\t未知\t未知\t未知\t未知","ip":"103.213.248.89","iplong":1742075993}
```
