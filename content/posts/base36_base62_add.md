---
title: "关于一个进制的简单加法实现"
date: 2019-05-08T18:23:15+08:00
---

## 概述
日常我们接触较多的都是10进制、二进制、八进制、十六进制等，其实进制这个只是我们为了处理某类问题时候，考虑到方便度和人类友好度进而确定使用某种进制的。 比如计算机处理01的二进制是由于它的物理硬件特性的。

我们也曾经接触这样的场景，为了表达和传递不可读的数据，比如二进制数据做网络传输，最常用的方法就是把数据进行base64编码； 有时候为了对数据进行压缩，引入了较多进制的处理方法，比如Varints的base128等，为了节约存储可以把较大的hash值做成62进制的映射等等。

总之，进制只是一个为了某种便利性进行的一个引入，你可以去定义任意的进制，随心所欲，只有你觉得能够达到目的。

我们这里给出2个较为简单的36进制和62进制的一个加法过程。

所谓36进制，就是字符表为0-9a-z的，所谓的62进制，就是字符表为0-9a-zA-Z。

## Show me the code

话不多说，直接看代码吧

- 36进制的加法
```golang
package main

import (
	"fmt"
	"os"
	"strconv"
	"strings"
)

// table 0-9a-z

func main() {

	if len(os.Args) != 3 {
		panic("参数不对，应该是: ./xx 1x 2b")
	}

	fmt.Printf("%s+%s=%s\n", os.Args[1], os.Args[2], Add(os.Args[1], os.Args[2]))
}

func Add(num1, num2 string) string {
	if !checkInput(num1) || !checkInput(num2) {
		return "Error Input"
	}

	// 对异常数据做下快速处理
	if num1 == "" && num2 == "" {
		return ""
	}
	if num1 == "" && num2 != "" {
		return num2
	}
	if num2 == "" && num1 != "" {
		return num1
	}
	// 先把长度对齐，方便循环相加
	len1, len2 := len(num1), len(num2)

	if len1 > len2 {
		num2 = strings.Repeat("0", len1-len2) + num2
	} else {
		num1 = strings.Repeat("0", len2-len1) + num1
	}

	var result []string
	carry := 0

	var _getvalue = func(s rune) rune {
		if s >= '0' && s <= '9' {
			return s - '0'
		}
		return s - 'a' + 10
	}

	var _toChar = func(i int) string {
		if i >= 0 && i <= 9 {
			return strconv.Itoa(i)
		}
		return string('a' + i - 10)
	}

	var _add = func(n1, n2 rune, carry int) (ret string, c int) {
		sum := int(_getvalue(n1)) + int(_getvalue(n2)) + carry
		if sum >= 36 {
			c = 1
		}
		ret = _toChar(sum % 36)
		return
	}

	for i := len(num1) - 1; i >= 0; i-- {
		var tmp string
		tmp, carry = _add(rune(num1[i]), rune(num2[i]), carry)
		result = append(result, tmp)
	}
	if carry == 1 {
		result = append(result, "1")
	}
	var ret string
	for _, v := range result {
		ret = v + ret
	}
	return ret
}

func checkInput(str string) bool {
	if str == "" {
		return true
	}
	for _, v := range str {
		if (v >= '0' && v <= '9') || (v >= 'a' && v <= 'z') {
		} else {
			return false
		}
	}
	return true
}


```

- 62进制的加法

```golang
package main

import (
	"fmt"
	"os"
	"strconv"
	"strings"
)

// table 0-9a-zA-Z

func main() {

	if len(os.Args) != 3 {
		panic("参数不对，应该是: ./xx Z3 2b")
	}

	fmt.Printf("%s+%s=%s\n", os.Args[1], os.Args[2], Add(os.Args[1], os.Args[2]))
}

func Add(num1, num2 string) string {
	if !checkInput(num1) || !checkInput(num2) {
		return "Error Input"
	}
	// 对异常数据做下快速处理
	if num1 == "" && num2 == "" {
		return ""
	}
	if num1 == "" && num2 != "" {
		return num2
	}
	if num2 == "" && num1 != "" {
		return num1
	}
	// 先把长度对齐，方便循环相加
	len1, len2 := len(num1), len(num2)

	if len1 > len2 {
		num2 = strings.Repeat("0", len1-len2) + num2
	} else {
		num1 = strings.Repeat("0", len2-len1) + num1
	}

	var result []string
	carry := 0

	var _getvalue = func(s rune) rune {
		if s >= '0' && s <= '9' {
			return s - '0'
		}
		if s >= 'a' && 's' <= 'z' {
			return s - 'a' + 10
		}
		return s - 'A' + 36
	}

	var _toChar = func(i int) string {
		if i >= 0 && i <= 9 {
			return strconv.Itoa(i)
		}
		if i > 9 && i < 36 {
			return string('a' + i - 10)
		}
		return string('A' + i - 36)
	}

	var _add = func(n1, n2 rune, carry int) (ret string, c int) {
		sum := int(_getvalue(n1)) + int(_getvalue(n2)) + carry
		if sum >= 62 {
			c = 1
		}
		ret = _toChar(sum % 62)
		return
	}

	for i := len(num1) - 1; i >= 0; i-- {
		var tmp string
		tmp, carry = _add(rune(num1[i]), rune(num2[i]), carry)
		result = append(result, tmp)
	}
	if carry == 1 {
		result = append(result, "1")
	}
	var ret string
	for _, v := range result {
		ret = v + ret
	}
	return ret
}

func checkInput(str string) bool {
	if str == "" {
		return true
	}
	for _, v := range str {
		if (v >= '0' && v <= '9') || (v >= 'a' && v <= 'z') || (v >= 'A' && v <= 'Z') {
		} else {
			return false
		}
	}
	return true
}

```

这里要注意别把思路定为先把对应字符按照进制转为熟悉的10进制，做完加法后然后再转为对应进制，因为十进制能表达的范围太小了，支持不了任意长度的输入参数相加。


