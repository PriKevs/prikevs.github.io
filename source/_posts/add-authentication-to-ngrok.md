---
title: 给ngrok添加身份验证
date: 2016-12-26
tags:
- ngrok
- golang
---

最近有个同学需要在校外访问学校内网，然而学校的SSL VPN出了问题，只能用自己的方式来解决了……
<!-- more -->

## 需求 ##

最近有个同学需要在校外访问学校内网，然而学校的SSL VPN出了问题，只能用自己的方式来解决了。
听到这个需求，我的第一反应是用N2N VPN，使用UDP Hole Punching的方式来穿透学校内网组建VPN。
于是立马写了一套简单的工具来检测学校的网络是否支持UPD Hole Punching，不过这不是今天的主题，以后有机会再说。

由于N2N年久失修，对Mac用户也不友好，我最终使用来一个更简单的方式——在内网本地搭一个shadowsoks的server，然后用ngrok将端口暴露到外网服务器，直接用ss的客户端就可以将流量转接到本地了。

首先我简单介绍一下ngrok:

> [ngrok](https://github.com/inconshreveable/ngrok) 
> is a reverse proxy that creates a secure tunnel from a public endpoint to a locally running web service.
> ngrok captures and analyzes all traffic over the tunnel for later inspection and replay.

简单来说，ngrok最主要的功能就是将本地的端口映射到外网的某个端口，以方便地完成很多需要外网访问却不希望deploy在服务器上的功能，比如现场demo、临时分享等。

目前ngrok有两种版本，一个是开源但是几近停止维护的ngrok 1.x，还有一个是闭源商用的ngrok 2.x。
这里我们选用1.x，方便self hosting部署以及自定义功能比如本文的题目：添加身份验证。

我们可以根据[Self Hosting Doc](https://github.com/inconshreveable/ngrok/blob/master/docs/SELFHOSTING.md)的文档完成最初的部署。
然而部署完我们会发现，原始版本的ngrok的server是没有身份验证功能的，也就是说任何人都可以通过ngrok 1.x的client使用我们的服务器，毕竟是私人的服务器，所以我希望给ngrok加上身份验证的功能。

## 分析 ##

下面我们开始分析相关的ngrok的源代码。

`src/ngrok/server/control.go#func NewControl`
```
func NewControl(ctlConn conn.Conn, authMsg *msg.Auth) {

	...

	failAuth := func(e error) {
		_ = msg.WriteMsg(ctlConn, &msg.AuthResp{Error: e.Error()})
		ctlConn.Close()
	}

	// register the clientid
	c.id = authMsg.ClientId
	if c.id == "" {
		// it's a new session, assign an ID
		if c.id, err = util.SecureRandId(16); err != nil {
			failAuth(err)
			return
		}
	}

	...

}
```

我们发现这一段代码实现的正是某种访问控制的功能，这个`authMsg`里说不定包含了一些我们需要的验证信息，下面我们查看`authMsg`的定义。

`src/ngrok/msg/msg.go#Auth struct`
```
// When a client opens a new control channel to the server
// it must start by sending an Auth message.
type Auth struct {
	Version   string // protocol version
	MmVersion string // major/minor software version (informational only)
	User      string
	Password  string
	OS        string
	Arch      string
	ClientId  string // empty for new sessions
}

```

这里我们看到了`User`和`Password`，下意识觉得这里有戏，我们需要查证这两个值的来源。
我们可以在client中寻找对应的参数设定位置。

<过程略……>

最终我们会发现，`Auth struct`中的User对应的正是client命令行参数中的`-authtoken`
因此，我们可以在-authtoken中包含我们的验证信息。

## 实现 ##

我们采用`-authtoken='username:password'`的方式来进行验证。具体实现的话只需要在`src/ngrok/server/control.go#func NewControl`中添加对`Auth struct`中的`User`验证的方法就行。

我的实现方案是，可以手动创建并用`-secretPath`指定（默认在`/etc/ngrok-secrets`）存储用户名和密码的文件的位置，格式为：

```
# username      password
  example-user  example-password
```

具体实现可以见我fork之后的[commit: add support to authenticate by token](https://github.com/prikevs/ngrok/commit/aa9b88d4e070069db6d8f88aa82526bcbcd1d0b6)

以后使用时只需要加上-authtoken参数即可，如:
```
$ ngrok -authtoken="username:password" -proto=tcp 8888
```

## 总结 ##

我只是添加了很简单的一种身份验证方式，大家其实可以根据自己的需求与其他验证系统结合起来=)

