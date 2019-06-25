---
title: "Go里的unsafe包是什么？"
author: "刘付强"
tags: ["go", "unsafe"]
date: 2019-06-24T15:01:08+08:00
---

> 注意：本文基于Go 1.12版本


从Unsafe这个包名上，我们会比较自然的意识是不用去使用它。 要去理解这个包可能不安全的原因，我们先来看下文档的描述吧：


>Package unsafe contains operations that step around the type safety of Go programs.\
Packages that import unsafe may be non-portable and are not protected by the Go 1 compatibility guidelines.\
unsafe 包包含了很多围绕go程序类型安全的操作。\
导入了unsafe的包可能会是不可移植的，也不受Go 1的兼容性原则保护。

因此，unsafe被用于和Go提供的安全类型相对立的一个名称了。让我们深入理解下文档中提到的2个要点。

## 类型安全

在Go语言里，每个变量都有一种类型，可以在分配给另一个变量之前转换为另一种类型。在转换期间，Go会对数据进行转换来适应新的期望的类型。这里有个例子：

```golang
var i int8 = -1 // -1 binary representation: 11111111
var j = int16(i) // -1 binary representation: 11111111 11111111
fmt.Println(i, j) // -1 -1

```

unsafe包可以给我们提供一种直接访问变量在内存中地址的方式，这里存放的是变量的原始二进制数据。 这样绕开了类型的约束，我们可以随意使用数据了：

```golang
var k uint8 = *(*uint8)(unsafe.Pointer(&i))
fmt.Println(k) // 255 is the uint8 value for the binary 11111111
```

在这里原始值被解释为uint8类型了。 



## Go 1 的兼容性原则

在Go 1的原则里明确解释过，如果Go升级改动后，unsafe包的使用会潜在的破坏你代码的兼容性：


>Packages that import unsafe may depend on internal properties of the Go implementation. We reserve the right to make changes to the implementation that may break such programs.

我们在头脑中要保持警惕，Go 1中，内部实现的改变可能会使我们的代码出问题。 而然Go的标准库里也在很多地方使用了unsafe包。


## 在反射包(reflection)里使用

relection包是我们日常使用较多的包。这个包是基于内部的一个空接口实现的。为了去读取数据，GO 必须把变量的数据转换为一个空接口，并通过将与空接口的内部表示相匹配的结构与指针地址处的内存映射来读取它们：

```golang
func ValueOf(i interface{}) Value {
   [...]
   return unpackEface(i)
}
// unpackEface converts the empty interface i to a Value.
func unpackEface(i interface{}) Value {
   e := (*emptyInterface)(unsafe.Pointer(&i))
   [...]
}
```

变量e现在包含了这个value的全部信息，比如像变量的类型或是否已经被导出等。 reflection包也使用unsafe包去直接通过操作内存来修改变量的value，如我们上边的例子。

## 在sync包里的使用

unsafe包另一个有趣的应用是在sync包里。

pools是在所有的 goroutines/processors 间通过访问一个内存段实现的共享池，所有的goroutines都可以通过它来访问unsafe包：

```golang
func indexLocal(l unsafe.Pointer, i int) *poolLocal {
   lp := unsafe.Pointer(uintptr(l) + uintptr(i)*unsafe.Sizeof(poolLocal{}))
   return (*poolLocal)(lp)
}
```
变量l是内存段地址，i是进程号。 indexLocal函数只需要读取这个内存段即可，这个段包括了X个（进程数量）poolLcal 的结构， 具有与其读取的索引相关的偏移量。 存储一个单独指针是指向整块内存段是一个实现共享内存池比较高效的方式。

## 在runtime包里的使用

Go同时在runtime里使用unsafe包，用来处理内存操作，如栈空间的申请和释放。 栈在它的结构上对应着2个边界：

```golang
type stack struct {
   lo uintptr
   hi uintptr
}
```
unsafe包会帮助实现内存操作：
```golang
func stackfree(stk stack) {
   [...]
   v := unsafe.Pointer(stk.lo)
   n := stk.hi - stk.lo
   // then memory freeing based on the pointer to the stack
   [...]
}
```
我们可以在自己的业务里使用unsafe包，比如实现两个struct的转换等。

## 开发中的使用

一个好的使用unsafe的场景是转换2个不同的struct，前提是2个结构在底层的数据布局是相同的。比如：
```golang
type A struct {
   A int8
   B string
   C float32

}

type B struct {
   D int8
   E string
   F float32

}

func main() {
   a := A{A: 1, B: `foo`, C: 1.23}
   //b := B(a) cannot convert a (type A) to type B
   b := *(*B)(unsafe.Pointer(&a))

   println(b.D, b.E, b.F) // 1 foo 1.23
}
```

source: https://play.golang.org/p/sjeO9v0T_Fs

另一个使用可以看下这个网站，[http://golang-sizeof.tips](http://golang-sizeof.tips), 这个工具可以帮助我们更好的理解struct结构里的内存布局。


总结下， unsafe包是非常有意思并且很强大的，但是使用它时候一定要多加小心。 如果需要更新包的特性，可以参考这里的[升级指南](https://github.com/golang/go/issues?utf8=%E2%9C%93&q=is%3Aopen+label%3AGo2+%22unsafe%22+in%3Atitle)


## 原文
[https://medium.com/@blanchon.vincent/go-what-is-the-unsafe-package-d2443da36350](https://medium.com/@blanchon.vincent/go-what-is-the-unsafe-package-d2443da36350)