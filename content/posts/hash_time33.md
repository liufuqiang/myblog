---
title: "简单的hash算法-time33"
date: 2018-09-04T22:55:30+08:00
tags: ["算法","hash","time33","Go"]
---

## 概述
我们今天介绍一个非常简单的hash算法Time33， 这是一个对字符串进行哈希的函数，现在几乎所有流行的HashMap都采用了DJB Hash Function，俗称“Times33”算法。Times33的算法很简单，就是对字符串进行逐个字符遍历，不断的乘33，然后把数值相加即可。

## 为什么是33和5381？
[这里](https://stackoverflow.com/questions/10696223/reason-for-5381-number-in-djb-hash-function/13809282#13809282)有关于这个问题的较好的回答。
5381的独特性有：

- 奇数
- 质数
- [亏数](https://zh.wikipedia.org/zh-cn/%E4%BA%8F%E6%95%B0)（在数论中，若一个正整数除了本身外之所有因数之和比此数自身小，则称此数为亏数。 更为严格地说，亏数是指使得函数σ < 2n的正整数，其中σ指的是因数和函数，即n的所有正因数之和。2n − σ称作n的亏度。 例如15的真因数有1,3,5，而1+3+5=9，9<15，所以15可称为亏数。）
- 二进制是001/010/100/000/101 

## 代码示例
```go
package main

import (
	"crypto/md5"
	"fmt"
	"strconv"
)

func main() {

	for i := 0; i < 1000000; i++ {
		hash := time33(strconv.Itoa(i))
		fmt.Println(i, "\t", hash)
	}
}

func time33(str string) int64 {
	hash := int64(5381) // 001 010 100 000 101 ,hash后的分布更好一些
	s := fmt.Sprintf("%x", md5.Sum([]byte(str)))
	for i := 0; i < len(s); i++ {
		hash = hash*33 + int64(s[i])
	}
	return hash
}
```

我们测试了100万的hash数据可以发现冲突率是0，可见time33还是比较靠谱的hash算法。
