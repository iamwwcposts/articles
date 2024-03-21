---
title: LVS流量转发模式
date: 2023-12-14
updated: 2023-12-14
issueid: 66
tags:
- net
---
LVS(linux virtual server) 做网关的几种转发模式

## DR(direct routing)
路由收包后先给到同vlan的LB，LB根据负载均衡策略选择后端X，将mac改为X的mac地址并发送

优势：
- X的response不需要过LB，X的response发给路由，路由送给client
- LB只是作为指路人，修改destination mac，并没有其他逻辑，所以性能很高
劣势
- LB和X必须处于同vlan
- 必须依赖ARP发现X的mac
- 修改MAC层，不对IP/TCP操作，所以不支持端口映射

## NAT
地址变换，LVS向后端转发时修改src-ip和dest-ip，确保X的response回包dest-ip是LVS
![](/assets/2023-12-14-1.png)
注意看第三个， S 是 200.200.200.2，对应第一张图的S，说明LB NAT向后端转发时，只修改了dest-ip。dest-ip指向X。

为了确保第四步response能够改回正确的src-ip，LB内部需要维护 (10.10.10.2, 200.200.200.2) 到 (200.200.200.1, 200.200.200.2)的映射，收到(10.10.10.2, 200.200.200.2)后查表改成(200.200.200.1, 200.200.200.2)

优势
- 支持端口映射
- 配置简单
劣势
- LVS和X必须同子网，并且LVS必须是网关，才能修改src-ip。不修改src-ip直接发给C，C四元组找不到对应的连接，TCP reset
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