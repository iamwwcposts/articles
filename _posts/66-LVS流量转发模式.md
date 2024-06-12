---
date: 2023-12-14
updated: 2024-05-28
issueid: 66
tags:
- net
title: LVS流量转发模式
---
LVS(linux virtual server) 做网关的几种转发模式

## DR(direct routing)
路由收包后先给到同vlan的LB，LB根据负载均衡策略选择后端X，将dest mac改为X的mac地址，src-mac为网关mac。用于X的回程
核心一点：RS和LVS都使用同一个入口VIP，初次收包LVS改MAC发给RS，RS的dest-ip也是LVS的dest-ip。

> [linux/net/netfilter/ipvs/ip\_vs\_xmit.c at 7e46731ed902354a86caf3510011eb2b827a8f73 · taikulawo/linux · GitHub](https://github.com/taikulawo/linux/blob/7e46731ed902354a86caf3510011eb2b827a8f73/net/netfilter/ipvs/ip_vs_xmit.c#L1421)
> dpvs 
> [neigh_output](https://github.com/willdeeper/dpvs/blob/4b384e1880cbd5df78f118cbd51e5df9890710dd/src/neigh.c#L691)
> [neigh_fill_mac](https://github.com/willdeeper/dpvs/blob/4b384e1880cbd5df78f118cbd51e5df9890710dd/src/neigh.c#L432)

优势：
- X的 response 不需要过LB，X的response发给路由，路由送给client
- LB作为指路人只修改destination mac，并没有其他逻辑，所以性能很高
劣势
- LB和X必须处于同vlan
- 必须依赖ARP发现X的mac
- 修改MAC层，不对IP/TCP操作，所以不支持端口映射

 X 回程到客户端时从其他网口收到了src-mac是入口网卡地址的包，交换机有三层路由功能，根据路由规则发现是外网ip，从入口网口发到公网了

### hub vs switch vs router
一层hub，二层交换，三层路由
- hub 只有广播功能，从一个网口收到的数据会转发到其他网口
- switch 二层交换能记忆MAC，能直接在两个网口转发数据，不广播其他网口
- router 能处理三层ip包，内置路由规则。根据ip地址转发到网口

## NAT
地址变换，LVS向后端转发时修改dest-ip，确保X的response回包dest-ip是LVS
![](/assets/2024-05-27-43.png)
注意看第三步， S 是 200.200.200.2，对应第一步的S，说明LB NAT向后端转发时，只修改了dest-ip。dest-ip指向X。

为了确保第四步response能够改回正确的src-ip，LB内部需要维护 (10.10.10.2, 200.200.200.2) 到 (200.200.200.1, 200.200.200.2)的映射，收到(10.10.10.2, 200.200.200.2)后查表改成(200.200.200.1, 200.200.200.2)

> 进行destination network translation 变换。调用tcp的dnat handler（拿tcp举例，udp也有handler）
> [linux/net/netfilter/ipvs/ip\_vs\_xmit.c at 7e46731ed902354a86caf3510011eb2b827a8f73 · taikulawo/linux · GitHub](https://github.com/taikulawo/linux/blob/7e46731ed902354a86caf3510011eb2b827a8f73/net/netfilter/ipvs/ip_vs_xmit.c#L901)
> 在handler里修改dport
> <https://github.com/taikulawo/linux/blob/7e46731ed902354a86caf3510011eb2b827a8f73/net/netfilter/ipvs/ip_vs_proto_tcp.c#L264> 
> 接下来修改dip
> [linux/net/netfilter/ipvs/ip\_vs\_xmit.c at 7e46731ed902354a86caf3510011eb2b827a8f73 · taikulawo/linux · GitHub](https://github.com/taikulawo/linux/blob/7e46731ed902354a86caf3510011eb2b827a8f73/net/netfilter/ipvs/ip_vs_xmit.c#L903)

优势
- 支持端口映射
- 配置简单
劣势
- LVS和X必须同子网，并且LVS必须是网关，才能修改destip。不修改destip直接发给C，C四元组找不到对应的连接而TCP reset

## FULL-NAT
和NAT大体相同，如果说上面的NAT是DNAT（destination-NAT），那么FULL-NAT更像是 SD-NAT(source-destination-NAT。同时修改src-ip和dest-ip，dest-ip依旧是X的rip(real ip)，src-ip是 LB。X和LB就算不同vlan，依旧可以经路由转发给LB，LB最后发给cip

优势
- LB 和 X 可以不同vlan
劣势
- FULL-NAT同时修改src和dest ip，会导致X拿不到真正的 cip，一些依赖cip的后端服务会出问题
解决办法
cip通过tcp option字段传递，后端服务器安装TOA模块，从option读出cip再给上层应用

## reference
> [Load Balancing - Linux Virtual Server (LVS) and Its Forwarding Modes](https://www.alibabacloud.com/blog/load-balancing---linux-virtual-server-lvs-and-its-forwarding-modes_595724)