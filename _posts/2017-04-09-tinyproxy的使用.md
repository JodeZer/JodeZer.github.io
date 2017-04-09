---
layout: post
title: tinyproxy的使用
date: 2017-04-09 11:06:26 +0800
comments: false
---

前言
=======

&#160; &#160; &#160; &#160;tinyproxy是一个开源的正向代理软件，仓库位于github，[tinyproxy - a light-weight HTTP/HTTPS proxy daemon for POSIX operating systems](https://github.com/tinyproxy/tinyproxy)。

&#160; &#160; &#160; &#160;提及代理技术，必谈Nginx。Nginx可能是地球上最著名的反向代理，也是最早解决C10K问题的服务器软件。但Nginx并不能满足作为一个正向代理的技术需求，因为Nginx不支持https的正向代理。

&#160; &#160; &#160; &#160;不支持https的技术原因是Nginx没有实现http1.1 Connect方法。关于Connect和隧道技术，详见[RFC2817](https://www.ietf.org/rfc/rfc2817.txt)。另外知乎上也有一个讲得很不错的答案。[什么是HTTP隧道，怎么理解HTTP隧道呢？](https://www.zhihu.com/question/21955083/answer/142736329)。隧道的含义大约就是帮助无法完成TLS握手的代理服务器透传可以完成TLS握手的客户端请求，而不再解析流量中的内容。

&#160; &#160; &#160; &#160;没有实现Connect的结果就是Nginx无法将加密的https报文正确地代理到要去的地方，而Nginx作者的意思是，Nginx没有必要再去实现https的正向代理了，市面上已经有很多实现了。因此经过一些调查，选择了这款tinyproxy作为正向代理的研究对象。

使用
===
##### 编译依赖(ubuntu14)
```
$ sudo apt-get install asciidoc autotools-dev automake
```
##### 下载编译
```
$ git clone https://github.com/tinyproxy/tinyproxy
$ cd tinyproxy
$ ./autogen.sh
$ ./configure
$ make
$ make install
```

##### apt安装
或者可以选择直接安装

```
$ sudo apt-get install tinyproxy
```
##### 配置文件简介
默认配置文件为```/etc/tinyproxy.conf```

重要配置如下：

```
代理端口
Port 8888

连接最大空闲时间
Timeout 600

日志文件位置和日志级别
Logfile "/var/log/tinyproxy/tinyproxy.log"
LogLevel Info

最大客户端数量
MaxClients 100

最大最小服务进程
MinSpareServers 5
MaxSpareServers 20

其实进程数
StartServers 10

子进程最大连接数
MaxRequestsPerChild 0

网段限制，客户端必须位于网段内，否则请求被拒绝
Allow 127.0.0.1
Allow 10.0.0.0/8

http header Via的值
ViaProxyName "tinyproxy"

http Connect 端口
ConnectPort 443
ConnectPort 563
```

##### golang example

```go
func main(){
	proxy := func(_ *http.Request) (*url.URL, error) {
		return url.Parse("http://127.0.0.0:8888")
	}
	transport := &http.Transport{Proxy: proxy}
	proxy_client = &http.Client{
		Transport:transport,
	}
	proxy_client.Get(...)
}

```
