# sk_buff

非常有名的结构体`struct sk_buff`中的`sk`便是`socket`的缩写

## 原始结构体

> 笔者注：此处代码截取自`Linux 7.3`源码未作任何改动

```c
// linux-7.3\include\linux\skbuff.h
/**
 * DOC: Basic sk_buff geometry
 *
 * struct sk_buff itself is a metadata structure and does not hold any packet
 * data. All the data is held in associated buffers.
 *
 * &sk_buff.head points to the main "head" buffer. The head buffer is divided
 * into two parts:
 *
 *  - data buffer, containing headers and sometimes payload;
 *    this is the part of the skb operated on by the common helpers
 *    such as skb_put() or skb_pull();
 *  - shared info (struct skb_shared_info) which holds an array of pointers
 *    to read-only data in the (page, offset, length) format.
 *
 * Optionally &skb_shared_info.frag_list may point to another skb.
 *
 * Basic diagram may look like this::
 *
 *                                  ---------------
 *                                 | sk_buff       |
 *                                  ---------------
 *     ,---------------------------  + head
 *    /          ,-----------------  + data
 *   /          /      ,-----------  + tail
 *  |          |      |            , + end
 *  |          |      |           |
 *  v          v      v           v
 *   -----------------------------------------------
 *  | headroom | data |  tailroom | skb_shared_info |
 *   -----------------------------------------------
 *                                 + [page frag]
 *                                 + [page frag]
 *                                 + [page frag]
 *                                 + [page frag]       ---------
 *                                 + frag_list    --> | sk_buff |
 *                                                     ---------
 *
 */

/**
 *	struct sk_buff - socket buffer
 *	@next: Next buffer in list
 *	@prev: Previous buffer in list
 *	@tstamp: Time we arrived/left
 *	@skb_mstamp_ns: (aka @tstamp) earliest departure time; start point
 *		for retransmit timer
 *	@rbnode: RB tree node, alternative to next/prev for netem/tcp
 *	@list: queue head
 *	@ll_node: anchor in an llist (eg socket defer_list)
 *	@sk: Socket we are owned by
 *	@dev: Device we arrived on/are leaving by
 *	@dev_scratch: (aka @dev) alternate use of @dev when @dev would be %NULL
 *	@cb: Control buffer. Free for use by every layer. Put private vars here
 *	@_skb_refdst: destination entry (with norefcount bit)
 *	@len: Length of actual data
 *	@data_len: Data length
 *	@mac_len: Length of link layer header
 *	@hdr_len: writable header length of cloned skb
 *	@csum: Checksum (must include start/offset pair)
 *	@csum_start: Offset from skb->head where checksumming should start
 *	@csum_offset: Offset from csum_start where checksum should be stored
 *	@priority: Packet queueing priority
 *	@ignore_df: allow local fragmentation
 *	@cloned: Head may be cloned (check refcnt to be sure)
 *	@ip_summed: Driver fed us an IP checksum
 *	@nohdr: Payload reference only, must not modify header
 *	@pkt_type: Packet class
 *	@fclone: skbuff clone status
 *	@ipvs_property: skbuff is owned by ipvs
 *	@inner_protocol_type: whether the inner protocol is
 *		ENCAP_TYPE_ETHER or ENCAP_TYPE_IPPROTO
 *	@remcsum_offload: remote checksum offload is enabled
 *	@offload_fwd_mark: Packet was L2-forwarded in hardware
 *	@offload_l3_fwd_mark: Packet was L3-forwarded in hardware
 *	@tc_skip_classify: do not classify packet. set by IFB device
 *	@tc_at_ingress: used within tc_classify to distinguish in/egress
 *	@redirected: packet was redirected by packet classifier
 *	@from_ingress: packet was redirected from the ingress path
 *	@nf_skip_egress: packet shall skip nf egress - see netfilter_netdev.h
 *	@peeked: this packet has been seen already, so stats have been
 *		done for it, don't do them again
 *	@nf_trace: netfilter packet trace flag
 *	@protocol: Packet protocol from driver
 *	@destructor: Destruct function
 *	@tcp_tsorted_anchor: list structure for TCP (tp->tsorted_sent_queue)
 *	@_sk_redir: socket redirection information for skmsg
 *	@_nfct: Associated connection, if any (with nfctinfo bits)
 *	@skb_iif: ifindex of device we arrived on
 *	@tc_depth: counter for packet duplication
 *	@tc_index: Traffic control index
 *	@hash: the packet hash
 *	@queue_mapping: Queue mapping for multiqueue devices
 *	@head_frag: skb was allocated from page fragments,
 *		not allocated by kmalloc() or vmalloc().
 *	@pfmemalloc: skbuff was allocated from PFMEMALLOC reserves
 *	@pp_recycle: mark the packet for recycling instead of freeing (implies
 *		page_pool support on driver)
 *	@active_extensions: active extensions (skb_ext_id types)
 *	@ndisc_nodetype: router type (from link layer)
 *	@ooo_okay: allow the mapping of a socket to a queue to be changed
 *	@l4_hash: indicate hash is a canonical 4-tuple hash over transport
 *		ports.
 *	@sw_hash: indicates hash was computed in software stack
 *	@wifi_acked_valid: wifi_acked was set
 *	@wifi_acked: whether frame was acked on wifi or not
 *	@no_fcs:  Request NIC to treat last 4 bytes as Ethernet FCS
 *	@encapsulation: indicates the inner headers in the skbuff are valid
 *	@encap_hdr_csum: software checksum is needed
 *	@csum_valid: checksum is already valid
 *	@csum_not_inet: use CRC32c to resolve CHECKSUM_PARTIAL
 *	@csum_complete_sw: checksum was completed by software
 *	@csum_level: indicates the number of consecutive checksums found in
 *		the packet minus one that have been verified as
 *		CHECKSUM_UNNECESSARY (max 3)
 *	@unreadable: indicates that at least 1 of the fragments in this skb is
 *		unreadable.
 *	@dst_pending_confirm: need to confirm neighbour
 *	@decrypted: Decrypted SKB
 *	@slow_gro: state present at GRO time, slower prepare step required
 *	@tstamp_type: When set, skb->tstamp has the
 *		delivery_time clock base of skb->tstamp.
 *	@napi_id: id of the NAPI struct this skb came from
 *	@sender_cpu: (aka @napi_id) source CPU in XPS
 *	@alloc_cpu: CPU which did the skb allocation.
 *	@secmark: security marking
 *	@mark: Generic packet mark
 *	@reserved_tailroom: (aka @mark) number of bytes of free space available
 *		at the tail of an sk_buff
 *	@vlan_all: vlan fields (proto & tci)
 *	@vlan_proto: vlan encapsulation protocol
 *	@vlan_tci: vlan tag control information
 *	@inner_protocol: Protocol (encapsulation)
 *	@inner_ipproto: (aka @inner_protocol) stores ipproto when
 *		skb->inner_protocol_type == ENCAP_TYPE_IPPROTO;
 *	@inner_transport_header: Inner transport layer header (encapsulation)
 *	@inner_network_header: Network layer header (encapsulation)
 *	@inner_mac_header: Link layer header (encapsulation)
 *	@transport_header: Transport layer header
 *	@network_header: Network layer header
 *	@mac_header: Link layer header
 *	@kcov_handle: KCOV remote handle for remote coverage collection
 *	@tail: Tail pointer
 *	@end: End pointer
 *	@head: Head of buffer
 *	@data: Data head pointer
 *	@truesize: Buffer size
 *	@users: User count - see {datagram,tcp}.c
 *	@extensions: allocated extensions, valid if active_extensions is nonzero
 */

struct sk_buff {
	union {
		struct {
			/* These two members must be first to match sk_buff_head. */
			struct sk_buff		*next;
			struct sk_buff		*prev;

			union {
				struct net_device	*dev;
				/* Some protocols might use this space to store information,
				 * while device pointer would be NULL.
				 * UDP receive path is one user.
				 */
				unsigned long		dev_scratch;
			};
		};
		struct rb_node		rbnode; /* used in netem, ip4 defrag, and tcp stack */
		struct list_head	list;
		struct llist_node	ll_node;
	};

	struct sock		*sk;

	union {
		ktime_t		tstamp;
		u64		skb_mstamp_ns; /* earliest departure time */
	};
	/*
	 * This is the control buffer. It is free to use for every
	 * layer. Please put your private variables there. If you
	 * want to keep them across layers you have to do a skb_clone()
	 * first. This is owned by whoever has the skb queued ATM.
	 */
	char			cb[48] __aligned(8);

	union {
		struct {
			unsigned long	_skb_refdst;
			void		(*destructor)(struct sk_buff *skb);
		};
		struct list_head	tcp_tsorted_anchor;
#ifdef CONFIG_NET_SOCK_MSG
		unsigned long		_sk_redir;
#endif
	};

#if defined(CONFIG_NF_CONNTRACK) || defined(CONFIG_NF_CONNTRACK_MODULE)
	unsigned long		 _nfct;
#endif
	unsigned int		len,
				data_len;
	__u16			mac_len,
				hdr_len;

	/* Following fields are _not_ copied in __copy_skb_header()
	 * Note that queue_mapping is here mostly to fill a hole.
	 */
	__u16			queue_mapping;

/* if you move cloned around you also must adapt those constants */
#ifdef __BIG_ENDIAN_BITFIELD
#define CLONED_MASK	(1 << 7)
#else
#define CLONED_MASK	1
#endif
#define CLONED_OFFSET		offsetof(struct sk_buff, __cloned_offset)

	/* private: */
	__u8			__cloned_offset[0];
	/* public: */
	__u8			cloned:1,
				nohdr:1,
				fclone:2,
				peeked:1,
				head_frag:1,
				pfmemalloc:1,
				pp_recycle:1; /* page_pool recycle indicator */
#ifdef CONFIG_SKB_EXTENSIONS
	__u8			active_extensions;
#endif

	/* Fields enclosed in headers group are copied
	 * using a single memcpy() in __copy_skb_header()
	 */
	struct_group(headers,

	/* private: */
	__u8			__pkt_type_offset[0];
	/* public: */
	__u8			pkt_type:3; /* see PKT_TYPE_MAX */
	__u8			ignore_df:1;
	__u8			dst_pending_confirm:1;
	__u8			ip_summed:2;
	__u8			ooo_okay:1;

	/* private: */
	__u8			__mono_tc_offset[0];
	/* public: */
	__u8			tstamp_type:2;	/* See skb_tstamp_type */
#ifdef CONFIG_NET_XGRESS
	__u8			tc_at_ingress:1;	/* See TC_AT_INGRESS_MASK */
	__u8			tc_skip_classify:1;
#endif
	__u8			remcsum_offload:1;
	__u8			csum_complete_sw:1;
	__u8			csum_level:2;
	__u8			inner_protocol_type:1;

	__u8			l4_hash:1;
	__u8			sw_hash:1;
#ifdef CONFIG_WIRELESS
	__u8			wifi_acked_valid:1;
	__u8			wifi_acked:1;
#endif
	__u8			no_fcs:1;
	/* Indicates the inner headers are valid in the skbuff. */
	__u8			encapsulation:1;
	__u8			encap_hdr_csum:1;
	__u8			csum_valid:1;
#ifdef CONFIG_IPV6_NDISC_NODETYPE
	__u8			ndisc_nodetype:2;
#endif

#if IS_ENABLED(CONFIG_IP_VS)
	__u8			ipvs_property:1;
#endif
#if IS_ENABLED(CONFIG_NETFILTER_XT_TARGET_TRACE) || IS_ENABLED(CONFIG_NF_TABLES)
	__u8			nf_trace:1;
#endif
#ifdef CONFIG_NET_SWITCHDEV
	__u8			offload_fwd_mark:1;
	__u8			offload_l3_fwd_mark:1;
#endif
	__u8			redirected:1;
#ifdef CONFIG_NET_REDIRECT
	__u8			from_ingress:1;
#endif
#ifdef CONFIG_NET_EGRESS
	__u8			nf_skip_egress:1;
#endif
#ifdef CONFIG_SKB_DECRYPTED
	__u8			decrypted:1;
#endif
	__u8			slow_gro:1;
#if IS_ENABLED(CONFIG_IP_SCTP)
	__u8			csum_not_inet:1;
#endif
	__u8			unreadable:1;
	__u8			tc_depth:2;
#if defined(CONFIG_NET_SCHED) || defined(CONFIG_NET_XGRESS)
	__u16			tc_index;	/* traffic control index */
#endif

	u16			alloc_cpu;

	union {
		__wsum		csum;
		struct {
			__u16	csum_start;
			__u16	csum_offset;
		};
	};
	__u32			priority;
	int			skb_iif;
	__u32			hash;
	union {
		u32		vlan_all;
		struct {
			__be16	vlan_proto;
			__u16	vlan_tci;
		};
	};
#if defined(CONFIG_NET_RX_BUSY_POLL) || defined(CONFIG_XPS)
	union {
		unsigned int	napi_id;
		unsigned int	sender_cpu;
	};
#endif
#ifdef CONFIG_NETWORK_SECMARK
	__u32		secmark;
#endif

	union {
		__u32		mark;
		__u32		reserved_tailroom;
	};

	union {
		__be16		inner_protocol;
		__u8		inner_ipproto;
	};

	__u16			inner_transport_header;
	__u16			inner_network_header;
	__u16			inner_mac_header;

	__be16			protocol;
	__u16			transport_header;
	__u16			network_header;
	__u16			mac_header;

	struct kcov_common_handle_id kcov_handle;

	); /* end headers group */

	/* These elements must be at the end, see alloc_skb() for details.  */
	sk_buff_data_t		tail;
	sk_buff_data_t		end;
	unsigned char		*head,
				*data;
	unsigned int		truesize;
	refcount_t		users;

#ifdef CONFIG_SKB_EXTENSIONS
	/* only usable after checking ->active_extensions != 0 */
	struct skb_ext		*extensions;
#endif
};
```

## sk_buff数据包头

> 笔者注：此处代码截取自`Linux 7.3`源码未作任何改动

```c
struct sk_buff {
	union {
		struct {
			/* These two members must be first to match sk_buff_head. */
			struct sk_buff		*next;
			struct sk_buff		*prev;

			union {
				struct net_device	*dev;
				/* Some protocols might use this space to store information,
				 * while device pointer would be NULL.
				 * UDP receive path is one user.
				 */
				unsigned long		dev_scratch;
			};
		};
		struct rb_node		rbnode; /* used in netem, ip4 defrag, and tcp stack */
		struct list_head	list;
		struct llist_node	ll_node;
	};
    // .......
}
```

此处是`sk_buff`中最开头的一段`union`其中包含了四种不同的队列管理头

### sk_buff队列

 ```c
 struct sk_buff		*next;
 struct sk_buff		*prev;
 ```

**sk_buff双向链表队列**

经典的 `sk_buff` 双向循环队列。socket 收发队列、qdisc、驱动的 rx/tx 队列都走 `skb_queue_*`

**典型场景：**

1. **socket 接收队列**

这是最典型的一条。`struct sock` 里的 `sk_receive_queue` 就是 `sk_buff_head`。协议栈在软中断里把包挂到队尾，用户态 `recvmsg` 从队头取走

双向是因为要能从中间摘。丢弃、`MSG_PEEK` 之后截断、或者释放某个指定包，都要 `skb_unlink()`，有 `prev` 才能 `O(1)` 完成。

2. **TCP 发送队列**

应用 `send()` 组出来、还没发出去的段挂在 `sk->sk_write_queue` 上。`tcp_skb_entail()` 调 `tcp_add_write_queue_tail()` 接到尾部

同一个 `skb` 先用 `next`/`prev`，发出去之后改用外层 `union` 里的 `rbnode`。这也说明这两套指针不能同时有效

3. **每 CPU 的收包 backlog**

驱动在不能直接走 `NAPI` 的路径上调用 `netif_rx()`，或者 `RPS` 把包转到别的 `CPU`，都会进 `enqueue_to_backlog()`，挂到该 `CPU` 的 `softnet_data.input_pkt_queue`：

```c
__skb_queue_tail(&sd->input_pkt_queue, skb);
```

`NET_RX_SOFTIRQ` 里再整表搬到 `process_queue`，然后逐个送进协议栈：

```c
skb_queue_splice_tail_init(&sd->input_pkt_queue,
			   &sd->process_queue);
```

`splice` 要改四条指针（新链表的头尾和目标链表的前后邻居），所以必须是双向链表。队列过长时按 `skb` 从中间 `__skb_unlink()` 丢掉，同样依赖 `prev`

### 红黑树

```c
struct rb_node		rbnode;
```

**红黑树节点**

用红黑树时挂在这里。树按一个键值排序，插入、按序取、按键查找都是 `O(log n)`，适合「不能按 `FIFO`（先进先出）简单排队」的路径

**典型场景：**

- `TCP` 乱序队列
    -  `tcp_input.c` 把序列号不连续的段插进 `tp->out_of_order_queue`，之后按序列号归并
- `IPv4` 分片重组
    - `inet_fragment.c` 按分片偏移把分片放进 `q->rb_fragments`。注释也写了：`skb->rbnode` 和 `skb->dev` 共用同一块存储
- 延迟类 `qdisc`
    -  `netem` 按发送时间、`ETF`（`Earliest TxTime First`）按绝对发送时刻、`FQ`（`Fair Queue`）按流内发送时刻，都把 `skb` 挂进各自的时间红黑树

报文还在设备路径上时用 `dev`；一旦改挂到红黑树，这块内存就改由 `rbnode` 解释

### list_head

```c
struct list_head	list;
```

**普通双向链表**

用内核通用的 `list_head API`（`list_add_tail`、`list_del`、`list_for_each_entry`）时挂在这里。语义还是链表，但接口和 `sk_buff_head` 那套 `next`/`prev` 宏不同，也更方便和别的 `list_head` 对象混在一起

**典型场景：**

- `NAPI / GRO` 批量处理
    -  `net/core/gro.c`、`ip_input.c`、`ip6_input.c`、`netfilter` 把一批报文挂到临时 `sublist`，再整批交给上层
- `mac80211` 接收路径
    -  把当前批次的帧挂在 `rx->list` 上
- `MPTCP backlog`
    - 把待处理的 `skb` 挂进 `msk->backlog_list`

这些路径要的是「临时攒一批再处理」，不是 `socket` 接收队列那种长期持有的 `sk_buff_head`

### list_node

```c
struct llist_node	ll_node;
```

**无锁单链表**

用 `llist`（`lockless list`，无锁链表）时挂在这里。多个 `CPU` 可以并发 `llist_add()`，而不先拿自旋锁；消费端再一次性把整条链摘下来处理

**典型场景：**

- 跨 `CPU` 延迟释放
    -  `skbuff.c` / `dev.c` 把待释放的 `skb` 放进 `defer_list`，避免在硬中断或软中断里立刻做昂贵的释放
- `UDP` 生产者队列
    -  `udp.c` 用 `llist_add(&skb->ll_node, &udp_prod_queue->ll_root)` 把报文塞进无锁队列，后面再批量接管

## 参考文档

[Release v7.3 · torvalds/linux · GitHub](https://github.com/torvalds/linux/releases/tag/v7.3)
