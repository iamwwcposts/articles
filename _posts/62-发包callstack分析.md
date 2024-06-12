---
tags:
- Kernel
title: 发包callstack分析
date: 2023-11-03
updated: 2023-11-03
issueid: 62
---
## 内核发包路径分析

触发方式

```bash
curl http://baidu.com
```

### udp dns callstack

![发包](/assets/2023-10-19-15-55-22.png)

### tcp callstack

![](/assets/2023-10-24-140558769.png)

```
__dev_xmit_skb (\root\codes\kernel-dev\linux\net\core\dev.c:3779)
__dev_queue_xmit (\root\codes\kernel-dev\linux\net\core\dev.c:4310)
dev_queue_xmit (\root\codes\kernel-dev\linux\include\linux\netdevice.h:3082)
neigh_hh_output (\root\codes\kernel-dev\linux\include\net\neighbour.h:529)
neigh_output (\root\codes\kernel-dev\linux\include\net\neighbour.h:543)
ip_finish_output2 (\root\codes\kernel-dev\linux\net\ipv4\ip_output.c:238)
__ip_finish_output (\root\codes\kernel-dev\linux\net\ipv4\ip_output.c:0)
NF_HOOK_COND (\root\codes\kernel-dev\linux\include\linux\netfilter.h:293)
ip_output (\root\codes\kernel-dev\linux\net\ipv4\ip_output.c:439)
dst_output (\root\codes\kernel-dev\linux\include\net\dst.h:458)
ip_local_out (\root\codes\kernel-dev\linux\net\ipv4\ip_output.c:128)
__ip_queue_xmit (\root\codes\kernel-dev\linux\net\ipv4\ip_output.c:543)
ip_queue_xmit (\root\codes\kernel-dev\linux\net\ipv4\ip_output.c:557)
__tcp_transmit_skb (\root\codes\kernel-dev\linux\net\ipv4\tcp_output.c:1415)
tcp_ack_snd_check (\root\codes\kernel-dev\linux\net\ipv4\tcp_input.c:5616)
tcp_rcv_state_process (\root\codes\kernel-dev\linux\net\ipv4\tcp_input.c:6738)
tcp_v4_do_rcv (\root\codes\kernel-dev\linux\net\ipv4\tcp_ipv4.c:1751)
__release_sock (\root\codes\kernel-dev\linux\net\core\sock.c:2983)
release_sock (\root\codes\kernel-dev\linux\net\core\sock.c:3520)
tcp_sendmsg (\root\codes\kernel-dev\linux\net\ipv4\tcp.c:1337)
inet_sendmsg (\root\codes\kernel-dev\linux\net\ipv4\af_inet.c:840)
sock_sendmsg_nosec (\root\codes\kernel-dev\linux\net\socket.c:730)
__sock_sendmsg (\root\codes\kernel-dev\linux\net\socket.c:745)
__sys_sendto (\root\codes\kernel-dev\linux\net\socket.c:2194)
__do_sys_sendto (\root\codes\kernel-dev\linux\net\socket.c:2206)
__se_sys_sendto (\root\codes\kernel-dev\linux\net\socket.c:2202)
__x64_sys_sendto (\root\codes\kernel-dev\linux\net\socket.c:2202)
do_syscall_x64 (\root\codes\kernel-dev\linux\arch\x86\entry\common.c:50)
do_syscall_64 (\root\codes\kernel-dev\linux\arch\x86\entry\common.c:80)
entry_SYSCALL_64 (\root\codes\kernel-dev\linux\arch\x86\entry\entry_64.S:120)
48 (@48..88:3)
```


## 参考
> [networking:kernel\_flow [Wiki]](https://wiki.linuxfoundation.org/networking/kernel_flow)