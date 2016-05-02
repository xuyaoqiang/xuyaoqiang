---
layout: post
title: 使用Gor实时复制线上流量，搭建服务端预发布系统
---
> 项目需要频繁上线，如何才能降低服务端上线的风险，又降低测试的成本呢？

## 预发布系统的作用
  作为一个手机应用，需要快速迭代和敏捷开发，因此服务端的接口面临频繁上线的需求 一般每周都需要发布正式的版本和功能，最多的情况下一天上线了三次。然而上线前一般都需要经历单元测试、功能回归测试、abtest甚至压力测试，测试人手不足， 如何才能在频繁上线的同时，降低服务端上线的风险和降低测试的成本呢？  
  
 app产品服务端的测试工具很多，包括apache bench等，然而这些工具都没有办法解决一个问题：
 
 	如何模拟生成环境流量
 	
 根据自己的经验，__80%的服务端故障发生在上线后的两个小时之内__。 如果能够搭建一个和生成环境一样的预发布环境，完全复制生成环境的流量，在正式发布前在预发布环境先发布一遍，就能够让准备发布的版本在不影响用户的情况下接受真实流量的洗礼。
 
 预发布系统有以下显而易见的优点：
 
  1. __回归测试__：及早发现功能漏洞，测试数据符合用户真实的请求分布，而且无需准备繁多的测试用例，极大降低了测试成本。
  2. __对比测试__：预发布系统和生成环境就是一对完美的abtest，你可以轻易的根据关心的指标找出新版本的代码在哪些方面的有了提升，哪些方面还有不足。
  3. __压力测试__：对于分布式的服务架构，可以将多台服务器的流量复制到一台或几台预发布服务器上，可以直接的知道被测试服务的极限是正常流量的几倍，而且不是对单一接口的测试，而是对服务整体的测试。
 
## 使用Gor复制生成环境HTTP流量

[__Gor__](https://github.com/buger/gor) 是一个用Go语言开发的一个http流量复制工具。复制流量过程中有listener和replayer两个角色(实际上是一个进程)。


![](https://camo.githubusercontent.com/556d4aa5db32de9535d84d6c6c07f6564b43fc0b/687474703a2f2f692e696d6775722e636f6d2f396d716a32534b2e706e67)

  
### 流量复制的流程是:
	
	1. listener监听生产环境端口
	2. listener复制http请求，修改目的地址，转发到replayer监听的端口
	3. replayer转发请求到测试服务器的端口
	4. 测试服务器处理复制的请求
	5. replayer丢弃测试服务器返回的响应。
	
### Gor的安装配置和使用
Gor的下载安装特别简单，可以在 https://github.com/buger/gor/releases进行下载。 安装则有两种形式：

	1. 直接下载二进制可执行文件，解压即可
	2. 通过源码编译安装：
		1. 搭建标准的Go语言环境 (http://golang.org/doc/code.html),设置$GOPATH 环境变量
		2. go get github.com/buger/gor
		3. cd $GOPATH/src/github.com/buger/gor
		4. go build生成二进制可执行文件

#### 启动Gor
在生成环境流量入口启动Gor listener:

	sudo gor --input-raw :80 --output-tcp replay.local:28020
	
在测试服务器上启动Gor replayer:

	gor --input-tcp replay.local:28020 --output-http http://staging.com

__gor 使用listener捕捉请求的时候需要sudo权限__

也可以将listener和replayer合二为一：

	gor --input-tcp replay.local:28020 --output-http http://staging.com
	
还可以将流量复制两份到不同的测试服务：
	
	gor --input-tcp :28020 --output-http "http://staging.com"  --output-http "http://dev.com"
	
也可以将流量像负载均衡一样分配到不同的服务器：

	gor --input-tcp :28020 --output-http "http://staging.com"  --output-http "http://dev.com" --split-output true
	
另外gor还支持对listener和replayer的rate-limit，流量过滤，以及流量放大：

	gor --input-tcp :28020 --output-http "http://staging.com|10"
	gor --input-tcp :28020 --output-http "http://staging.com|10"

更多的配置可以查看 [文档](https://github.com/buger/gor)

	
## 预发布系统的搭建
预发布系统主要由`测试服务器`和 `Gor流量复制工具`组成，为了查看和分析测试效果，还可以接入各种监控分析系统。


为了将回归测试和压力测试合二为一，我们将线上的五台api服务器的流量复制到了一台preview(预发布)服务器：

	1. 部署搭建测试服务，测试服务使用的数据库，消息队列，缓存等等需要生成环境隔离，避免对生产环境造成影响；同时，测试环境使用的数据尽量保证和生成环境一致，以保证测试效果。
	2. 在api服务器和preview服务器上安装gor，为了方便，我们直接使用了官方提供的二进制文件
	3. 在api服务器上启动gor listener，复制80端口的流量，转发到预发布服务器的28020端口：
		sudo gor --input-raw :80 --output-tcp preview:28020
	4. 	在preview服务器上启动gor replayer， 监听28020端口，同时将请求转发到测试服务器上5000端口
	5. 启动测试服务器上的测试进程，监听5000端口

就这么简单，预发布系统就搭建成功了。
	
	
	
## 非http协议的流量复制
对于不是基于http协议的流量复制，可以使用我司 王斌 维护的流量复制工具 [tcpcopy](https://github.com/wangbin579/tcpcopy), tcpcopy 支持tcp协议流量的复制、转发、拦截，非常棒的工具，感兴趣的可以研究。因为其安装配置稍显复杂，这里不在赘述。

## 总结

	1. 极少的情况下，Gor会有少量的丢包问题出现，但是不影响测试效果。
	2. 建议将Gor的listener和replayer分开，减少对生成环境性能的影响。
	3. 部署搭建测试服务，测试服务使用的数据库，消息队列，缓存等等需要生成环境隔离，避免对生产环境造成影响；同时，测试环境使用的数据尽量保证和生成环境一致，以保证测试效果。
	4. 由于测试环境和生成环境的数据无法做到实时的同步，所以要注意区分哪些错误是有程序的bug，哪些是由数据不一致导致的。
	5. 预发布系统能够一站解决回归测试、abtest和压力测试，然而还是会有局限性：对于比较复杂的业务逻辑，并不能直观的表现出来，需要后台对数据进行进一步的分析比较后才能判断。


## 附录

1. gor: https://github.com/buger/gor
2. tcpcopy: https://github.com/wangbin579/tcpcopy




  
