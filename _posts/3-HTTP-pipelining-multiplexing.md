---
title: HTTP-pipelining-multiplexing
date: 2019-09-11
updated: 2020-10-06
issueid: 3
tags:
---
![](https://webcdn.chaochaogege.net/images/20190321152954.png)

#### HTTP/1.1

可以将请求一股脑发送出去，然后 `client` 等待服务器回应，如图二，但第一个请求如果被阻塞，那么后面的请求都没办法处理
缺点：
    1. 对服务器负担很大
    2. http request 级别的 队首阻塞

#### HTTP/2

h2 通过对请求进行分帧，一个 HTTP 请求，回应对应着 `Stream`

逻辑上
![image](https://user-images.githubusercontent.com/24750337/95217161-75513300-0825-11eb-83d2-6c834202697b.png)

实际上
![image](https://user-images.githubusercontent.com/24750337/95217278-974ab580-0825-11eb-952d-34f4ebce39a8.png)


通过将一个请求打散，h2 解决了 `http request` 级别的阻塞，但使用同一个 `TCP` ，全部的请求遵守同一个流量控制，只要前方有一个帧遭到了阻塞（丢失），后面的请求始终不能处理（ `TCP` 不会上上层交付）

#### Quic (Quick UDP Internet Connections)

Google开发的使用 `UDP` 模拟 `TCP`，集成了流量控制与拥塞控制，由于UDP面向数据报，所以从根本上解决了单个 `TCP` 造成的拥塞问题

TCP 使用 四元组(src port,src ip, dst port, dst ip)来唯一标识连接
一个 TCP 连接建立需要 1.5RTT，
当从wifi切换到移动网络，由于 IP 更改，导致原来的连接丢失，所以会出现短暂的掉线现象

而Quic 使用一个由 client 生成的8字节的id来标识 连接，即使网络环境改变，也能使用id来进行RTT为0的重连
https://docs.google.com/document/d/1gY9-YNDNAB1eip-RTPbqphgySwSNSDHLq9D5Bty4FSU/edit

#### HTTP/3

IETF(互联网工程任务组) 觉着 Quic 是个好东西，希望能用来承载别的应用层协议

最终分层

UDP -> Quic -> Application Layer

使用 `Quic` 来承载 `HTTP` 流量


参考
> https://liudanking.com/arch/what-is-head-of-line-blocking-http2-quic/
