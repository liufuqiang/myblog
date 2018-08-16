---
title: "Go语言世界里的HTTP2的探索"
date: 2018-08-16T10:26:13+08:00
draft: false
---

Go语言的标准库HTTP默认支持HTTP/2，它有非常多的文档和非常棒的代码实例。 在这篇文章里我会首先介绍一些go的HTTP/2的服务端新特性，并介绍客户端如何去使用这些新的特性。 之后我会推介一下[h2conn](https://github.com/posener/h2conn)项目，这是一个可以比较简单优雅的实现HTTP/2全双工通讯的工具库。

## HTTP/2 服务端
首先让我们用Go语言搭建一个HTTP/2的服务端吧！ 根据[HTTP2文档](https://godoc.org/golang.org/x/net/http2)描述的，所有的事情都是自动配置的，我们甚至不需要导入go的标准库 **http2包**：
> http2包是一个底层实现，它很少被人直接拿来使用。很多使用者会通过自动使用net/http包来间接的使用它。

HTTP/2是强制使用TLS(传送安全层)的。 为了能够完成需求我们需要一个私钥（private key)和一个证书（certificate)。 在Linux上，产生私钥的方法如下。运行以下命令并且按照提示执行操作即可。
```
openssl req -newkey rsa:2048 -nodes -keyout server.key -x509 -days 365 -out server.crt
```
这个命令会产生2个文件： server.key和server.crt
1. server.key: 包含我们服务端的私钥，在线上产品里请务必保持此文件的安全性和隐私性。这个key会用来对HTTPS的响应内容做加密处理，加密的内容可以用服务端的公钥（public key)解密。
2. server.crt: 服务端的证书。该文件可以代表着服务端的身份标识，并包含了服务端的公钥。这个文件可以被公开分享，它的内容也会在服务端和客户端做TLS握手的时候发送给客户端。

现在，我们看下服务端的实现代码，我们会使用go的标准库 HTTP server， 并依赖刚刚产生的SSL文件（私钥和正式）启用TLS模式。

```
package main

import (
	"log"
	"net/http"
)

func main() {
	// 创建一个 8000端口的服务
	// 方式如同创建 HTTP/1.1 
	srv := &http.Server{Addr: ":8000", Handler: http.HandlerFunc(handle)}

	// 因为我们使用的HTTP/2,所以需要用TLS启动服务
	// 和用 HTTP/1.1 启用 TLS 连接一样
	log.Printf("Serving on https://0.0.0.0:8000")
	log.Fatal(srv.ListenAndServeTLS("server.crt", "server.key"))
}

func handle(w http.ResponseWriter, r *http.Request) {
	// 把请求的协议log下
	log.Printf("Got connection: %s", r.Proto)
	// 发送消息给客户端
	w.Write([]byte("Hello"))
}
```

不用TLS不可以吗？ 也不是，H2C（ HTTP/2 Cleartext)协议就是不基于TLS的。Go的标准库需要到原计划是1.12才支持这个协议，之后又放弃了对H2c的支持。目前扩展包[x/net/http2/h2c](https://github.com/golang/net/blob/master/http2/h2c) 已经可以使用了，也是官方持续推荐的包了。

## HTTP/2 客服端
在Go语言里， 标准的 http.Client同样适用于HTTP/2的请求。 唯一的区别是在client的Transport字段里用http2.Transport 代替掉 http.Transport。
我们在服务端产生的证书是“自签”的，这就意味着它不是一个知名的证书授权机构（CA）签署的，这会导致我们的客户端不信任这个证书。


```
package main

import (
	"fmt"
	"net/http"
)

const url = "https://localhost:8000"

func main() {
	_, err := http.Get(url)
	fmt.Println(err)
}
```
让我们来运行下这个程序：

```
$ go run h2-client.go 
Get https://localhost:8000: x509: certificate signed by unknown authority
```
在服务端的日志里，我们同样会看到这样的一个错误：
```
http: TLS handshake error from [::1]:58228: remote error: tls: bad certificate
```
要解决这个问题， 我们可以在客户端定制下TLS的配置。我们可以把服务端的证书文件加入到客户端的“证书池”里，这样即使这个证书不是权威的CA机构颁发的，客户端也会信任这个证书了。
我们同样也会增加一个选项，可以通过命令行参数的方式，决定是选择HTTP/1.1协议还是HTTP/2协议。

```
package main

import (
	"crypto/tls"
	"crypto/x509"
	"flag"
	"fmt"
	"io/ioutil"
	"log"
	"net/http"

	"golang.org/x/net/http2"
)

const url = "https://localhost:8000"

var httpVersion = flag.Int("version", 2, "HTTP version")

func main() {
	flag.Parse()
	client := &http.Client{}

	// 用服务端的证书来创建一个证书池，因为证书不是权威CA颁发的
	caCert, err := ioutil.ReadFile("server.crt")
	if err != nil {
		log.Fatalf("Reading server certificate: %s", err)
	}
	caCertPool := x509.NewCertPool()
	caCertPool.AppendCertsFromPEM(caCert)

	// 用服务端的证书来创建TLS配置
	tlsConfig := &tls.Config{
		RootCAs: caCertPool,
	}

	// 在client端选择使用合适的传输协议
	switch *httpVersion {
	case 1:
		client.Transport = &http.Transport{
			TLSClientConfig: tlsConfig,
		}
	case 2:
		client.Transport = &http2.Transport{
			TLSClientConfig: tlsConfig,
		}
	}

	// 执行请求
	resp, err := client.Get(url)
	if err != nil {
		log.Fatalf("Failed get: %s", err)
	}
	defer resp.Body.Close()
	body, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		log.Fatalf("Failed reading response body: %s", err)
	}
	fmt.Printf(
		"Got response %d: %s %s\n",
		resp.StatusCode, resp.Proto, string(body))
}
```
这一次我们得到了正确的结果：
```
$ go run h2-client.go 
Got response 200: HTTP/2.0 Hello
```
在服务端的日志里我们看到了正确的日志行：```2018/08/13 16:26:18 Got connection: HTTP/2.0``` 

但是当我们使用HTTP/1.1传输协议的时候会发生什么呢？
```
$ go run h2-client.go -version 1
Got response 200: HTTP/1.1 Hello
```
我们发现服务端并没有对HTTP/2做什么特殊的处理，所以它依然是支持HTTP/1.1协议连接的。这一点对向后兼容是非常关键的。另外，服务的日志表明了连接是HTTP/1.1:```2018/08/13 16:28:08 Got connection: HTTP/1.1```


## HTTP/2 高级特性
我们创建了一个HTTP/2的C-S（client-server)连接，并且我们可以享受到HTTP/2协议带来的红利---安全和高效率的连接。HTTP/2有很多新的特性，让我们来以此研究下他们吧！

### 服务端推送技术（Server Push）
HTTP/2允许服务端推送，这个技术就是“使用给定的目标来构建一个合成的请求”。
这个在服务端的handler里可以比较容易就可以实现：

```
func handle(w http.ResponseWriter, r *http.Request) {
	// 把请求协议记到log
	log.Printf("Got connection: %s", r.Proto)

	// 捕获 第二个请求， 必须是为避免递归调用而在之前push的消息。
	// 不要担心， Go会帮助我们解决递归push的问题的
	if r.URL.Path == "/2nd" {
		log.Println("Handling 2nd")
		w.Write([]byte("Hello Again!"))
		return
	}

	// 处理第一次的请求
	log.Println("Handling 1st")

	// 服务端push必须是在body响应之前发起。为了检测连接是否支持push技术，我们应该在response Writer上使用类型断言。
	// 如果连接不支持push，或者是puhs失败了，我们就直接忽略掉push。服务端的push技术仅仅是为了提高HTTP/2客户端性能的。
	pusher, ok := w.(http.Pusher)
	if !ok {
		log.Println("Can't push to client")
	} else {
		err := pusher.Push("/2nd", nil)
		if err != nil {
			log.Printf("Failed push: %v", err)
		}
	}

	// 发送响应主体
	w.Write([]byte("Hello"))
}
```
**一句话聊一下http.Pusher的实现**：我必须承认用类型断言的来判断连接是否支持服务端push的特性的设计是比较奇怪的选择，我也搞不太清楚。我就假设他是为了向后兼容Go1.1而设计的，但我想知道是否有一种更加优雅的方式去实现兼容问题。http.Flusher的实现也是如此，下面会讨论到。


### 服务端推送的客户端处理方式（Consuming Server Push）
我们再次启动服务端程序，并且测试下客户端程序。
HTTP/1.1的客户端：
```
$ go run ./h2-client.go -version 1
Got response 200: HTTP/1.1 Hello
```

服务端日志:
```
2018/08/13 16:52:11 Got connection: HTTP/1.1
2018/08/13 16:52:11 Handling 1st
2018/08/13 16:52:11 Can't push to client
```

HTTP/1.1的客户端传输协议在http.ResonseWriter上的连接结果没有实现http.Pusher接口，这个是符合预期的。在我们服务端代码里我们是可以选择什么样的客户端类型我们去做什么事情。

HTTP/2 的客服端:
```
go run ./h2-client.go -version 2
Got response 200: HTTP/2.0 Hello
```
服务端日志：
```
2018/08/13 16:52:15 Got connection: HTTP/2.0
2018/08/13 16:52:15 Handling 1st
2018/08/13 16:52:15 Failed push: feature not supported
```
这就奇怪了， 我们客户端使用HTTP/2的传输协议也仅仅得到了第一个请求的内容“Hello”的响应。服务端日志表明这个链接实现了http.Pusher的接口，但是当我们真的去调用Push()函数的时候它失败了。
我发现这个[StackOverflow](https://stackoverflow.com/questions/43852955/how-can-i-read-http-2-push-frames-from-a-net-http-request)讨论帖里有关于如何去实现go客户端支持服务端推送的例子。显然，HTTP/2的客户端传输协议硬编码设置了 HTTP/2的标识位，这个标识位表明要禁用掉推送功能。这里是关于这个bug的 [Github issue](https://github.com/golang/go/issues/18594),这个是关于这个bug的[改进提议](https://go-review.googlesource.com/c/net/+/85577)，但目测已经沉寂了很长时间了。

> 关于go硬编码导致的go client不支持server Push的实现如下，另外到目前最新的 go1.10.3版本里依然没有解决 (src/net/http/h2_bundle.go)

```
    initialSettings := []http2Setting{
    		{ID: http2SettingEnablePush, Val: 0},
    		{ID: http2SettingInitialWindowSize, Val: http2transportDefaultStreamFlow},
    	}
 ```

因此，目前我们就不要指望可以通过Go client的方式来测试服务端推送的新特性了。
最为另一个选择，我们可以使用chrome浏览器来作为client，它可以帮我们测试服务端推送的新特性。

![image](http://p6.qhimg.com/d/inn/74075fd1/chrome-http2-not-secured.png)
![image](http://p6.qhimg.com/d/inn/8bee253c/chrome-http2-hello.png)
> 因为我们是自签的证书，chrome会提示证书风险，我们需要在高级菜单里选择继续打开页面即可。

服务端日志有比较精确的显示，即使客户端实际上就请求了 path=/ 一次，但handler 被调用了2次， 分别是 path=/ 和 path=/2nd。

```
2018/08/13 17:41:50 Got connection: HTTP/2.0
2018/08/13 17:41:50 Handling 1st
2018/08/13 17:41:50 Got connection: HTTP/2.0
2018/08/13 17:41:50 Handling 2nd
```

### 全双工通讯
在Go的[HTTP/2 Demo](https://http2.golang.org/)页面上有个echo的例子，它演示了客户端和服务端的全双工的通讯流程。
我们先用cURL来测试一下：
```
$ curl -i -XPUT --http2 https://http2.golang.org/ECHO -d hello
HTTP/2 200 
content-type: text/plain; charset=utf-8
date: Tue, 24 Jul 2018 12:20:56 GMT

HELLO 
```
我们配置cURL 使用HTTP/2协议，发送一个PUT /ECHO 的请求，用“hello”作为请求内容。服务端返回一个 HTTP/2 200的响应，和“HELLO”的内容。在这里我们并没有做任何复杂的事情，除了在Header方面的差异之外它看起来和旧版本HTTP/1.1的半双工通讯是一样的。让我们在深入的研究进去，研究下如何使用HTTP/2进行全双工通讯的功能。


### 服务端实现
一个简单版本的实现HTTP echo的handler（这个版本不实现大写的响应内容的功能）如下。它使用http.ResponseWriter里新增的http.Flusher接口。
```
type flushWriter struct {
	w io.Writer
}

func (fw flushWriter) Write(p []byte) (n int, err error) {
	n, err = fw.w.Write(p)
	// Flush - 发送缓冲区里的数据给客户端
	if f, ok := fw.w.(http.Flusher); ok {
		f.Flush()
	}
	return
}

func echoCapitalHandler(w http.ResponseWriter, r *http.Request) {
	// 首先刷新响应头数据
	if f, ok := w.(http.Flusher); ok {
		f.Flush()
	}
	// 从请求主体里拷贝数据到 resonse writer上，并且执行刷新操作，发送内容给客户端
	io.Copy(flushWriter{w: w}, r.Body)
}
```

服务端拷贝从请求主体里发送来的全部内容到“flush writer"里，这就会写到ResponseWriter 并且调用 Flush()方法。 在这里我们又一次看到了类型断言的实现风格，这里的flush操作会把缓冲区里的数据发送给客户端。
注意一下，这里的实现是全双工的，服务端会在一个HTTP的handler调用里重复的进行这样的操作，读取到一行数据就刷新输出一行数据。



### Go客户端的实现
我尝试寻找Go客户端如何实现这个特性，发现了这个[Github issue](https://github.com/golang/go/issues/13444#issuecomment-161115822). Brad 给了一些建议如下代码所示，这里都是比较“低层”的代码实现，所以我增加了很多备注信息。
```

const url = "https://http2.golang.org/ECHO"

func main() {
    // 创建一个pipe（管道），这个对象实现了 io.Reader和io.Writer方法。写端会写到writer上，而读端会从reader读取。
	pr, pw := io.Pipe()
	
    // 创建一个 http.Request 对象，并把它的body设置为管道的读端。 当发生请求之后，就会写入到管道里去，会被作为请求主体发送。这样会使请求内容动态化，因此我们不需要在发送请求之前去定义它。
	req, err := http.NewRequest(http.MethodPut, url, ioutil.NopCloser(pr))
	if err != nil {
		log.Fatal(err)
	}
	
    // 发送请求
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		log.Fatal(err)
	}
	log.Printf("Got: %d", resp.StatusCode)
	
	// 执行循环，每隔1秒向管道写入一次，内容是当前的时间
	go func() {
		for {   
			time.Sleep(1 * time.Second)
			fmt.Fprintf(pw, "It is now %v\n", time.Now())
		}
	}()
	
    // 把服务端响应数据拷贝到标准输出 
	_, err = io.Copy(os.Stdout, resp.Body)
	log.Fatal(err)
}
```
这个例子非常有趣， 我们创建了一个“动态”的请求体，这个对于普通Go程序员来说不是一件好事。然而我们收到一个“动态”的响应内容，这个就比较正常了，但不太常见。在HTTP/1.1里这些请求和响应是用于发送或接受数据流的。但在这里响应的数据流在请求数据流结束之前就开始启动了。每一次我们通过管道发送数据给请求对象，响应数据从服务端的响应数据里返回。这个非常棒，我们用go程序实现了一个基于HTTP/2连接的全双工的通讯实例。

上边这个例子中的缺点是语言的API，标准库给我们提供了非常强大的工具，但是需要具备很多低层编程的知识才能把它们用好。

### 用 posener/h2conn 实现全双工通讯
h2conn 是一个非常小的库，它可以改进用户使用HTTP/2全双工的开发体验。

比如，上边实例的代码改用h2conn来实现，是这样的：

```
package main

import (
	"context"
	"fmt"
	"io"
	"log"
	"net/http"
	"os"
	"time"

	"github.com/posener/h2conn"
)

const url = "https://http2.golang.org/ECHO"

func main() {
	// 创建一个client ，使用HTTP的PUT方法
	c := h2conn.Client{Method: http.MethodPut}

	// 连接到 HTTP/2服务端， 返回的连接有2种情况：
	// 1.Write - 发送数据到服务端
	// 2.Read  - 接收服务端返回的数据
	conn, resp, err := c.Connect(context.Background(), url)
	if err != nil {
		log.Fatal(err)
	}
	defer conn.Close()
	log.Printf("Got: %d", resp.StatusCode)

	// 定期发送时间数据给服务端
	go func() {
		for {
			time.Sleep(1 * time.Second)
			fmt.Fprintf(conn, "It is now %v\n", time.Now())
		}
	}()

	// 读取服务端响应的数据到标准输出
	_, err = io.Copy(os.Stdout, conn)
	if err != nil {
		log.Fatal(err)
	}
}
```
可以看到服务端代码简化了很多。下面的代码示例实现了同上边echo 服务一样的功能：
```
func echo(w http.ResponseWriter, r *http.Request) {
	// 接收返回的连接，有2种情况
	// 1.Write - 发送数据到服务端
	// 2.Read  - 接收服务端返回的数据
	conn, err := h2conn.Accept(w, r)
	if err != nil {
		log.Printf(
			"Failed creating connection from %s: %s",
			r.RemoteAddr, err)
		http.Error(w,
			http.StatusText(http.StatusInternalServerError),
			http.StatusInternalServerError)
		return
	}
	defer conn.Close()

	// 把收到的任何数据都返回给客户端
	io.Copy(conn, conn)
}
```
如果需要看更多例子，可以来[这里](https://github.com/posener/h2conn/tree/master/example)

## 总结
Go 启用HTTP/2连接可以支持服务端推送和全双工通讯，同时它也向下兼容可以支持标准库实现的HTTP/1.1协议的TLS服务，这个是非常棒的。至于标准库的HTTP的client端不支持服务端推送，但它支持全双工通讯。本文中我介绍了一个使开发全双工通讯更优雅高效的工具库，可以在[posener/h2conn](https://github.com/posener/h2conn)找到它。

#### 原文自处
[https://posener.github.io/http2/](https://posener.github.io/http2/)



