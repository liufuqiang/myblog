---
title: "【译】用Go语言实现一个简单的EventBus模型"
date: 2019-07-17T15:02:44+08:00
---

 [原文](https://levelup.gitconnected.com/lets-write-a-simple-event-bus-in-go-79b9480d8997)

> EventBus 这里解释为事件总线， Event：事件  Bus：总线 ，为了避免译文会造成理解困惑，很多地方直接使用EventBus了。 \
callback: 回调\
slice: 切片

事件驱动的架构是计算机科学领域里高扩展性的一个典范,它可以让我们通过分布式和异步的方式处理事件，一个EventBus是通过pub/sub模式来实现的，即生产者去生成数据，而消费者订阅自己关心的数据，这样就可以实现了生产者和订阅者的解耦。生产者发布数据到EventBus里，而Bus负责把数据分发给它们的订阅者。

![EventBus](https://miro.medium.com/max/1022/1*6jeHWE0f2Mgd2CWJLTAZjg.png)

传统的实现EventBus的途径是通过callback（回调）的手段。 订阅者可以实现一个接口，EventBus通过这个接口来传递数据。
从go的并发模型上，我们可以看到channel可以被用来代替callback，这里我们就聚焦下如何用go的channels来实现一个EventBus。

> 我们聚焦在基于topic的事件总线上，发布者把数据发布到topic上，订阅者可以监听并获取到数据

## 定义数据结构

为了实现这个模型，我们需要定义一个传输数据的数据结构。 我们先用struct来简单创建一个新的数据类型，代码如下：
```golang
type DataEvent struct {
   Data interface{}
   Topic string
}
```

在这个struct里我们通过interface{}定义了一个底层的数据类型Data，这意味着它可以是任意的数据类型。 我们附加定义了一个叫Topic的成员,这是因为一个订阅者可以关注1个或多个topic，有了这个Topic成员就比较方便的区分出来这个数据到底是哪个topic的了。

## 引入channels

到目前为止我们已经定义了实现EventBus关键的数据结构，我们还需要一种传递数据的方式。为此我们可以定义一个**DataChannel**，用来传递**DataEvent**。

```golang
// DataChannel is a channel which can accept an DataEvent
type DataChannel chan DataEvent
// DataChannelSlice is a slice of DataChannels
type DataChannelSlice [] DataChannel
//DataChannelSlice was created for keeping a slice of channels and to reference them easily.
```


### Event Bus（事件总线）

```golang
// EventBus stores the information about subscribers interested for a particular topic
type EventBus struct {
   subscribers map[string]DataChannelSlice
   rm sync.RWMutex
}
```

**EventBus** 数据结构里有一个subscribers成员，是一个map类型，它的值是一个**DataChannelSlices**类型的数据. 我们同时定义了一个Mutex用来做并发场景时的读写锁机制。
通过使用一个map和定义好topic就可以比较容易的来组织事件了。 topic对应map里里的key，当有人向一个topic里发布数据的时候，我们可以容易的通过topic作为key在map里找到这个topic的数据，然后把数据投递到channel里做下一步的处理。

## 订阅一个topic

为了实现订阅一个topic，channel派上了用场，它的作用就好比是传统方式里的callback函数。 当一个发布者向topic里发布数据的时候，channel会接收到数据。
```golang
func (eb *EventBus)Subscribe(topic string, ch DataChannel)  {
   eb.rm.Lock()
   if prev, found := eb.subscribers[topic]; found {
      eb.subscribers[topic] = append(prev, ch)
   } else {
      eb.subscribers[topic] = append([]DataChannel{}, ch)
   }
   eb.rm.Unlock()
}
```

我们只需要简单的把监听者加入到channel切片里，为了保证并发安全，我们在操作前后会通过加锁和解锁来保护。

## 发布数据到topic

为了发布一个事件，发布者需要提供topic，以及要发送的数据，这个数据会以广播的形式传播给所以的订阅者。

```golang
func (eb *EventBus) Publish(topic string, data interface{}) {
   eb.rm.RLock()
   if chans, found := eb.subscribers[topic]; found {
      // this is done because the slices refer to same array even though they are passed by value
      // thus we are creating a new slice with our elements thus preserve locking correctly.
      channels := append(DataChannelSlice{}, chans...)
      go func(data DataEvent, dataChannelSlices DataChannelSlice) {
         for _, ch := range dataChannelSlices {
            ch <- data
         }
      }(DataEvent{Data: data, Topic: topic}, channels)
   }
   eb.rm.RUnlock()
}
```

如上代码所示，在这个方法里，我们首先检查对应的topic是否存在订阅者，如果存在订阅者，之后我们通过迭代chennel的切片可以获取到此topic对应的数据，并发送数据到topic里。

> 注意下，我们这里使用goroutine的方式来避免发布操作被阻塞。
## 来试一下吧

首先，我们需要创建一个EventBus的实例，真实情景下，你可以使用单例模式从一个包里导出一个EventBus对象。

```golang
var eb = &EventBus{
   subscribers: map[string]DataChannelSlice{},
}
```
为了测试刚创建的EventBus对象，我们实现一个方法来持续的发布消息到给定的topic里，这里会随机sleep一段时间

```golang
func publisTo(topic string, data string)  {
   for {
      eb.Publish(topic, data)
      time.Sleep(time.Duration(rand.Intn(1000)) * time.Millisecond)
   }
}
```

接下来，我们需要一个main函数，里边可以实现订阅topic的功能。 还要实现一个方法来帮助打印事件数据。

```golang
func printDataEvent(ch string, data DataEvent)  {
   fmt.Printf("Channel: %s; Topic: %s; DataEvent: %v\n", ch, data.Topic, data.Data)
}
func main()  {
   ch1 := make(chan DataEvent)
   ch2 := make(chan DataEvent)
   ch3 := make(chan DataEvent)
   eb.Subscribe("topic1", ch1)
   eb.Subscribe("topic2", ch2)
   eb.Subscribe("topic2", ch3)
   go publisTo("topic1", "Hi topic 1")
   go publisTo("topic2", "Welcome to topic 2")
   for {
      select {
      case d := <-ch1:
         go printDataEvent("ch1", d)
      case d := <-ch2:
         go printDataEvent("ch2", d)
      case d := <-ch3:
         go printDataEvent("ch3", d)
      }
   }
}
```

我们创建了3个channel来订阅topic，其中2个如ch2,ch3订阅了相同的topic。我们使用select语句来从channel里获取数据，获取到数据后使用goroutine的方式来打印数据。这个不是必要的，但是在某些场景下，收到事件数据后你可能会做非常重的操作，为了保护select操作不会被阻塞，我们还是需要使用goroutine比较好的。

输出数据如下：
```bash
Channel: ch1; Topic: topic1; DataEvent: Hi topic 1
Channel: ch2; Topic: topic2; DataEvent: Welcome to topic 2
Channel: ch3; Topic: topic2; DataEvent: Welcome to topic 2
Channel: ch3; Topic: topic2; DataEvent: Welcome to topic 2
Channel: ch2; Topic: topic2; DataEvent: Welcome to topic 2
Channel: ch1; Topic: topic1; DataEvent: Hi topic 1
Channel: ch3; Topic: topic2; DataEvent: Welcome to topic 2
...
```
通过输出你可以看到了EventBus使用channel来进行数据的分发了吧。

## 完整代码

```golang
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

type DataEvent struct {
	Data  interface{}
	Topic string
}

// DataChannel is a channel which can accept an DataEvent
type DataChannel chan DataEvent

// DataChannelSlice is a slice of DataChannels
type DataChannelSlice []DataChannel

// EventBus stores the information about subscribers interested for a particular topic
type EventBus struct {
	subscribers map[string]DataChannelSlice
	rm          sync.RWMutex
}

func (eb *EventBus) Publish(topic string, data interface{}) {
	eb.rm.RLock()
	if chans, found := eb.subscribers[topic]; found {
		// this is done because the slices refer to same array even though they are passed by value
		// thus we are creating a new slice with our elements thus preserve locking correctly.
		// special thanks for /u/freesid who pointed it out
		channels := append(DataChannelSlice{}, chans...)
		go func(data DataEvent, dataChannelSlices DataChannelSlice) {
			for _, ch := range dataChannelSlices {
				ch <- data
			}
		}(DataEvent{Data: data, Topic: topic}, channels)
	}
	eb.rm.RUnlock()
}

func (eb *EventBus) Subscribe(topic string, ch DataChannel) {
	eb.rm.Lock()
	if prev, found := eb.subscribers[topic]; found {
		eb.subscribers[topic] = append(prev, ch)
	} else {
		eb.subscribers[topic] = append([]DataChannel{}, ch)
	}
	eb.rm.Unlock()
}

var eb = &EventBus{
	subscribers: map[string]DataChannelSlice{},
}

func printDataEvent(ch string, data DataEvent) {
	fmt.Printf("Channel: %s; Topic: %s; DataEvent: %v\n", ch, data.Topic, data.Data)
}

func publisTo(topic string, data string) {
	for {
		eb.Publish(topic, data)
		time.Sleep(time.Duration(rand.Intn(1000)) * time.Millisecond)
	}
}

func main() {
	ch1 := make(chan DataEvent)
	ch2 := make(chan DataEvent)
	ch3 := make(chan DataEvent)

	eb.Subscribe("topic1", ch1)
	eb.Subscribe("topic2", ch2)
	eb.Subscribe("topic2", ch3)

	go publisTo("topic1", "Hi topic 1")
	go publisTo("topic2", "Welcome to topic 2")

	for {
		select {
		case d := <-ch1:
			go printDataEvent("ch1", d)
		case d := <-ch2:
			go printDataEvent("ch2", d)
		case d := <-ch3:
			go printDataEvent("ch3", d)
		}
	}
}
```

## 为什么使用channel代替callback？

传统的callback方式需要你去实现某种接口，比如：

```golang
type Subscriber interface {
   onData(event Event)
}
```

现在，如果你想要订阅到一个事件上，你需要去实现这个接口，以便EventBus可以通过这个方法传播它。

```golang
type MySubscriber struct {
}
func (m MySubscriber) onData(event Event)  {
   // do anything with event
}
```

而使用channel可以允许你在一个方法里的简单的注册一个订阅，而不需要去实现一个接口。

```golang
func main() {
   ch1 := make(chan DataEvent)
   eb.Subscribe("topic1", ch1)
   fmt.Println((<-ch1).Data)
   ...
}
```

## 总结

本文的目的是指出了一种实现EventBus模型的不同思路。
> 这个可能不是最佳的解决方案。
比如，channel会被阻塞一直到有人来消费他们为止，这期间消息就会一直堆积在内存里，这个是有较多限制的。

> 我这里使用了一个切片（slice)来存储所有的topic的订阅者,目的是可以让文章写的更简单一些。这里需要同一个集合（SET)来替换，这样就可以避免重复数据出现在队列里了。

传统的callback方式可以通过使用提到的相同的机制来简单的实现，你可以简单的通过使用goroutine来实现异步方式发布的封装。

