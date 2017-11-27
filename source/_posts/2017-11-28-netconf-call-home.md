layout: post
title: SDN之NETCONF Call Home
published: true
comments: true
sharing: true
footer: true
categories: [SDN]
tags: [NETCONF, RFC, RFC8071]
date: 2017-11-28 00:18:35
keywords: SDN之NETCONF Call Home
---
本文主要内容都来自于[RFC8071](https://tools.ietf.org/html/rfc8071)。

## 介绍

`NETCONF Call Home`支持两种安全传输网络配置协议分别是`Secure Shell(SSH)`和传输层安全`(TLS)`。  
> `NETCONF`协议​​的绑定到`SSH`在[RFC6242](https://tools.ietf.org/html/rfc6242)中定义。
> `NETCONF`协议​​的绑定到`TLS`在[RFC7589](https://tools.ietf.org/html/rfc7589)中定义。
> `SSH`协议在[RFC4253](https://tools.ietf.org/html/rfc4253)中定义，`TLS`协议是在[RFC5246](https://tools.ietf.org/html/rfc4253)中定义。`SSH`和`TLS`协议都是`TCP`协议之上的协议。

### 动机

`call home`对于网络设备的初始化部署和持续管理都是非常有帮助的。那网络设备为什么使用`call home`这种方式？

 - 网络设备在第一次启动后可以主动`call home`，以便在其管理系统上注册。

- 网络设备可以以一种动态分配`IP`地址的方式访问网络，但是不会将其分配的IP地址注册到映射服务(例如，动态`DNS`)。

- 网络设备可以部署在实现所有内部网络IP地址的网络地址转换（NAT）的防火墙后面。

- 网络设备元件可以部署在不允许任何管理访问内部网络的防火墙之后。

- 网络设备可以配置为“隐身模式”，因此没有任何的端口可以提供给管理系统打开连接。

- 运营商可能倾向于让网络设备发起管理连接，认为在数据中心中保护一个开放端口比在网络中的每个网络设备上具有开放端口更容易。
<!-- more -->
### 解决方案概述

下图说明了协议分层的`call home`

![NETCONF Call Home Sequence](https://raw.githubusercontent.com/tonydeng/sdn-handbook/master/sdn/images/netconf-call-home-sequence.png)

> 消息层流程PlantUML请查看[netconf-messages-layer-flow.puml](https://raw.githubusercontent.com/tonydeng/sdn-handbook/master/puml/netconf-call-home.puml)

这张图有以下几点：

 1. `NETCONF`服务器首先启动一个`TCP`连接到`NETCONF`客户端。
 2. 使用这个`TCP`连接，`NETCONF`客户端启动到`NETCONF`服务器的`SSH/TLS`会话。
 3. 使用此`SSH/TLS`会话，`NETCONF`客户端启动一个到`NETCONF`服务器的`NETCONF`会话。

## NETCONF客户端

术语“客户端”在[RFC6241第1.1节](https://tools.ietf.org/html/rfc6241#section-1.1)中定义。 在网络管理的情况下，`NETCONF`客户端可能是一个网络管理系统。

### 客户端协议操作事项

- C1 `NETCONF`客户端侦听来自`NETCONF`服务器的`TCP`连接请求。 客户端必须支持在[第6节](https://tools.ietf.org/html/rfc8071#section-6)中定义的`IANA`分配的端口上接受`TCP`连接，但可以配置为侦听不同的端口。
- C2 `NETCONF`客户端接受传入的`TCP`连接请求，并建立`TCP`连接。
- C3 使用此`TCP`连接，`NETCONF`客户端启动`SSH`客户端[RFC4253](https://tools.ietf.org/html/rfc4253)或`TLS`客户端[RFC5246](https://tools.ietf.org/html/rfc5246)协议。 例如，假定使用`IANA分`配的端口，则在端口`4334`接受连接时启动`SSH`客户端协议，并且在端口`4335`或端口`4336`上接受连接时启动`TLS`客户端协议。
- C4 当使用`TLS`时，`NETCONF`客户端必须告知`"peer_allowed_to_send"`，如[RFC6520](https://tools.ietf.org/html/rfc6520)所定义。 这是必需的，以便`NETCONF`服务器知道在`call home`连接时需要发送心跳包，保持长连接。
- C5 作为建立`SSH`或`TLS`连接的一部分，`NETCONF`客户端必须验证服务器提供的主机密钥或证书。 该验证可以通过证书路径验证或通过将主机密钥或证书与先前信任的或“固定的”值进行比较来完成。 如果证书被提交并且包含撤销检查信息，`NETCONF`客户端应该检查证书的撤销状态。 如果确定证书已被吊销，客户端必须马上关闭连接。
- C6 如果使用证书路径验证，则`NETCONF`客户端必须确保提供的证书具有对预先配置的颁发者证书的有效信任链，并且所呈现的证书对客户端之前知道的“标识符”[RFC6125](https://tools.ietf.org/html/rfc6125)进行编码 连接尝试。 如何在证书中编码标识符可以由与证书颁发者相关的策略来确定。 例如，可以知道给定的颁发者只在`X.509`证书的`“CommonName”`字段中签署具有唯一标识符（例如，序列号）的`IDevID`证书[Std-802.1AR-2009](https://tools.ietf.org/html/rfc8071#ref-Std-802.1AR-2009)。
- C7 服务器的主机密钥或证书经过验证后，客户端将以`SSH`或`TLS`协议进行建立`SSH`或`TLS`连接。 在使用`NETCONF` 服务器执行客户端认证时，`NETCONF`客户端必须仅使用先前为`NETCONF`服务器提供的主机密钥或服务器证书关联的凭证。
- C8 一旦`SSH`或`TLS`连接建立，`NETCONF`客户端启动`NETCONF`客户端[RFC6241](https://tools.ietf.org/html/rfc6241)或`RESTCONF`客户端[RFC8040](https://tools.ietf.org/html/rfc8040)协议。 假设使用`IANA`分配的端口，当在端口`4334`或端口`4335`上接受连接时启动`NETCONF`客户端协议，并且当在端口`4336`上接受连接时启动`RESTCONF`客户端协议。


### 客户端配置数据模型

如何配置`NETCONF`或`RESTCONF`客户端超出了本文的范围。

例如，可以使用什么样的配置来启用对`call home`的监听，配置可信证书颁发者，或者为预期的连接配置标识符。 也就是说，在[NETCONF-MODELS](https://tools.ietf.org/html/rfc8071#ref-NETCONF-MODELS)和[RESTCONF-MODELS](https://tools.ietf.org/html/rfc8071#ref-RESTCONF-MODELS)中提供了用于配置`NETCONF`和`RESTCONF`客户端的`YANG` [RFC7950](https://tools.ietf.org/html/rfc7950)数据模块，包括`call home`。

## NETCONF服务器

术语“服务器”在[RFC6241第1.1节](https://tools.ietf.org/html/rfc6241#section-1.1)中定义。 在网络管理的情况下，`NETCONF`服务器可能是网络元件或设备。