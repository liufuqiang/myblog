---
title: "利用hugo+caddy快速搭建一个简单的博客系统"
date: 2018-08-16T17:50:07+08:00
tags: ["博客","hugo","caddy"]
---
## 技术选型介绍
[hugo](https://gohugo.io/) 是一个非常简单的基于markdown生成博客的系统，是go语言开发的博客系统。
[caddy](https://caddyserver.com/) 是一个简单易用的webserver服务，很好的支持了H2，尤其是基于[Let's Encrypt](https://letsencrypt.org/)的 自动签发证书的功能，让你用的毫无压力。

## 安装 caddy
安装方式有多种，可以到[官网](https://caddyserver.com/)上看，这里我们介绍3种安装方式：一个是用官方发布的二进制包安装，一个用Go的方式安装的步骤,一个是在官网的自助下载页面定制插件下载。
### 二进制包安装：
```bash
wget https://github.com/mholt/caddy/releases/download/v0.11.0/caddy_v0.11.0_linux_amd64.tar.gz
tar xvfz caddy_v0.11.0_linux_amd64.tar.gz
cp caddy /usr/bin/
```
### go源码安装

```bash
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

```golang
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
```bash
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

```bash
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

到此，caddy的安装就算完成了。下面我们介绍下hugo的安装。

## hugo安装
安装方式有很多种，可以到[官方安装指导](https://gohugo.io/getting-started/installing/)了解更多安装方式，我们选择一个Linux系统的安装方式(从 官方项目的github release下载对应二进制包）：
```bash
wget https://github.com/gohugoio/hugo/releases/download/v0.46/hugo_0.46_Linux-64bit.tar.gz
tar xvfz hugo_0.46_Linux-64bit.tar.gz
cp hugo /usr/local/bin/
```
执行 hugo version 可以测试下是否安装成功
```bash
Hugo Static Site Generator v0.46 linux/amd64 BuildDate: 2018-08-01T09:00:55Z
```

## caddy与hugo结合
利用caddy + hugo、git插件 可以实现自动更新博客的功能。
用我博客站的配置来介绍下吧。先看配置文件：
```bash
https://blog.thor.today {
    tls user@xx.com
    gzip
    root /da1/liufuqiang/web/public
    git github.com/liufuqiang/myblog.git {
        path /da1/liufuqiang/myblog
        then hugo --destination=/da1/liufuqiang/web/public
        hook /webhook **********
        hook_type github
        clone_args --recursive
        pull_args --recurse-submodules
    }
    hugo
}
```

重点关注下 git 模块的配置：
- path: git代码拉取下来的目标位置
- then hugo --destination: hugo解析md静态化后的文件的目标位置，注意这里要和我们的web root保持一致。
- hook ：这里是为了实现能够监听github上的代码变更事件定义的hook api。这块需要到github对应的项目里，在Settings->Webhooks里去配置一个webhook，本处星号是配置的密码，两边要保存一致。
> 这里有个git版本的问题需要注意，因为就centos6/7而言，默认安装（yum里带的）git的版本都比较低，如我的是centos6，git是1.7的，不支持recurse-submodules的拉取代码参数，会报错。这里可以需要对git的版本做下更新，文章后边会有介绍。

配置好Caddayfile后，就可以直接启动服务，一个依赖githum+markdown机制切可以自动更新的博客站就完美搞定了。



### 快捷更新git版本的方法
传统的源码安装非常耗时，我们还是选择用yum方式来安装，如下：
```bash
yum remove git
yum install epel-release
yum install https://centos6.iuscommunity.org/ius-release.rpm 
yum install git2u
```
