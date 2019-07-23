---
title: "【译】Go项目开发里最常犯的10个错误"
date: 2019-07-18T15:46:28+08:00
---

> 以下是一篇译文，适当加入了我的一些观点，如有出入之处欢迎与我沟通。

原文:[https://itnext.io/the-top-10-most-common-mistakes-ive-seen-in-go-projects-4b79d4f6cd65](https://itnext.io/the-top-10-most-common-mistakes-ive-seen-in-go-projects-4b79d4f6cd65)

这个文章是列举了下我目前在go项目看到的最常犯错的10个错误，以下的顺序并不重要哦。

## 1、未知类型的枚举值

我们来看一个简单的例子：

```golang
type Status uint32

const (
	StatusOpen Status = iota
	StatusClosed
	StatusUnknown
)
```

这里我们使用iota定义了一组枚举变量表示结果的状态：

```golang
StatusOpen = 0
StatusClosed = 1
StatusUnknown = 2
```

现在，我们假设这个**Status**类型是一个JSON请求的一部分，它会被执行 marshall/ummarshall 操作。 我们可以设计如下的一个结构体：

```golang
type Request struct {
	ID        int    `json:"Id"`
	Timestamp int    `json:"Timestamp"`
	Status    Status `json:"Status"`
}
```

之后，收到的请求像这样：

```javascript
{
  "Id": 1234,
  "Timestamp": 1563362390,
  "Status": 0
}
```

到这里还没有什么特殊的地方，status会被unmarshall到**StatusOpen**上，对吗？

好了，我们在来看一个另外的请求数据，这次请求里status变量没有设置（先不纠结是什么原因了）：
```javascript
{
  "Id": 1235,
  "Timestamp": 1563362390
}
```

在这个情况下，Request结构体里的Status字段就会被初始化为他的零值（是一个uint32类型的：0）。 因此，**StatusOpen** 代替了 **StatusUnknow**。（按道理来说，如果请求里不传递status应该代表状态未知才对哦）

最佳的实践应该是给未知的值设置为枚举变量的0值：
```golang
type Status uint32

const (
	StatusUnknown Status = iota
	StatusOpen
	StatusClosed
)
```



这样的话，如果JSON请求里不传递status字段，它就会别初始化为 **StatusUnknow**,这样就符合我们的预期了。


## 2、基准化分析（Benchmarking）

要完成一个准确无误的基础测试是比较难的，有非常多的因素会影响测试结果。

一个比较常见的错误是会被编译器优化给愚弄。 让我们来看一个具体的示例吧，例子源于：[teivah/bitvector](https://github.com/teivah/bitvector)库
```golang
func clear(n uint64, i, j uint8) uint64 {
	return (math.MaxUint64<<j | ((1 << i) - 1)) & n
}
```

这个函数清除指定范围的bit位，我们可能会这样对它做基准测试：

```golang
func BenchmarkWrong(b *testing.B) {
    b.ResetTimer()
	for i := 0; i < b.N; i++ {
		clear(1221892080809121, 10, 63)
	}
}
```

在这个基准测试里，编译器会注意到clear函数是一个叶子函数（即它没有调用任何其他函数），因此编译器会把这个函数作为内联函数。 一旦这个函数被内联处理了，编译器会发现这里也没有任何副作用。所以，clear函数调用就会被简单的移除掉了，这会导致不准确的基准测试结果。

一个方法是可以设置一个全局变量来存储计算结果，如下代码所示：

```golang
var result uint64

func BenchmarkCorrect(b *testing.B) {
	var r uint64
    b.ResetTimer()
	for i := 0; i < b.N; i++ {
		r = clear(1221892080809121, 10, 63)
	}
	result = r
}
```

这样的话，编译器就不太确定这个函数调用是否会有副作用，因此它就不会做内联优化处理，基准测试得到的结果会比较正确了。

> 在我的环境下没有复现出来编译器内联优化的差别 :( , 在介绍一种可以编码编译器对函数做内联的黑科技吧：\
在函数上方可以加上 //go:noinline 即可，详细内容可以参考这个文章[Go’s hidden #pragmas
](https://dave.cheney.net/2018/01/08/gos-hidden-pragmas)

## 3、指针！到处都是指针

使用值传递来传递一个变量，会创建一个变量的副本（拷贝变量的值），而使用指针来传递变量的话，只会拷贝一个内存地址。

因此，使用指针做为值传递总是更快的，是这样吗？

如果你是这么认为的，请看下这个例子 [pointer_test.go](https://gist.github.com/teivah/a32a8e9039314a48f03538f3f9535537)
为了方便大家直观的阅读，我把代码放过来吧：
```golang
package main

import (
	"encoding/json"
	"testing"
)

type foo struct {
	ID            string  `json:"_id"`
	Index         int     `json:"index"`
	GUID          string  `json:"guid"`
	IsActive      bool    `json:"isActive"`
	Balance       string  `json:"balance"`
	Picture       string  `json:"picture"`
	Age           int     `json:"age"`
	EyeColor      string  `json:"eyeColor"`
	Name          string  `json:"name"`
	Gender        string  `json:"gender"`
	Company       string  `json:"company"`
	Email         string  `json:"email"`
	Phone         string  `json:"phone"`
	Address       string  `json:"address"`
	About         string  `json:"about"`
	Registered    string  `json:"registered"`
	Latitude      float64 `json:"latitude"`
	Longitude     float64 `json:"longitude"`
	Greeting      string  `json:"greeting"`
	FavoriteFruit string  `json:"favoriteFruit"`
}

type bar struct {
	ID            string
	Index         int
	GUID          string
	IsActive      bool
	Balance       string
	Picture       string
	Age           int
	EyeColor      string
	Name          string
	Gender        string
	Company       string
	Email         string
	Phone         string
	Address       string
	About         string
	Registered    string
	Latitude      float64
	Longitude     float64
	Greeting      string
	FavoriteFruit string
}

var input foo

func init() {
	err := json.Unmarshal([]byte(`{
    "_id": "5d2f4fcf76c35513af00d47e",
    "index": 1,
    "guid": "ed687a14-590b-4d81-b0cb-ddaa857874ee",
    "isActive": true,
    "balance": "$3,837.19",
    "picture": "http://placehold.it/32x32",
    "age": 28,
    "eyeColor": "green",
    "name": "Rochelle Espinoza",
    "gender": "female",
    "company": "PARLEYNET",
    "email": "rochelleespinoza@parleynet.com",
    "phone": "+1 (969) 445-3766",
    "address": "956 Little Street, Jugtown, District Of Columbia, 6396",
    "about": "Excepteur exercitation labore ut cupidatat laboris mollit ad qui minim aliquip nostrud anim adipisicing est. Nisi sunt duis occaecat aliquip est irure Lorem irure nulla tempor sit sunt. Eiusmod laboris ex est velit minim ut cillum sunt laborum labore ad sunt.\r\n",
    "registered": "2016-03-20T12:07:25 -00:00",
    "latitude": 61.471517,
    "longitude": 54.01596,
    "greeting": "Hello, Rochelle Espinoza!You have 9 unread messages.",
    "favoriteFruit": "banana"
  }`), &input)
	if err != nil {
		panic(err)
	}
}

func byPointer(in *foo) *bar {
	return &bar{
		ID:            in.ID,
		Address:       in.Address,
		Email:         in.Email,
		Index:         in.Index,
		Name:          in.Name,
		About:         in.About,
		Age:           in.Age,
		Balance:       in.Balance,
		Company:       in.Company,
		EyeColor:      in.EyeColor,
		FavoriteFruit: in.FavoriteFruit,
		Gender:        in.Gender,
		Greeting:      in.Greeting,
		GUID:          in.GUID,
		IsActive:      in.IsActive,
		Latitude:      in.Latitude,
		Longitude:     in.Longitude,
		Phone:         in.Phone,
		Picture:       in.Picture,
		Registered:    in.Registered,
	}
}

func byValue(in foo) bar {
	return bar{
		ID:            in.ID,
		Address:       in.Address,
		Email:         in.Email,
		Index:         in.Index,
		Name:          in.Name,
		About:         in.About,
		Age:           in.Age,
		Balance:       in.Balance,
		Company:       in.Company,
		EyeColor:      in.EyeColor,
		FavoriteFruit: in.FavoriteFruit,
		Gender:        in.Gender,
		Greeting:      in.Greeting,
		GUID:          in.GUID,
		IsActive:      in.IsActive,
		Latitude:      in.Latitude,
		Longitude:     in.Longitude,
		Phone:         in.Phone,
		Picture:       in.Picture,
		Registered:    in.Registered,
	}
}

var pointerResult *bar

func BenchmarkByPointer(b *testing.B) {
	var r *bar
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		r = byPointer(&input)
	}
	pointerResult = r
}

var valueResult bar

func BenchmarkByValue(b *testing.B) {
	var r bar
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		r = byValue(input)
	}
	valueResult = r
}

```

在这个基准测试例子里，对比了通过指针传递和值传递的方式来传递0.3KB大小的数据的区别。 0.3KB并不算太大的数据，但是这个和我们日常用到的数据结构是差不多的了（比较有代表性了）。

当我在我本地执行这个基准测试的时候，值传递比指针传递要**快4倍**。这个和结果是非常违反我们直觉的，是不是呀？

要解释这个结果，就涉及到了Go语言是如何进行内存管理的了。我不能像  [William Kennedy](https://www.ardanlabs.com/blog/2017/05/language-mechanics-on-stacks-and-pointers.html) 解释的那么精彩和详尽，但我来做下简述吧。

一个变量可以在堆(heap)和栈(stack)上进行空间分配:

- 栈空间上包含了依托在一个goroutine是的 **ongoing** 的变量，一旦函数返回后，变量就会从栈里弹出。
- 堆空间包含的是共享的变量（如全局变量等）

让我们用一个例子来检查下我们是在哪里返回的值:
```golang
func getFooValue() foo {
	var result foo
	// Do something
	return result
}
```
如上代码所示， 一个 result的变量在当前的goroutine里被创建出来，这个变量被压入到当前的栈里，一旦这个函数返回后，调用端可以收到一个该变量的一个拷贝，而这个变量本身会从栈里弹出。而这个被弹出的变量直到它被其他的变量擦除之前仍然在内存里存在，但是它不能被再次访问的到了。

现在再来看一个使用指针的例子：

```golang
func getFooPointer() *foo {
	var result foo
	// Do something
	return &result
}
```
从上边代码可以看到 result变量还是在当前的goroutine里被创建，但是调用者会收到一个指针（这个变量地址的拷贝），如果result变量被栈弹出掉了，调用这个函数的调用者就再也不能访问到它了。
在这种情景下，Go的编译器会把这个变量逃逸到一个可以共享的内存空间，即堆(heap)。 这里可以去了解下[逃逸分析](https://studygolang.com/articles/12444)

另一种指针传递的情况，如下：
```golang
func main()  {
	p := &foo{}
	f(p)
}
```

因为我们是在相同的goroutine里调用f函数，变量 p 不需要被逃逸，它会被压入到栈里，子函数可以正常的访问到它。

例如， 这是io.Reader的Read方法接受一个切片而不是返回一个切片的一个重要原因。如果是返回一个切片（这里是个指针）就会造成这个变量会逃逸到堆空间。

```golang
type Reader interface {
        Read(p []byte) (n int, err error)
}
```
现在的问题是，为什么栈是更快的呢？ 这里有2个原因：

- 在栈空间上的变量是自动回收的，因此它不需要垃圾回收(GC),如我们前边说的，一个变量一旦它被创建时候会简单的压栈，而一旦它所在的函数返回的时候就会从栈里弹出，这里就不需要复杂的过程来进行未使用变量的回收等工作了
- 由于栈是隶属于一个goroutine的，因此和堆上存储的变量相对比，栈上存储的变量就不需要去做同步处理了，这一点也是栈在性能上提升的很重要的。

总结来说，当我们创建了一个函数，我们缺省的行为应该是使用值代替指针。 指针应该仅仅在我们希望去共享一个变量的时候才使用。

再补充一点，如果我们遇到了性能问题， 一个优化点是检查一下是否有指针误用的情况，通过以下的命令我们可以知道编译器是否对变量进行了逃逸处理：

```bash
go build -gcflag "-m -m"
```
最后说一句，在我们日常的开发中，值传递是最好的选择。

[这里](https://www.youtube.com/watch?v=ZMZpH4yT7M0&list=WL&index=5&t=152s)是Jacob Walker 在GopherConn 2019的一个topic，想更多了解stack和heap的可以看一下。

## 4、中断一个 for/switch 或 for/select

大家来猜一下，下面代码里的f函数如果返回true的话，会发生什么情况？
```golang
for {
  switch f() {
  case true:
    break
  case false:
    // Do something
  }
}
```
我们会走到break语句上，但是，这里的break只会中断switch语句，而不是for循环。

相同的问题，如下：

```golang
for {
  select {
  case <-ch:
  // Do something
  case <-ctx.Done():
    break
  }
}
```
这里的break也只会中断select语句，而不是for循环。

如果要去中断for/switch或for/select 的循环，一个可行的方案是使用标签break，比如这样：

```golang
loop:
	for {
		select {
		case <-ch:
		// Do something
		case <-ctx.Done():
			break loop
		}
	}
```

## 5、错误管理

Go语言在对待错误处理上还是有些年轻，如果说这个是Go 2里最期待的特性之一的话，这并不是个恰合。

当前的标准库(Go1.13之前)仅仅提供了一些构造错误的函数，你可以到[pkg/errors](https://github.com/pkg/errors)来看一下.

这个库是一个遵守如下经验法则的好方法，但它其实并不总是可以被遵守的：

> 一个错误应该只被处理一次，日志记录一个错误就是在处理一个错误，殷超一个错误要么被日志记录要么被传播到下个阶段去。

使用当前的标准库，我们是很难遵守这个原则的，因为我们希望去增加一些上下文到错误上，并且可能是具有某种层级关系的形式。

让我们看一个期望通过REST调用导致数据库问题的示例:

```bash
 unable to server HTTP POST request for customer 1234
 |_ unable to insert customer contract abcd
     |_ unable to commit transaction
```

如果我们使用 pkg/erros库，我们可能会这么做：

```golang
func postHandler(customer Customer) Status {
	err := insert(customer.Contract)
	if err != nil {
		log.WithError(err).Errorf("unable to server HTTP POST request for customer %s", customer.ID)
		return Status{ok: false}
	}
	return Status{ok: true}
}

func insert(contract Contract) error {
	err := dbQuery(contract)
	if err != nil {
		return errors.Wrapf(err, "unable to insert customer contract %s", contract.ID)
	}
	return nil
}

func dbQuery(contract Contract) error {
	// Do something then fail
	return errors.New("unable to commit transaction")
}
```

最初的error（如果不是从第三方库里返回的）可以是用errors.New方法来创建。中间层 insert 方法包裹了这个错误，并且追加了更多的上下文到这个error上，最后，上层的调用方通过记录日志来处理了这个错误，每个层级要么返回要么处理了这个错误。

我们可能想知道引起错误的原因从而做进一步的处理，比如说去做一次重试操作。比如这样的情况，我们有一个从外部第三方库里引入的db包，用来处理数据库的访问，这个库可能返回一个临时的error，叫做db.DBError,为了确定要不要去做重试操作，我们必须要对错误的原因进行检查：

```golang
func postHandler(customer Customer) Status {
	err := insert(customer.Contract)
	if err != nil {
		switch errors.Cause(err).(type) {
		default:
			log.WithError(err).Errorf("unable to server HTTP POST request for customer %s", customer.ID)
			return Status{ok: false}
		case *db.DBError:
			return retry(customer)
		}

	}
	return Status{ok: true}
}

func insert(contract Contract) error {
	err := db.dbQuery(contract)
	if err != nil {
		return errors.Wrapf(err, "unable to insert customer contract %s", contract.ID)
	}
	return nil
}
```

以上代码里使用的errors.Cause 是由第三方包提供的 [github.com/pkg/errors](https://github.com/pkg/errors).

我看到一个常见错误是部分的使用了pkg/errors包，例如用以下的方式进行错误检查:

```golang
switch err.(type) {
default:
  log.WithError(err).Errorf("unable to server HTTP POST request for customer %s", customer.ID)
  return Status{ok: false}
case *db.DBError:
  return retry(customer)
}
```
在这个例子里，如果**db.DBError是被包裹的，这里的switch永远不会进入此分支，即retry永远不会被触发。


## 6、切片初始化

有时候我们知道切片的最终长度是多少，比如这个场景，假如我们想要把一个切片Foo转换为另一个切片Bar，这就意味着着2个切片将会是同样的长度。

我经常看到用这样的方式进行切片的初始化：

```golang
var bars []Bar
bars := make([]Bar, 0)
```

切片并不是一个多么神奇的数据结构，在底层，切片实现了一个增长策略，当在当前切片没有足够的空间的时候会自动增长，在这个情况发生时，一个新的切片变量会被自动创建出来（这个数组将会有更大的容量），之后原切片的所有数据会被拷贝到新的切片上。

现在，让我们想象一下，如果我们需要多次的对我们包含成千上万元素的切片变量Foo执行重复的增长操作，插入操作的时间复杂度仍然是O(1)，但实际上这个会严重的影响程序的性能。

因此，如果我们知道最终的长度，我们可以这么来搞：

- 初始化时候指定一个预定义的长度
```golang
func convert(foos []Foo) []Bar {
	bars := make([]Bar, len(foos))
	for i, foo := range foos {
		bars[i] = fooToBar(foo)
	}
	return bars
}
```
- 或者初始化它时候给指定0长度和预定义的容量
```golang
func convert(foos []Foo) []Bar {
	bars := make([]Bar, 0, len(foos))
	for _, foo := range foos {
		bars = append(bars, fooToBar(foo))
	}
	return bars
}
```

到底哪个才是最佳的方式呢？ 第一个方式会稍微快一些的。然而你可能更喜欢选择第二个方式，因为它更加有一致性：即无论我们是否知道切片的大小，我们在切片尾部追加元素都可以使用append方法来完成。

## 7、Context管理

context.Context 是一个经常被开发者误解的对象，根据官方文档描述：

> A Context carries a deadline, a cancelation signal, and other values across API boundaries.

这个描述非常通用，通用的足以让很多人对为什么要使用和如何去使用context感到非常疑惑。

下面让我们来详细的分解一下，一个Context可以包含：

- deadline（最终期限）， 它可以指一个期限（如250ms）或是一个日期（如2019-01-08 01:00:00),这期限或日期是表示当它到达的时候我们必须取消一个正在执行的活动（一个I/O请求，等待一个channel的输入等）
- cancelation signal（取消信号，基本上是 <-chan struct{} ），这里的行为和之前的是相似的，一旦我们收到一个信号，我们必须停止正在执行的活动。比如，假如我们收到2个请求，一个是插入一些数据，另一个是取消第一个请求（2个请求是不相关的），这个可以通过在第一个请求调用里使用一个可以取消的context，一旦我们收到第二个请求的时候就可以调用这个context发送信号，进而让第一个请求停止执行。
- 一个key/value对的列表（都是 interface{}类型）

有2点需要补充下：

- 第一，context是可以组合的，因此我们可以有一个即包含了deadline，也包含了一个key/value列表的context。
- 第二，多个goroutine可以共享一个相同的context，因此，一个取消信号可能会导致多个活动停止执行。

言归正传，这里是一个我见过的具体错误示例。

一个Go应用是基于 [urfave/cli](https://github.com/urfave/cli) (这是go语言里一个非常好用的创建命令行应用的第三方库)，一旦启动，开发人员就会继承一种应用的context，这就意味着当应用停止的时候，这个库会发送一个取消信号。

我遇到的情况是，在我调用一个gRPC服务的时候，这个context被直接传递过去了，这并不是我们所期望的。（因为这个context从cli库里继承来的，里边有不符合预期的内容，会引起其他问题的）
相反，我们希望指示gRPC库： 请在程序停止或者100ms以后取消这个请求。为此，我们可以简单的创建一个组合的context，如果parent(父级)是cli库应用的名字，我们可以简单的这样做：

```golang
ctx, cancel := context.WithTimeout(parent, 100 * time.Millisecond)
response, err := grpcClient.Send(ctx, request)
```
上下文的理解并不复杂，在我看来，context是go语言的最佳特性之一。

延伸阅读：

- [Understanding the context package in golang
](http://p.agnihotry.com/post/understanding_the_context_package_in_golang)

- [gRPC and Deadlines](https://grpc.io/blog/deadlines/)

## 8、不使用 -race 选项

我经常见到的一个错误是在测试go应用的时候没有带 -race 选项。

正如这篇[报告](https://blog.acolyer.org/2019/05/17/understanding-real-world-concurrency-bugs-in-go)所描述的，虽然Go是“旨在使并发编程变得更容易，更不易出错”，但实际上我们仍然会遭遇很多并发的问题。

显然，Go的竞争检查(race detector)无法解决每一个并发问题，然而它依然是一个有价值的工具，我们应当确保在做测试的时候（go test)始终使用它。

延伸阅读

[Does the Go race detector catch all data race bugs?
](https://medium.com/@val_deleplace/does-the-race-detector-catch-all-data-races-1afed51d57fb)


## 9、使用文件名作为输入参数

另一个常见的错误是使用文件名作为函数的输入参数。

假设我们要实现一个函数，来统计一个文件里空行的数量，最常见的实现方式大概是这样的：

```golang
func count(filename string) (int, error) {
	file, err := os.Open(filename)
	if err != nil {
		return 0, errors.Wrapf(err, "unable to open %s", filename)
	}
	defer file.Close()

	scanner := bufio.NewScanner(file)
	count := 0
	for scanner.Scan() {
		if scanner.Text() == "" {
			count++
		}
	}
	return count, nil
}
```

文件名是作为参数形式传递给函数的，所以，我们打开文件并实现我们的逻辑，有问题吗？

现在，假设我们要基于这个函数实现一个单元测试，来分别测试普通的文件，空文件，和使用不同编码格式的文件等等，很容易变的难以管理。

此外，如果我们要实现相同的逻辑（计算空行数量）但这次是针对的内容是HTTP包体（body），我们就必须在去实现另一个函数来满足了。

Go语言里带有两个非常棒的抽象对象： io.Reader和io.Writer，我们可以简单的传递一个io.Reader对象来代替传递文件名，这样的话就更具通用性了。

数据源是一个文件？是一个HTTP的包体？还是一个字节buffer？这些都不重要，我们可以使用相同的读取方法实现对内容的读取操作。

在我们的例子里，我们甚至可以使用缓存输入进而进行按行读取，因此我们可以使用bufio.Reader和它的ReadLine方法：

```golang
func count(reader *bufio.Reader) (int, error) {
	count := 0
	for {
		line, _, err := reader.ReadLine()
		if err != nil {
			switch err {
			default:
				return 0, errors.Wrapf(err, "unable to read")
			case io.EOF:
				return count, nil
			}
		}
		if len(line) == 0 {
			count++
		}
	}
}
```

打开文件的职责就委托给count的调用端了(client),如：

```golang
file, err := os.Open(filename)
if err != nil {
  return errors.Wrapf(err, "unable to open %s", filename)
}
defer file.Close()
count, err := count(bufio.NewReader(file))
```

可以看到我们第二次的实现方式，无论数据源是什么样的，都可以调用这个函数，同时也使我们单元测试函数更加简单，因为我们可以简单的从一个字符串创建一个bufio.Reader即可。

```golang
count, err := count(bufio.NewReader(strings.NewReader("input")))
```

## 10、goroutines和循环变量

最后一个常见错误是使用循环变量的方式创建goroutine。

先来猜测下一些代码的输出结果是什么？

```golang
ints := []int{1, 2, 3}
for _, i := range ints {
  go func() {
    fmt.Printf("%v\n", i)
  }()
}
```

无论如何都会按序输出：1，2，3？ 大错特错了

在这个例子里，每个goroutine都会共享相同的循环变量，所以它会输出3，3，3（大概率会这样）

有2种方案来解决这个问题，第一种是把变量传递到闭包里（内部的函数）：

```golang
ints := []int{1, 2, 3}
for _, i := range ints {
  go func(i int) {
    fmt.Printf("%v\n", i)
  }(i)
}
```

第二种方式是在for循环里（作用域）创建另一个变量：

```golang
ints := []int{1, 2, 3}
for _, i := range ints {
  i := i
  go func() {
    fmt.Printf("%v\n", i)
  }()
}
```

```i := i``` 这样的赋值看起来有些奇怪，但这个真的是非常有效的。 进入循环体力就意味着进入了一个新的作用域，因此 ```i:=i`` 创建了一个新的变量实例，名字也是i. 当然为了提高可读性，我们也可以使用其他的名称。

延伸阅读

[CommonMistakes](https://github.com/golang/go/wiki/CommonMistakes)



