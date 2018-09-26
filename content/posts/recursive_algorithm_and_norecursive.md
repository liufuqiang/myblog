---
title: "递归算法，递归优化和非递归算法的比较"
date: 2018-07-29T15:10:30+08:00
tags: ["算法","递归"]
---

### 递归算法
递归算法是一种直接或者间接调用自身函数或者方法的算法。相对于把复杂问题变成小问题，然后小问题可以最终收敛到有解状态（递归的终止条件）。 递归的用途非常广，如我们在数的各种遍历操作里非常常见的用法。 但递归本身因为是频繁的产生调用栈的入栈、出栈，性能上是比较差的。

下边我们用经典的斐波纳挈数列来做几个比较， 包括

- 纯递归的方式
- 优化的递归方式（增加Cache机制）
- 尾递归方式
- 非递归的方式

### 程序示例
```go
package main

import (
	"fmt"
	"time"
)

func main() {

	n := 50

	t := time.Now()
	fmt.Println("recursion:", fib(n))
	fmt.Println("cosume:", time.Now().Sub(t))

	t = time.Now()
	fmt.Println("recursion WithCache:", fibWithCache(n))
	fmt.Println("cosume:", time.Now().Sub(t))

	t = time.Now()
	fmt.Println("recursion WithTail:", fibWithTail(n, 1, 1))
	fmt.Println("cosume:", time.Now().Sub(t))

	t = time.Now()
	fmt.Println("No Recursion:", fibNoRecursion(n))
	fmt.Println("cosume:", time.Now().Sub(t))
}

var cache = map[int]int{1: 1, 2: 1}

// 带Cache的递归方式
func fibWithCache(n int) int {
	if val, ok := cache[n]; ok {
		return val
	}
	val := fibWithCache(n-1) + fibWithCache(n-2)
	cache[n] = val
	return val
}

// 尾递归方式
func fibWithTail(n, result1, result2 int) int {
	if n < 2 {
		return result1
	}
	return fibWithTail(n-1, result2, result1+result2)
}

// 纯递归方式
func fib(n int) int {
	if n <= 2 {
		return 1
	}
	return fib(n-1) + fib(n-2)
}

// 非递归方式
func fibNoRecursion(n int) int {

	cache := map[int]int{1: 1, 2: 1}
	for i := 3; i <= n; i++ {
		cache[i] = cache[i-1] + cache[i-2]
	}
	return cache[n]
}
```

对于fib（纯递归方式），运行结果是非常慢的，其原因是由于递归的层次越多，重复执行的函数调用就越多，从时间复杂度上讲这个属于O($2^N$)的时间复杂度 ,在N=50时候，运算结果如下：
```bash
recursion: 12586269025
cosume: 1m2.463850053s
recursion WithCache: 12586269025
cosume: 63.342µs
recursion WithTail: 12586269025
cosume: 3.479µs
No Recursion: 12586269025
cosume: 15.15µs
```
可见 纯递归的算法是如此之低效（因为涉及了太多的栈帧），而采用了尾递归方式，性能有极大的提升，这是因为尾递归只占用了一个栈帧就够了的原因 。
另外，对于不太好改写为尾递归和非递归的算法，可以考虑下cache模式，由于减少了大量的重复函数调用，Cache的递归算法还是非常棒的，几乎和递归的打平。

### 小结
递归是比较好用和常用的算法之一，但使用中可以尽可能的对其进行优化，尽可能的实现尾递归或增加cache机制，如果优化效果不明显，那就考虑采用非递归的方式。