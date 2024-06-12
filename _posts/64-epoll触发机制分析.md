---
title: epoll触发机制分析
date: 2023-11-07
updated: 2023-11-16
issueid: 64
tags:
- Kernel
---
目标：

    分析 kernel 驱动收到后skb

    1. 如何唤醒epoll callback？
    2. accpet syscall 如何从listener socket spawn 子 socket？

一个极简epoll server程序（无法编译，只说明流程）

```c
// 创建listener socket
int listener_fd = socket();
bind(listener_fd);
listen(listener_fd);
set_socket_non_blocking(listener_fd);

// 创建socket
ep = epoll_create1(0);
struct epoll_event event;
event.data.fd = listener_fd;
event.events = EPOLLIN | EPOLLET;
// 将listener socket注册进epoll
s = epoll_ctl(efd, EPOLL_CTL_ADD, sfd, &event);
for ( ;; ) {
    // eventloop
    int n = epoll_wait(ep, &events, 1024, -1);
    for (itn i = 0; i < n ; i ++) {
        if(events[i].data.fd == listener_fd) {
            // new incoming connection
            int child = accept(listener_fd, &addr, &addr_len);
            // 将子连接socket加入epoll eventloop
            make_socket_non_blocking(infd);
            struct epoll_event;
            event.data.fd = infd;
            event.events = EPOLLIN | EPOLLET;
            s = epoll_ctl(ep, EPOLL_CTL_ADD, child, &event);
        }
    }
}
```


## epoll  注册 wake  callback

直接看追踪的call stack

1. `epoll_ctl(ep, EPOLL_CTL_ADD, child, &event)`: called from epoll_ctl syscall
2. [ep_insert](https://elixir.bootlin.com/linux/v6.6/source/fs/eventpoll.c#L1552):
```c
// ep_insert
epq.epi = epi;
init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);
```

把 `ep_ptable_queue_proc` 放入 `_qproc`，在接下来的 `poll_wait` 调用
 ```c
static inline void init_poll_funcptr(poll_table *pt, poll_queue_proc qproc)
{
	pt->_qproc = qproc;
	pt->_key   = ~(__poll_t)0; /* all events enabled */
}

```


3. ep_item_poll
    1. 如果fd不是file epoll，那么通过vfs定向给具体实现
4. vfs_poll: virtual fs poll，call `file->f_op->poll(file, pt);`
    1. sock poll: 文件底层对应的是sock
    2. unix_poll: address family unix poll
5. sock_poll: socket 文件类型的poll
6. tcp_poll: tcp socket poll 实现
7. sock_poll_wait
8. poll_wait
9. [ep_ptable_queue_proc](https://elixir.bootlin.com/linux/v6.6/source/fs/eventpoll.c#L1288)
        1. ep_poll_callback 添加到 wait queue
![](assets/2023-11-07-3.png)

## struct file 转换成 struct sock 

`struct sock` 有最核心弄明白的点在于skb buff 如何转换成 `struct socket` 结构

```c
static __poll_t sock_poll(struct file *file, poll_table *wait)
{
    // 关键点
	struct socket *sock = file->private_data;
	const struct proto_ops *ops = READ_ONCE(sock->ops);
	__poll_t events = poll_requested_events(wait), flag = 0;

	if (!ops->poll)
		return 0;

	if (sk_can_busy_loop(sock->sk)) {
		/* poll once if requested by the syscall */
		if (events & POLL_BUSY_LOOP)
			sk_busy_loop(sock->sk, 1);

		/* if this socket can poll_ll, tell the system call */
		flag = POLL_BUSY_LOOP;
	}

	return ops->poll(file, sock, wait) | flag;
}
```

sock_poll将 `file->private_data` 转换成 `struct sock`，从而调用到epoll的socket实现

[Why does the Linux kernel have \`struct sock\` and \`struct socket\`? - Stack Overflow](https://stackoverflow.com/a/46415123/7529562)

## sock wake epoll callback

唤醒`ep->wq`[ep_poll_callback](https://elixir.bootlin.com/linux/v6.6/source/fs/eventpoll.c#L1217)

![](/assets/2023-11-07-2.png)