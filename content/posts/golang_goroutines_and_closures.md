---
title: "Go的闭包在并发下的问题"
date: 2018-08-20T11:40:02+08:00
tags: ["Go","goroutine","闭包"]
---
### 问题
在go里如果我们使用了闭包(closures)，那么在使用并发(goroutines)编程的时候会发生一些奇怪的问题。 看下面一段代码：



```golang
package main

import (
	"fmt"
)

func main() {
	done := make(chan bool)

	values := []string{"a", "b", "c"}
	for _, v := range values {
		go func() {
			fmt.Println(v)
			done <- true
		}()
	}

	// 等待所有的goroutines完成后再推出程序
	for _ = range values {
		<-done
	}
}
```

我们可能会认为这个程序的输出是 a 、 b、 c，但实际上输出是这样的：
```golang
$go run main.go
c
c
c
```
为什么会出现这样的情况呢？ 原因是，每一次循环迭代里都使用相同的变量v,所以每个闭包共享了同一个变量。当闭包执行的时候，当调用fmt.Println 函数的时候他会打印v的值，但是v的值可能被其他的goroutine修改掉了，如果希望能够诊断这类潜在问题或发现其他潜在风险，我们可以运行```go vet``来协助。
```golang
$go vet main.go
# command-line-arguments
./main.go:13: loop variable v captured by func literal
```

为了在闭包执行时候可以把当前的value绑定给他，那就必须在循环内部创建一个新的变量。一种方法是用参数把当前value传递给闭包函数。
```golang
  for _, v := range values {
        go func(u string) {
            fmt.Println(u)
            done <- true
        }(v)
    }
```
上边的代码示例是把v变量用参数形式传递给了匿名函数。在匿名函数内部v的值赋值给了新的变量u，这样打印的内容就是符合预期的了。

但是还有一种看起来奇怪但非常有用的方法，那就是在闭包函数（或匿名函数）被go关键字执行之前重新定义一个同名变量，如下：
```golang
	for _, v := range values {
		v := v //创建一个同名的变量，其实就是新变量了
		go func() {
			fmt.Println(v)
			done <- true
		}()
	}
```

这是一个小技巧，仅此而已。



