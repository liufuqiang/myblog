---
title: "利用hugo+caddy快速搭建一个简单的博客系统"
date: 2018-08-16T17:50:07+08:00
---

## 技术选型介绍
[hugo](https://gohugo.io/) 是一个非常简单的基于markdown生成博客的系统，是go语言开发的。
[caddy](https://caddyserver.com/) 是一个简单易用的webserver服务，很好的支持了H2，尤其是基于[Let's Encrypt](https://letsencrypt.org/)的 自动签发证书的功能。

## 安装 caddy
安装方式有多种，可以到[官网](https://caddyserver.com/)上看，这里我们介绍3种安装方式：一个是用官方发布的二进制包安装，一个用Go的方式安装的步骤,一个是在官网的自助下载页面定制插件下载。
### 二进制包安装：
```
wget https://github.com/mholt/caddy/releases/download/v0.11.0/caddy_v0.11.0_linux_amd64.tar.gz
tar xvfz caddy_v0.11.0_linux_amd64.tar.gz
cp caddy /usr/bin/
```
### go源码安装

```
go get github.com/mholt/caddy/caddy 
go get github.com/caddyserver/builds
cd $GOPATH/src/github.com/mholt/caddy/caddy
go run build.go
```
> 这里有个要注意的是，caddy为了支持多平台的build配置，把普通的go build 升级为单独用build.go的方式进行包装。
安装好后，确保caddy在Path可找到的地方，这里因为我们是自己build的源码，需要自己把caddy二进制文件放到PATH中的某个目录下。

### 官方定制插件下载
在官方的下载页面可以根据自己的操作系统、需要的插件种类、License类型等定制下载包。 这里需要注意的是license我们选择person，因为不想付费哈。 插件选择太多或选择了某些插件会导致服务端500错误，这个比较坑，希望官方早点解决这个bug。 根据本文的主题需要，我们这里选择http.hugo 和http.git两个插件。
> 小技巧： 如果你清楚需要什么插件，可以直接用命令行方式进行下载安装即可： ```  curl https://getcaddy.com | bash -s personal http.git,http.hugo```

```
[liufuqiang@cloud liufuqiang]$ curl https://getcaddy.com | bash -s personal http.git,http.hugo
Downloading Caddy for linux/amd64 (personal license)...
Download verification OK
Extracting...
Backing up /usr/bin/caddy to /usr/bin/caddy_old
(Password may be required.)
[sudo] password for liufuqiang:
Putting caddy in /usr/local/bin (may require password)
Caddy 0.11.0 (non-commercial use only)
Successfully installed
```


我们可以执行下 caddy -h 测试安装是否成功。
```
[liufuqiang:~]$caddy -h
Usage of caddy:
  -agree
    	Agree to the CA's Subscriber Agreement
  -ca string
    	URL to certificate authority's ACME server directory (default "https://acme-v02.api.letsencrypt.org/directory")
  -catimeout duration
    	Default ACME CA HTTP timeout
  -conf string
    	Caddyfile to load (default "Caddyfile")
  -cpu string
    	CPU cap (default "100%")
  -disable-http-challenge
    	Disable the ACME HTTP challenge
  -disable-tls-sni-challenge
    	Disable the ACME TLS-SNI challenge
  -disabled-metrics string
    	Comma-separated list of telemetry metrics to disable
  -email string
    	Default ACME CA account email address
  -env string
    	Path to file with environment variables to load in KEY=VALUE format
  -grace duration
    	Maximum duration of graceful shutdown (default 5s)
  -host string
    	Default host
  -http-port string
    	Default port to use for HTTP (default "80")
  -http2
    	Use HTTP/2 (default true)
  -https-port string
    	Default port to use for HTTPS (default "443")
  -log string
    	Process log file
  -pidfile string
    	Path to write pid file
  -plugins
    	List installed plugins
  -port string
    	Default port (default "2015")
  -quic
    	Use experimental QUIC
  -quiet
    	Quiet mode (no initialization output)
  -revoke string
    	Hostname for which to revoke the certificate
  -root string
    	Root path of default site (default ".")
  -type string
    	Type of server to run (default "http")
  -validate
    	Parse the Caddyfile but do not start the server
  -version
    	Show version
```

安装好后就可以非常便捷的使用caddy了，先演示一个自动签署CA的过程吧，
> 因为普通账号启动程序是不能启动1024以下的端口的，为了让caddy可以直接启动80/443端口，我们用setcap对这个程序进行设置： ``` sudo setcap 'cap_net_bind_service=+ep' /usr/local/bin/caddy```

```
[liufuqiang@cloud liufuqiang]$ caddy -host blog.thor.today
[sudo] password for liufuqiang:
Activating privacy features...

Your sites will be served over HTTPS automatically using Let's Encrypt.
By continuing, you agree to the Let's Encrypt Subscriber Agreement at:
  https://letsencrypt.org/documents/LE-SA-v1.2-November-15-2017.pdf
Please enter your email address to signify agreement and to be notified
in case of issues. You can leave it blank, but we don't recommend it.
  Email address: liufuqiang0123@gmail.com
done.
http://blog.thor.today
https://blog.thor.today
```
整个过程毫无压力，唯一需要你做的就是提供一个邮箱，而且邮箱也不是必须要填的。之后，一个web服务就跑起来了，打开网站既可以看到你的网站是https的，同时也是h2协议了。
> 注意你需要先具备一个域名，并且解析到这个机器上，因为Let's Encrypt签署证书的过程中会通过域名和443/80端口来和你的网站做通讯。

