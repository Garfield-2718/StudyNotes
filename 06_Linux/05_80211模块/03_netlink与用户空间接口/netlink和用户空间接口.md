# netlink和用户空间交互接口

[TOC]

## libnl库

**libnl** 是 `Linux` 平台上用于与内核 **Netlink** 协议通信的一套用户态库（ `LGPL 2.1` 许可），由 `Thomas Graf` 开发维护。它把内核通过 `Netlink socket` 暴露的各种网络配置接口封装成易用的 `C API`，是 `iproute2`（`ip`、`tc`、`bridge` 等命令）之外最重要的 `Netlink` 编程接口之一

**版本与组成**

现在通常指 **libnl3**（`libnl1`/`libnl2` 是旧版，`API` 不兼容）。它拆成多个子库，按需链接：

| 库              | 用途                                                         |
| :-------------- | :----------------------------------------------------------- |
| `libnl-3`       | 核心：`socket` 管理、消息构造/解析、通用 `Netlink`           |
| `libnl-route-3` | `rtnetlink`：接口、地址、路由、邻居、`qdisc`、`class`、`filter` |
| `libnl-genl-3`  | `Generic Netlink`（如 `nl80211`、`taskstats`）               |
| `libnl-nf-3`    | `netfilter` 相关（`nfnetlink`、`conntrack`、`queue`、`log`） |
| `libnl-cli-3`   | 命令行辅助工具                                               |
| `libnl-idiag-3` | `inet_diag`（查询 `socket` 状态，类似 `ss`）                 |
| `libnl-xfrm-3`  | `IPsec/XFRM` 状态与策略                                      |

## 原始socket通信

在使用原始`socket`操作`nl80211`时应使用 `AF_NETLINK`（`PF_NETLINK`），协议号填 `NETLINK_GENERIC`

```c
int fd = socket(AF_NETLINK, SOCK_RAW, NETLINK_GENERIC);
```

`nl80211` 不是独立的地址族，而是挂在 `Generic Netlink`（通用 `netlink`）上的一个 `family`（家族），内核里注册名为 `"nl80211"`：

```c
// linux-7.3\net\wireless\nl80211.c

static struct genl_family nl80211_fam __ro_after_init = {
	.name = NL80211_GENL_NAME,	/* have users key off the name instead */
	.hdrsize = 0,			/* no private header */
    // ......
}
```

**对应关系是：**

| 层级        | 取值              | 含义                                               |
| :---------- | :---------------- | :------------------------------------------------- |
| `family`    | `AF_NETLINK`      | `netlink` 地址族                                   |
| `type`      | `SOCK_RAW`        | `netlink` 常规用法                                 |
| `protocol`  | `NETLINK_GENERIC` | `Generic Netlink` 总线                             |
| `family` 名 | `nl80211`         | 再通过 `CTRL_CMD_GETFAMILY` 解析出动态 `family id` |

### 设计的取舍

`nl80211`没有做成一个单独的协议族，主要原因是`AF_NETLINK`协议族在设计之初`protocol`的数量被限制在了`32`个，总线用尽以后，`16` 号 `NETLINK_GENERIC` 成了扩展点：新接口注册 `family` 名字，运行时分配 `family id`，需要时再手动查询`family id`，不再申请新的 `NETLINK_*`。**如果强行修改这个值会导致`ABI`的改变**

### wifi扫描实例

> 笔者注：由于排版问题，具体内容见“原始socket发起wifi扫描代码实例”章节

## nl80211协议规范

`nl80211` 不是独立二进制协议，而是跑在 `Generic Netlink`（通用 `Netlink`）上的 `family`。`nl80211` 没有自己的二进制帧格式，它的完整布局就是 `Generic Netlink` 消息：`nlmsghdr` 后面紧跟 `genlmsghdr`，再后面是一串 `nlattr`

```C
[ nlmsghdr ][ genlmsghdr ][ nlattr ... ]
    ↑            ↑
公共路由头   family 内命令头
```

### 整体布局

```c
+------------------+
|  nlmsghdr        |  Netlink 公共头（16 字节）
+------------------+
|  genlmsghdr      |  Generic Netlink 头（4 字节对齐）
+------------------+
|  nlattr ...      |  nl80211 属性（TLV，4 字节对齐）
|  nlattr ...      |
|  ...             |
+------------------+
```

### 详细布局

```c
<--------------------------- skb / recv 缓冲区 ---------------------------->
+--------------------------- 一条 Netlink 消息 ----------------------------+
|                                                                        |
|  <--- nlmsg_total_size()(含尾部 Pad，NLMSG_ALIGN 到 4) --->              |
|  <-- nlmsg_len(含头，不含尾部 Pad) -->                                    |
|                                                                        |
|  +---------------- nlmsghdr(16 字节) ----------------+                  |
|  | u32 nlmsg_len    本条总长(含本头)                   |                  |
|  | u16 nlmsg_type   family id("nl80211" 动态分配)     |                  |
|  | u16 nlmsg_flags  NLM_F_REQUEST / ACK / DUMP ...  |                  |
|  | u32 nlmsg_seq    序号                             |                  |
|  | u32 nlmsg_pid    port id，用户态请求一般是 0         |                  |
|  +-----------------------------------------------------+               |
|  | Pad |  把 nlmsghdr 对齐到 4；16 已对齐，这里为 0                         |
|                                                                         |
|  <------ hdrlen = GENL_HDRLEN + hdrsize = 4 + 0 ------>                 |
|  +---------------- genlmsghdr(4 字节) ----------------+                  |
|  | u8  cmd        NL80211_CMD_*                       |                 |
|  | u8  version    family->version，现在是 1            |                 |
|  | u16 reserved   0                                   |                 |
|  +----------------------------------------------------+                 |
|  | Pad |  GENL_HDRLEN 已是 4，这里为 0                                    |
|                                                                         |
|  nlmsg_attrdata(nlh, 4) 从这里开始                                        |
|  +---------------- nlattr(NL80211_ATTR_*) -------------+                |
|  | u16 nla_len    含这 4 字节头，不含尾部 Pad              |                |
|  | u16 nla_type   低 14 位是属性号                       |                 |
|  |            bit15 NLA_F_NESTED  嵌套                  |                |
|  |            bit14 NLA_F_NET_BYTEORDER                |                |
|  | payload        u32 / MAC / 二进制 / 又一层 nlattr     |                 |
|  | Pad            把本属性对齐到 4                        |                |
|  +------------------------------------------------------+                |
|  +---------------- 下一个 nlattr ------------------------+                |
|  | ...                                                  |                |
|  +------------------------------------------------------+                |
|                                                                          |
|  | Pad |  把整条消息对齐到 4，nlmsg_next() 跳过它                             |
+--------------------------------------------------------------------------+
| 下一条 nlmsghdr ...(dump 时 NLM_F_MULTI，最后一条是 NLMSG_DONE)              |
+--------------------------------------------------------------------------+
```

#### 1. nlmsghdr（Netlink 消息头）

`Netlink` 公共消息头，告诉内核/用户态「这是一条多长的消息、发给谁处理、要不要 `ACK`」

| 字段          | 含义（对 nl80211）                                           |
| :------------ | :----------------------------------------------------------- |
| `nlmsg_len`   | 整条消息长度（含本头）                                       |
| `nlmsg_type`  | `family id`（动态分配，先用 `CTRL_CMD_GETFAMILY` 查 `"nl80211"`） |
| `nlmsg_flags` | 如 `NLM_F_REQUEST`、`NLM_F_ACK`、`NLM_F_DUMP`                |
| `nlmsg_seq`   | 序号，匹配应答                                               |
| `nlmsg_pid`   | `port id`，用户态通常填 `0`                                  |

#### 2. genlmsghdr（Generic Netlink 头）

`Generic Netlink` 协议头，在某个 `family` 内部再选「做什么操作」

| 字段       | 含义                                                         |
| :--------- | :----------------------------------------------------------- |
| `cmd`      | `nl80211` 命令，如 `NL80211_CMD_TRIGGER_SCAN`、`NL80211_CMD_GET_SCAN` |
| `version`  | 一般填 `1`                                                   |
| `reserved` | 填 `0`                                                       |

命令枚举在 `linux/nl80211.h` 的 `enum nl80211_commands`。

#### 3. nlattr（属性 TLV）

```c
+---------+----------+------------------+-----+
| nla_len | nla_type |     payload      | pad |
+---------+----------+------------------+-----+
     2B       2B        nla_len - 4       对齐到 4
```

- `nla_len`：含属性头，不含尾部对齐填充

- `nla_type`：低 `14` 位是属性 `ID`（`NL80211_ATTR_*`）；可带 `NLA_F_NESTED`（嵌套）等标志

    

- `payload` 类型决定每种载荷形态：`u32`、`u8`、`MAC`、嵌套属性等

    **常见属性：**

    | 属性                            | 典型内容             |
    | :------------------------------ | :------------------- |
    | `NL80211_ATTR_IFINDEX`          | `u32` 网卡 `ifindex` |
    | `NL80211_ATTR_WIPHY`            | `u32 wiphy index`    |
    | `NL80211_ATTR_MAC`              | `6` 字节 `MAC`       |
    | `NL80211_ATTR_SSID`             | `SSID` 字节串        |
    | `NL80211_ATTR_SCAN_FREQUENCIES` | 嵌套：一串频率 `u32` |
    | `NL80211_ATTR_SCAN_SSIDS`       | 嵌套：一串 `SSID`    |

## 原始socket发起wifi扫描代码实例

```c
/* nl_scan.c — 不用 libnl，直接用 netlink socket 发起一次 nl80211 扫描 */
#define _GNU_SOURCE
#include <errno.h>
#include <poll.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <stdint.h>
#include <net/if.h>
#include <sys/socket.h>
#include <linux/netlink.h>
#include <linux/genetlink.h>
#include <linux/nl80211.h>

/*
* data 用来拼一条待发送的 Netlink 消息（nlmsghdr + genlmsghdr + nlattr）。
* 8192 只是缓冲区上限，sendto 按 nlmsg_len 发送，不会把整块都发出去。
* 这个示例里的请求只有几十个字节；8192 与内核组包常用的 NLMSG_GOODSIZE
* 一致，给以后追加 SSID、频率、Information Element 留余量。
* len 记录 data 里已经写入的有效字节数。
*/
struct nlbuf {
    char data[8192];
    size_t len;
};

static void die(const char *msg)
{
    perror(msg);
    exit(1);
}

/* 组一条 Generic Netlink 请求：nlmsghdr + genlmsghdr */
static void buf_reset(struct nlbuf *b, uint16_t type, uint16_t flags,
            uint8_t cmd, uint32_t seq)
{
    struct nlmsghdr *nlh;
    struct genlmsghdr *gh;

    memset(b, 0, sizeof(*b));
    nlh = (struct nlmsghdr *)b->data;

    nlh->nlmsg_len = NLMSG_LENGTH(GENL_HDRLEN);
    nlh->nlmsg_type = type;
    nlh->nlmsg_flags = flags;
    nlh->nlmsg_seq = seq;

    gh = NLMSG_DATA(nlh);
    gh->cmd = cmd;
    gh->version = 1;
    b->len = nlh->nlmsg_len;
}

/* 追加一个 netlink attribute（属性）。nla_len 含头部，不含 4 字节对齐填充 */
static void nla_put(struct nlbuf *b, uint16_t type, const void *payload, int plen)
{
    struct nlmsghdr *nlh = (struct nlmsghdr *)b->data;
    struct nlattr *nla;
    int pos = NLA_ALIGN(b->len);

    if (pos + NLA_HDRLEN + plen > (int)sizeof(b->data))
        die("netlink 消息过长");
    if (pos > (int)b->len)
        memset(b->data + b->len, 0, pos - b->len);

    nla = (struct nlattr *)(b->data + pos);
    nla->nla_type = type;
    nla->nla_len = NLA_HDRLEN + plen;
    if (plen && payload)
        memcpy((char *)nla + NLA_HDRLEN, payload, plen);

    b->len = pos + NLA_ALIGN(nla->nla_len);
    nlh->nlmsg_len = b->len;
}

static int nla_start(struct nlbuf *b, uint16_t type)
{
    int off = NLA_ALIGN(b->len);

    nla_put(b, type, NULL, 0);

    return off;
}

static void nla_end(struct nlbuf *b, int off)
{
    struct nlattr *nla = (struct nlattr *)(b->data + off);

    nla->nla_len = b->len - off;
}

static int nl_send(int fd, struct nlbuf *b)
{
    struct sockaddr_nl dst = { .nl_family = AF_NETLINK };
    struct nlmsghdr *nlh = (struct nlmsghdr *)b->data;
    ssize_t n;

    n = sendto(fd, b->data, nlh->nlmsg_len, 0, (struct sockaddr *)&dst, sizeof(dst));
    if (n < 0)
        return -1;

    return 0;
}

/* 在一段 attribute 流里按类型查找。type 要去掉 NLA_F_NESTED 标志 */
static struct nlattr *nla_find(void *start, int len, uint16_t type)
{
    struct nlattr *nla = start;
    int rem = len;

    while (rem >= (int)sizeof(*nla) &&
        nla->nla_len >= sizeof(*nla) &&
        nla->nla_len <= (uint16_t)rem) {
        if ((nla->nla_type & NLA_TYPE_MASK) == type)
            return nla;

        rem -= NLA_ALIGN(nla->nla_len);
        nla = (struct nlattr *)((char *)nla + NLA_ALIGN(nla->nla_len));
    }

    return NULL;
}

static void *nla_data(struct nlattr *nla)
{
    return (char *)nla + NLA_HDRLEN;
}

static int nla_payload(struct nlattr *nla)
{
    return nla->nla_len - NLA_HDRLEN;
}

/* 读一次 socket，直到找到指定 seq 的 ACK，或者超时。返回 0 表示成功 ACK */
static int wait_ack(int fd, uint32_t seq, int timeout_ms)
{
    char buf[8192];
    struct pollfd pfd = { .fd = fd, .events = POLLIN };

    while (poll(&pfd, 1, timeout_ms) > 0) {
        ssize_t n = recv(fd, buf, sizeof(buf), 0);
        struct nlmsghdr *nlh;

        if (n < 0) {
            if (errno == EINTR)
                continue;
            return -1;
        }

        for (nlh = (struct nlmsghdr *)buf; NLMSG_OK(nlh, n); nlh = NLMSG_NEXT(nlh, n)) {
            if (nlh->nlmsg_type != NLMSG_ERROR || nlh->nlmsg_seq != seq)
                continue;

            struct nlmsgerr *err = NLMSG_DATA(nlh);
            if (err->error) {
                errno = -err->error;
                return -1;
            }
            return 0;
        }
    }

    errno = ETIMEDOUT;
    return -1;
}

/*
* 向 Generic Netlink 控制器查询 "nl80211"：
* family id 是 u16；组播组在 CTRL_ATTR_MCAST_GROUPS 里，id 是 u32。
*/
static int resolve_nl80211(int fd, uint32_t seq, uint16_t *family, int *scan_grp)
{
    struct nlbuf b;
    char rbuf[8192];
    const char *name = NL80211_GENL_NAME;
    struct sockaddr_nl dst = { .nl_family = AF_NETLINK };
    ssize_t n;
    struct nlmsghdr *nlh;

    buf_reset(&b, GENL_ID_CTRL, NLM_F_REQUEST, CTRL_CMD_GETFAMILY, seq);
    nla_put(&b, CTRL_ATTR_FAMILY_NAME, name, strlen(name) + 1);
    if (sendto(fd, b.data, ((struct nlmsghdr *)b.data)->nlmsg_len, 0,
        (struct sockaddr *)&dst, sizeof(dst)) < 0)
        return -1;

    n = recv(fd, rbuf, sizeof(rbuf), 0);
    if (n < 0)
        return -1;

    *family = 0;
    *scan_grp = -1;
    for (nlh = (struct nlmsghdr *)rbuf; NLMSG_OK(nlh, n); nlh = NLMSG_NEXT(nlh, n)) {
        struct genlmsghdr *gh;
        struct nlattr *id, *grps, *grp;
        int rem;

        if (nlh->nlmsg_type == NLMSG_ERROR) {
            struct nlmsgerr *err = NLMSG_DATA(nlh);
            if (err->error) {
                errno = -err->error;
                return -1;
            }
            continue;
        }

        if (nlh->nlmsg_type != GENL_ID_CTRL)
            continue;

        gh = NLMSG_DATA(nlh);
        if (gh->cmd != CTRL_CMD_NEWFAMILY)
            continue;

        id = nla_find(gh + 1, nlh->nlmsg_len - NLMSG_LENGTH(GENL_HDRLEN), CTRL_ATTR_FAMILY_ID);
        if (!id || nla_payload(id) < (int)sizeof(uint16_t))
            continue;
        *family = *(uint16_t *)nla_data(id);

        grps = nla_find(gh + 1, nlh->nlmsg_len - NLMSG_LENGTH(GENL_HDRLEN), CTRL_ATTR_MCAST_GROUPS);
        if (!grps)
            continue;

        grp = nla_data(grps);
        rem = nla_payload(grps);
        while (rem >= (int)sizeof(*grp) &&
                grp->nla_len >= sizeof(*grp) &&
                grp->nla_len <= (uint16_t)rem) {
            struct nlattr *gname, *gid;

            gname = nla_find(nla_data(grp), nla_payload(grp),
                        CTRL_ATTR_MCAST_GRP_NAME);
            gid = nla_find(nla_data(grp), nla_payload(grp),
                    CTRL_ATTR_MCAST_GRP_ID);

            if (gname && gid && strcmp(nla_data(gname), NL80211_MULTICAST_GROUP_SCAN) == 0)
                *scan_grp = *(uint32_t *)nla_data(gid);

            rem -= NLA_ALIGN(grp->nla_len);
            grp = (struct nlattr *)((char *)grp + NLA_ALIGN(grp->nla_len));
        }
    }

    if (!*family || *scan_grp < 0) {
        errno = ENODEV;
        return -1;
    }

    return 0;
}

/* 等 "scan" 组播：扫描结束是 NEW_SCAN_RESULTS，失败是 SCAN_ABORTED */
static int wait_scan_done(int fd, uint16_t family, int timeout_ms)
{
    char buf[8192];
    struct pollfd pfd = { .fd = fd, .events = POLLIN };

    while (poll(&pfd, 1, timeout_ms) > 0) {
        ssize_t n = recv(fd, buf, sizeof(buf), 0);
        struct nlmsghdr *nlh;

        if (n < 0) {
            if (errno == EINTR)
                continue;
            return -1;
        }

        for (nlh = (struct nlmsghdr *)buf; NLMSG_OK(nlh, n); nlh = NLMSG_NEXT(nlh, n)) {
            struct genlmsghdr *gh;
            if (nlh->nlmsg_type != family)
                continue;

            gh = NLMSG_DATA(nlh);
            if (gh->cmd == NL80211_CMD_NEW_SCAN_RESULTS)
                return 0;

            if (gh->cmd == NL80211_CMD_SCAN_ABORTED) {
                errno = ECANCELED;
                return -1;
            }
        }
    }

    errno = ETIMEDOUT;
    return -1;
}

/* 从 802.11 Information Element 里取出 Element ID 0，也就是 SSID */
static void print_ssid(const uint8_t *ie, int len)
{
    int i = 0;

    while (i + 2 <= len) {
        int id = ie[i];
        int elen = ie[i + 1];

        if (i + 2 + elen > len)
            break;

        if (id == 0) {
            printf(" ssid=\"");
            fwrite(ie + i + 2, 1, elen, stdout);
            printf("\"");
            return;
        }

        i += 2 + elen;
    }
}

static void dump_bss(int fd, uint16_t family, uint32_t seq, uint32_t ifindex)
{
    struct nlbuf b;
    char rbuf[16384];
    uint32_t ifindex_attr = ifindex;

    buf_reset(&b, family, NLM_F_REQUEST | NLM_F_DUMP, NL80211_CMD_GET_SCAN, seq);
    nla_put(&b, NL80211_ATTR_IFINDEX, &ifindex_attr, sizeof(ifindex_attr));
    if (nl_send(fd, &b) < 0)
        die("发送 GET_SCAN");

    for (;;) {
        ssize_t n = recv(fd, rbuf, sizeof(rbuf), 0);
        struct nlmsghdr *nlh;
        int done = 0;

        if (n < 0) {
            if (errno == EINTR)
                continue;
            die("接收扫描结果");
        }

        for (nlh = (struct nlmsghdr *)rbuf; NLMSG_OK(nlh, n); nlh = NLMSG_NEXT(nlh, n)) {
            struct genlmsghdr *gh;
            struct nlattr *bss, *mac, *ie, *sig;

            if (nlh->nlmsg_type == NLMSG_DONE ||
                (nlh->nlmsg_type == NLMSG_ERROR && nlh->nlmsg_seq == seq)) {
                done = 1;
                continue;
            }

            if (nlh->nlmsg_type != family || nlh->nlmsg_seq != seq)
                continue;

            gh = NLMSG_DATA(nlh);
            bss = nla_find(gh + 1, nlh->nlmsg_len - NLMSG_LENGTH(GENL_HDRLEN), NL80211_ATTR_BSS);
            if (!bss)
                continue;

            mac = nla_find(nla_data(bss), nla_payload(bss), NL80211_BSS_BSSID);
            sig = nla_find(nla_data(bss), nla_payload(bss), NL80211_BSS_SIGNAL_MBM);
            ie = nla_find(nla_data(bss), nla_payload(bss), NL80211_BSS_INFORMATION_ELEMENTS);
            if (mac && nla_payload(mac) >= 6) {
                uint8_t *m = nla_data(mac);
                printf("%02x:%02x:%02x:%02x:%02x:%02x",
                    m[0], m[1], m[2], m[3], m[4], m[5]);
            }

            if (sig && nla_payload(sig) >= (int)sizeof(int32_t))
                printf(" signal=%d dBm", (*(int32_t *)nla_data(sig)) / 100);

            if (ie)
                print_ssid(nla_data(ie), nla_payload(ie));
            printf("\n");
        }

        if (done)
            break;
    }
}

int main(int argc, char **argv)
{
    int fd, scan_grp;
    uint16_t family;
    uint32_t ifindex, seq = 1;
    struct sockaddr_nl local = { .nl_family = AF_NETLINK };
    struct nlbuf b;
    uint32_t ifindex_attr;
    int nest;

    if (argc != 2) {
        fprintf(stderr, "用法: %s <无线网卡名>\n", argv[0]);
        return 1;
    }

    ifindex = if_nametoindex(argv[1]);
    if (!ifindex)
        die("if_nametoindex");

    fd = socket(AF_NETLINK, SOCK_RAW, NETLINK_GENERIC);
    if (fd < 0)
        die("socket");
    if (bind(fd, (struct sockaddr *)&local, sizeof(local)) < 0)
        die("bind");

    if (resolve_nl80211(fd, seq++, &family, &scan_grp) < 0)
        die("查询 nl80211 family");
    printf("nl80211 family=%u scan group=%d\n", family, scan_grp);

    /* group id 可能大于 32，不能写进 sockaddr_nl.nl_groups */
    if (setsockopt(fd, SOL_NETLINK, NETLINK_ADD_MEMBERSHIP,
            &scan_grp, sizeof(scan_grp)) < 0)
        die("加入 scan 组播组");

    /* 空 SSID：主动扫描，probe request 使用广播 SSID */
    ifindex_attr = ifindex;
    buf_reset(&b, family, NLM_F_REQUEST | NLM_F_ACK, NL80211_CMD_TRIGGER_SCAN, seq);
    nla_put(&b, NL80211_ATTR_IFINDEX, &ifindex_attr, sizeof(ifindex_attr));
    nest = nla_start(&b, NL80211_ATTR_SCAN_SSIDS);
    nla_put(&b, 1, "", 0);
    nla_end(&b, nest);

    if (nl_send(fd, &b) < 0)
        die("发送 TRIGGER_SCAN");
    if (wait_ack(fd, seq, 3000) < 0)
        die("TRIGGER_SCAN 被拒绝");
    printf("扫描已开始，等待结果...\n");

    if (wait_scan_done(fd, family, 20000) < 0)
        die("等待扫描完成");

    dump_bss(fd, family, ++seq, ifindex);
    close(fd);

    return 0;
}
```

**编译和运行**

```shell
gcc -Wall -o nl_scan nl_scan.c
sudo ip link set wlan0 up
sudo ./nl_scan wlan0
```

## 参考文档

[Release v7.3 · torvalds/linux · GitHub](https://github.com/torvalds/linux/releases/tag/v7.3)

