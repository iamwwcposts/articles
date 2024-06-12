---
title: tcp-ip-rcv-skb-流程分析
date: 2023-11-16
updated: 2023-11-16
issueid: 65
tags:
- net
- Kernel
---
tcp rev + tcp syn send

```
__dev_xmit_skb (\root\codes\kernel-dev\linux\net\core\dev.c:3760)
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
__tcp_send_ack (\root\codes\kernel-dev\linux\net\ipv4\tcp_output.c:4082)
tcp_send_ack (\root\codes\kernel-dev\linux\net\ipv4\tcp_output.c:4088)
tcp_rcv_synsent_state_process (\root\codes\kernel-dev\linux\net\ipv4\tcp_input.c:6363)
tcp_rcv_state_process (\root\codes\kernel-dev\linux\net\ipv4\tcp_input.c:6539)
tcp_v4_do_rcv (\root\codes\kernel-dev\linux\net\ipv4\tcp_ipv4.c:1751)
tcp_v4_rcv (\root\codes\kernel-dev\linux\net\ipv4\tcp_ipv4.c:2150)
ip_protocol_deliver_rcu (\root\codes\kernel-dev\linux\net\ipv4\ip_input.c:205)
ip_local_deliver_finish (\root\codes\kernel-dev\linux\net\ipv4\ip_input.c:233)
NF_HOOK (\root\codes\kernel-dev\linux\include\linux\netfilter.h:304)
ip_local_deliver (\root\codes\kernel-dev\linux\net\ipv4\ip_input.c:254)
dst_input (\root\codes\kernel-dev\linux\include\net\dst.h:468)
ip_sublist_rcv_finish (\root\codes\kernel-dev\linux\net\ipv4\ip_input.c:580)
ip_list_rcv_finish (\root\codes\kernel-dev\linux\net\ipv4\ip_input.c:631)
ip_sublist_rcv (\root\codes\kernel-dev\linux\net\ipv4\ip_input.c:639)
ip_list_rcv (\root\codes\kernel-dev\linux\net\ipv4\ip_input.c:674)
__netif_receive_skb_list_ptype (\root\codes\kernel-dev\linux\net\core\dev.c:5570)
__netif_receive_skb_list_core (\root\codes\kernel-dev\linux\net\core\dev.c:5618)
__netif_receive_skb_list (\root\codes\kernel-dev\linux\net\core\dev.c:5670)
netif_receive_skb_list_internal (\root\codes\kernel-dev\linux\net\core\dev.c:5761)
gro_normal_list (\root\codes\kernel-dev\linux\include\net\gro.h:439)
napi_complete_done (\root\codes\kernel-dev\linux\net\core\dev.c:6101)
e1000_clean (\root\codes\kernel-dev\linux\drivers\net\ethernet\intel\e1000\e1000_main.c:3811)
__napi_poll (\root\codes\kernel-dev\linux\net\core\dev.c:6531)
napi_poll (\root\codes\kernel-dev\linux\net\core\dev.c:6598)
net_rx_action (\root\codes\kernel-dev\linux\net\core\dev.c:6731)
__do_softirq (\root\codes\kernel-dev\linux\kernel\softirq.c:553)
invoke_softirq (\root\codes\kernel-dev\linux\kernel\softirq.c:427)
__irq_exit_rcu (\root\codes\kernel-dev\linux\kernel\softirq.c:632)
irq_exit_rcu (\root\codes\kernel-dev\linux\kernel\softirq.c:644)
sysvec_apic_timer_interrupt (\root\codes\kernel-dev\linux\arch\x86\kernel\apic\apic.c:1074)
```
![](/assets/2023-10-23-16-52-49.png)


```c
static inline struct sock *__inet_lookup(struct net *net,
					 struct inet_hashinfo *hashinfo,
					 struct sk_buff *skb, int doff,
					 const __be32 saddr, const __be16 sport,
					 const __be32 daddr, const __be16 dport,
					 const int dif, const int sdif,
					 bool *refcounted)
{
	u16 hnum = ntohs(dport);
	struct sock *sk;

	sk = __inet_lookup_established(net, hashinfo, saddr, sport,
				       daddr, hnum, dif, sdif);
	*refcounted = true;
	if (sk)
		return sk;
	*refcounted = false;
	return __inet_lookup_listener(net, hashinfo, skb, doff, saddr,
				      sport, daddr, hnum, dif, sdif);
}
```
到TCP层后，skb需要转换成 `struct sock`，从 kernel 协议栈查找skb -> struct sk 映射

```c
struct sock *__inet_lookup_established(struct net *net,
				  struct inet_hashinfo *hashinfo,
				  const __be32 saddr, const __be16 sport,
				  const __be32 daddr, const u16 hnum,
				  const int dif, const int sdif)
{
	INET_ADDR_COOKIE(acookie, saddr, daddr);
	const __portpair ports = INET_COMBINED_PORTS(sport, hnum);
	struct sock *sk;
	const struct hlist_nulls_node *node;
	/* Optimize here for direct hit, only listening connections can
	 * have wildcards anyways.
	 */
	// hashtable查询
	unsigned int hash = inet_ehashfn(net, daddr, hnum, saddr, sport);
	// 找到hashtable桶
	unsigned int slot = hash & hashinfo->ehash_mask;
	struct inet_ehash_bucket *head = &hashinfo->ehash[slot];

begin:
	sk_nulls_for_each_rcu(sk, node, &head->chain) {
		if (sk->sk_hash != hash)
			continue;
		if (likely(inet_match(net, sk, acookie, ports, dif, sdif))) {
			if (unlikely(!refcount_inc_not_zero(&sk->sk_refcnt)))
				goto out;
			if (unlikely(!inet_match(net, sk, acookie,
						 ports, dif, sdif))) {
				sock_gen_put(sk);
				goto begin;
			}
			goto found;
		}
	}
	/*
	 * if the nulls value we got at the end of this lookup is
	 * not the expected one, we must restart lookup.
	 * We probably met an item that was moved to another chain.
	 */
	if (get_nulls_value(node) != slot)
		goto begin;
out:
	sk = NULL;
found:
	return sk;
}
```

sock#sk_data_ready 回调通知

![](/assets/2023-11-07-4.png)

注意看左侧callstack，协议栈收到数据后

1. tcp_rcv_established
    1. syn rcv 后进入 established 状态
    2. tcp_queue_rcv
        1. \_\_skb_queue_tail
    3. tcp_data_ready
1. tcp_data_queue
    1. tcp_queue_rcv
        1. \_\_skb_queue_tail
    2. tcp_data_ready
        1. `struct sock#sk_data_ready`默认是[sock_def_readable](https://elixir.bootlin.com/linux/v6.6/source/net/core/sock.c#L3447)

        唤醒

        ```c
        void sock_def_readable(struct sock *sk)
        {
        	struct socket_wq *wq;
        
        	trace_sk_data_ready(sk);
        
        	rcu_read_lock();
        	wq = rcu_dereference(sk->sk_wq);
        	// CC-NET-TCP 唤醒 sk 的 waiters
        	if (skwq_has_sleeper(wq))
        		wake_up_interruptible_sync_poll(&wq->wait, EPOLLIN | EPOLLPRI |
        						EPOLLRDNORM | EPOLLRDBAND);
        	sk_wake_async(sk, SOCK_WAKE_WAITD, POLL_IN);
        	rcu_read_unlock();
        }
        ```

### 如何唤醒accept syscall 

tcp server运行过程
> bind: 分配 port
> listen: Move a socket into listening state.
> blocking accept: thread blocking to require new socket


```c
bind();
listen();
for(;;) {
     connfd = accept(sockfd, (SA*)&cli, &len);
}
```

```c
const struct proto_ops inet_stream_ops = {
	.family		   = PF_INET,
	.owner		   = THIS_MODULE,
	.release	   = inet_release,
	.bind		   = inet_bind,
	.connect	   = inet_stream_connect,
	.socketpair	   = sock_no_socketpair,
	// accept syscall 调用到 tcp proto_ops 回调 inet_accept
	.accept		   = inet_accept,
	.getname	   = inet_getname,
	.poll		   = tcp_poll,
	.ioctl		   = inet_ioctl,
	.gettstamp	   = sock_gettstamp,
	// tcp listen
	.listen		   = inet_listen,
	.shutdown	   = inet_shutdown,
	.setsockopt	   = sock_common_setsockopt,
	.getsockopt	   = sock_common_getsockopt,
	.sendmsg	   = inet_sendmsg,
	.recvmsg	   = inet_recvmsg,
#ifdef CONFIG_MMU
	.mmap		   = tcp_mmap,
#endif
	.splice_eof	   = inet_splice_eof,
	.splice_read	   = tcp_splice_read,
	.read_sock	   = tcp_read_sock,
	.read_skb	   = tcp_read_skb,
	.sendmsg_locked    = tcp_sendmsg_locked,
	.peek_len	   = tcp_peek_len,
#ifdef CONFIG_COMPAT
	.compat_ioctl	   = inet_compat_ioctl,
#endif
	.set_rcvlowat	   = tcp_set_rcvlowat,
};
```

注册唤醒回调
accept blocking
[inet_csk_accept](https://elixir.bootlin.com/linux/v6.6/source/net/ipv4/inet_connection_sock.c#L683)
[prepare_to_wait_exclusive](https://elixir.bootlin.com/linux/v6.6/source/net/ipv4/inet_connection_sock.c#L609)
```c
bool
prepare_to_wait_exclusive(struct wait_queue_head *wq_head, struct wait_queue_entry *wq_entry, int state)
{
	unsigned long flags;
	bool was_empty = false;

	wq_entry->flags |= WQ_FLAG_EXCLUSIVE;
	spin_lock_irqsave(&wq_head->lock, flags);
	if (list_empty(&wq_entry->entry)) {
		was_empty = list_empty(&wq_head->head);
		__add_wait_queue_entry_tail(wq_head, wq_entry);
	}
	set_current_state(state);
	spin_unlock_irqrestore(&wq_head->lock, flags);
	return was_empty;
}
```


### 如何唤醒epoll等异步回调机制

未完待续...