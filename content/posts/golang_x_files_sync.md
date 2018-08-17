---
title: "【译】Golang x-files 里的rsync介绍（并发问题的解决方案）"
date: 2018-08-17T17:23:12+08:00
---


> x-files 指的是 golang.org/x/ 下的官方辅助包，本文我们介绍下 golang.org/x/sync 这个包。[原文地址](https://rodaine.com/2018/08/x-files-sync-golang/)

go语言里使用go关键字就可以轻松搞定并发的问题，它讲究的是“不要通过共享内存来通信，而应该通过通信来共享内存”的原则，在Go里channel是个线程安全的多线程通讯介质。Go本身的这些原语基本上可以解决常见的各种需求了。

但是有些时候在某些情况下，面对并发问题或在错误处理上需要引入更多的协调手段，比如，goroutines会去访问一个非线程安全的资源（如map,slice等）。Go的标准库给我们提供了WaitGroup, Once, 和 Mutex， 如果再野一点还可以用 sync/atomic(原子操作包，偏底层，特殊场景的时候用到)，这些工具结合上chanel确实可以解决并发场景下数据竞争的问题，但是这样做多少回让我们的实现变的比较繁琐（死板），难道就不能对这样的需求做下更优雅的抽象吗？

非常不幸，Go的sync包不提供更高层次的模式和封装来解决我们的痛点。但有个非常好的消息是，golang.org/x/sync 把标准库没去做的事情给实现了。 本文我就对这个辅助库进行介绍，会涉及 errgroup、semaphore、singleflight、syncmap 4个工具的举例阐述，介绍他们如何解决常见的并发场景的需求。

## Action/Executor 模式

Action/Executor 模式出自Gang of Four的设计模式里的一种，也被叫做命令模式，是非常强大的一个设计模式。它从实际运行的方式里（Excutor)里抽象出了行为（Action),以下是这2个组件的基本接口：

```
// 一个Action接口，带一个Execute方法
type Action interface {
	// Execute 负责执行任务，这个方法会用context做参数，以便可以提供取消执行的功能
	Execute(ctx context.Context) error
}

// 一个Executor接口， 有个Execute方法可以执行多个Action。它根据实现的类型来定义action的并发、打开/关闭的失败行为.
type Executor interface {
	// Execute 执行参数里提供的所有的action，通过调用他们对应的Execute方法。这个方式也会传递一个context用于终止操作。
	Execute(ctx context.Context, actions []Action) error
}

// ActionFunc 本类型定义如同Action接口的Execute方法，这就意味着可以运行使用一个独立的函数来充当Action 
type ActionFunc func(context.Context) error

// Execute 实现了Action接口，把调用委托给了调用对象实现的函数
func (fn ActionFunc) Execute(ctx context.Context) error { return fn(ctx) }
```

Action接口类型执行具体性的任务，Executro负责把一组Action放在一起运行。创建一个Executor序列只是表面上的好处。但这会使并发的实现变的更有用，但是也需要更多的技巧。也就是说，Executor需要很好的处理好错误且把Actions的生命周期同步好，这是至关重要的。

## 用errgroup 来解决错误传播和取消
在我一开始写Go程序的时候，我非常迷信和滥用了Go的并发特性。我当时一时兴奋，却没有考虑到如果有多个goroutine有错误产出的时候会发生什么事情。

捕获到第一个错误后取消之后的goroutine执行，这是较为常见的处理模式。一个并行的Executor会被叠加到一起，使用一组 WaitGroups和channel来完成协调。 另一个优雅的实现方式就是使用 errgroup包来解决，如下代码：

```
// Parallel 是一个Executor的并行实现
type Parallel struct{}

// Execute方法并行的执行多个Action。如果第一个错误产生或者ctx被取消都会关闭执行
func (p Parallel) Execute(ctx context.Context, actions []Action) error {
	grp, ctx := errgroup.WithContext(ctx)

	for _, a := range actions {
		grp.Go(p.execFn(ctx, a))
	}

	return grp.Wait()
}

// execFn 将Context和Action绑定到errgroup.Group正确的函数签名上
func (p Parallel) execFn(ctx context.Context, a Action) func() error {
	return func() error { return a.Execute(ctx) }
}
```


...待续...

