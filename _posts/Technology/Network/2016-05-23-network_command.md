---

layout: post
title: Linux网络命令操作
category: 技术
tags: Network
keywords: network ovs

---

## 简介

* TOC
{:toc}

http://fishcried.com/ 有一个linux 网络的基础知识系列，要研读下

## tcpdump

[tcpdump抓包使用小结](https://segmentfault.com/a/1190000017346947)

    ## 监听某个网卡的数据包
    tcpdump -i eth0
    ## 监听某个端口的数据包，默认是第一个网卡     
    tcpdump port 1024
    ## 监控本机 eth0 网卡与目标主机的往来数据包
    tcpdump -i eth0 -nn 'host 目标主机ip'

Wireshark 还可以用来读取 tcpdump 保存的 pcap 文件。你可以使用 tcpdump 命令行在没有 GUI 界面的远程机器上抓包然后在 Wireshark 中分析数据包。

## ip命令

我们知道经典的OSI七层网络模型，学要致用，要将其掺入到对linux网络命令的理解中。

我们知道网卡有mac地址和ip地址，分别用在链路层和网络层。打个比方，MAC地址像是我们的身份证，到哪都是那个样子；IP像是居住证，换了地方信息就要变了。政府机构同时给公民发身份证和居住证以便管理动态的社会，网络管理机构则通过给所有的上网设备同时分配MAC和IP达到这个目的。（mac地址是和位置无关的，所以是不可路由的）（这个比方来自知乎）是不是有点动静结合的意思。

一开始mac地址都是烧在网卡中的，后来则是可以动态设置（虚拟的网卡mac地址就更可以改变了）。网卡的ip地址和mac地址可变也没什么关系，因为交换机和路由器可以学习所连接网络pc的ip、mac和swtich/router的port的对应关系。网卡可以设置为混杂模式，这样一个网卡可以接受所有收到的数据包（即便数据包的目的ip地址不是该网卡的ip地址）。

iproute2是一个套件，包含的是一套命令，类似于docker，所有的docker操作命令以docker开头，就可以完成关于docker的所有操作。具体的子操作，则类似于”docker network xx”。

既然网络是分层的，那么理论上讲不同的ip命令负责不同层的事情。比如`ip link` （Data Link layer，所以叫ip link）负责第二层，`ip address`负责第三层。所以，当你想设置mtu时（肯定是第二层的事），你得找`ip link`。

较高版本的linux内核支持namespace，因此ip命令还可以设置某个namespace的网卡（实际上，我们通常在root namespace执行ip命令，root namespace可以“看见”所有的子namespace）。

通过`man ip`我们可以看到

    ip link add link DEVICE [ name ] NAME
                   [ txqueuelen PACKETS ]
                   [ address LLADDR ] [ broadcast LLADDR ]
                   [ mtu MTU ]
                   type TYPE [ ARGS ]
           TYPE := [ vlan | veth | vcan | dummy | ifb | macvlan | can | bridge]
           
这说明，ip命令不仅可以添加网卡，还可以添加网桥等网络设备。

## brctl 

写的挺好，都不忍心翻译

Software defined networking (SDN) is the current wave sweeping the networking industry. And one of the key enablers of SDN is virtual networking. While SDN and virtual networking are in vogue these days, the support for virtual networking is not a recent development. And Linux bridge has been the pioneer in this regard.（简述SDN、virtual networking、Linux Bridge之间的关系）

Virtual networking requires the presence of a virtual switch inside a server/hypervisor. Even though it is called a bridge, the Linux bridge is really a virtual switch and used with KVM/QEMU hypervisor. Linux Bridge is a kernel module, first introduced in 2.2 kernel (circa 2000). And it is administered using brctl command on Linux.

以下来自`man btctl`

The command `brctl addbr <name>` creates a new instance of the ethernet bridge. **The network interface corresponding to the bridge will be called "name"**.

The command `brctl delbr <name>` deletes the instance "name" of the ethernet bridge. **The network interface corresponding to the bridge must be down** before it can be deleted!

Each bridge has a number of ports attached to it. Network traffic coming in on any of these ports will be forwarded to the other ports transparently, so that the bridge is  invisible to the rest of the network.

The command `brctl addif <brname> <ifname>` will make the interface "ifname" a port of the bridge "brname". This means that all frames received on "ifname" will be processed as if destined for the bridge. 

总的来说，就是使用brctl

1. 可以查看所有的linux bridge，增加和删除linux bridge
2. 针对一个linux bridge，可以将一个interface挂到bridge或移除，可以查看“挂到”上面的所有interface
3. 每建一个网桥，都会建一个跟网桥同名的interface，并挂在网桥上面。

## iptables

《网络是怎样连接的》：网络包的头部包含了用于控制通信操作的控制信息，经常用于设置包过滤规则的字段

|头部类型|规则判断条件|含义|
|---|---|---|
|MAC头部|发送方MAC地址|路由器在对包进行转发时会改写 MAC 地址，将转发目 标路由器的 MAC 地址设为接收方 MAC 地址，将自己 的 MAC 地址设为发送方 MAC 地址。通过发送方 MAC 地址，可以知道上一个转发路由器的 MAC 地址|
|IP头部|发送方 IP 地址|发送该包的原始设备的 IP 地址。如果要以发送设备来 设置规则，需要使用这个字段|
||接收方 IP 地址|包的目的地 IP 地址，如果要以包的目的地来设置规则， 需要使用这个字段|
||协议号|TCP/IP 协议为每个协议分配了一个编号，如果要以协 议类型来设置规则，需要使用这个编号。主要的协议号 包括 IP∶0;ICMP∶1;TCP∶6;UDP∶17;OSPF∶89|
|TCP 头部或 UDP 头部|发送方端口号|发送该包的程序对应的端口号。服务器程序对应的端口 号是固定的，因此根据服务器返回的包的端口号可以分 辨是哪个程序发送的。不过，客户端程序的端口号大多 是随机分配的，难以判断其来源，因此很少使用客户端 发送的包的端口号来设置过滤规则|
||接收方端口号|包的目的地程序对应的端口号。和发送方端口号一样， 一般使用服务器的端口号来设置规则，很少使用客户端 的端口号|
||TCP 控制位|TCP 协议的控制信息，主要用来控制连接操作<br>ACK 表示接收数据序号字段有效，一般用于通知发 送方数据已经正确接收<br>PSH 表示发送方应用程序希望不等待发送缓冲区填 充完毕，立即发送这个包<br>RST 强制断开连接，用于异常中断 <br>SYN 开始通信时连接操作中发送的第一个包中， SYN 为 1，ACK 为 0。如果能够过滤这样的包， 则后面的操作都无法继续，可以屏蔽整个访问<br>FIN 表示断开连接|
||分片|通过 IP 协议的分片功能拆分后的包，从第二个分片开 始会设置该字段|
|ICMP 消息 (非头部)的内容|ICMP 消息类型||

示例如下

|接收方IP地址|接收方端口号|发送方IP地址|发送方端口号|TCP控制位|允许/阻止|
|---|---|---|---|---|---|
|192.0.2.0/24|80|-|-|-|允许|
|-|-|192.0.2.0/24|80|SYN=1<br>ACK=0|阻止|
|-|-|192.0.2.0/24|80|-|允许|
|-|-|-|-|-|阻止|

### 从tomcat filter说起

[tomcat filter分析](https://zsr.github.io/2017/11/15/tomcat-filter%E5%88%86%E6%9E%90/)

![](/public/upload/network/tomcat_filter_chain.png)

每次请求过来都会创建一个过滤器链(filterChain)，并把待执行的servlet对象存放到过滤器链中。对于每个url，对应的filter个数都是不固定的，filterchain需要保存每个请求所对应的一个filter数组，以及调用到的filter的position，以便继续向下调用filter

### tomcat filter ==> netfilter 

netfilter 是linux内核的一个机制，用于在网络发送和转发的关键节点上加上 hook 函数。netfilter一个著名的实现，就是内核模块 ip_tables。在用户态，还有一个客户端程序 iptables，用命令行来干预内核的规则， iptables只是一个操作 Linux 内核 Netfilter 子系统的“界面”

![](/public/upload/network/linux_netfilter.png)

因为客户端发来的报文可能并不是本机而是其他服务器，当本机的内核支持IP_FORWARD时，我们可以将报文转发给其它服务器。**不考虑这种特殊情况，若将上图的forward chain 移除掉时，会清晰很多**。

![](/public/upload/network/netfilter_chain_flow.png)

1. 链 是指iptables 过滤的时机，类比到 java ee filter，就是你的逻辑是加在 `filterChain.doFilter(req, res);` 之前 还是之后。或者说 类似于 springmvc HandlerInterceptor的preHandle  和 postHandle 方法。
2. 表是负责 完成某个过滤 功能的模块

    1. filter 表，确定是否放行数据包
    2. nat表，**修改数据包的源ipport 和 目的ipport**

netfilter 在网络包收发及路由的 管道上，一共切了 5 个口，分别是 PREROUTING，FORWARD，POSTROUT- ING，INPUT 以及 OUTPUT;同时 netfilter 定义了包括 nat、filter 在内的若干个**网络包处理方式**。

![](/public/upload/network/iptables.png)

需要注意的是，routing 和 forwarding 很大程度上增加了以上 netfilter 的复杂 程度，如果我们不考虑 routing 和 forwarding，那么 netfilter 会变得跟一般过 滤器框架一样简单。

### 源码上的体现

    // 从tcp层向ip层发送数据包
    int __ip_local_out(struct net *net, struct sock *sk, struct sk_buff *skb){
        struct iphdr *iph = ip_hdr(skb);
        iph->tot_len = htons(skb->len);
        skb->protocol = htons(ETH_P_IP);
        // 可以看到第一个hook点NF_INET_LOCAL_OUT
        return nf_hook(NFPROTO_IPV4, NF_INET_LOCAL_OUT,
                net, sk, skb, NULL, skb_dst(skb)->dev,
                dst_output);
    }
    // 从ip层向link层发送数据包
    int ip_output(struct net *net, struct sock *sk, struct sk_buff *skb){
        struct net_device *dev = skb_dst(skb)->dev;
        skb->dev = dev;
        skb->protocol = htons(ETH_P_IP);
        // 可以看到第一个hook点NF_INET_POST_ROUTING
        return NF_HOOK_COND(NFPROTO_IPV4, NF_INET_POST_ROUTING,
                    net, sk, skb, NULL, dev,
                    ip_finish_output,
                    !(IPCB(skb)->flags & IPSKB_REROUTED));
    }
    // 从link到ip层的接收逻辑
    int ip_rcv(struct sk_buff *skb, struct net_device *dev, struct packet_type *pt, struct net_device *orig_dev){
        const struct iphdr *iph;
        struct net *net;
        u32 len;
        ......
        net = dev_net(dev);
        ......
        iph = ip_hdr(skb);
        len = ntohs(iph->tot_len);
        skb->transport_header = skb->network_header + iph->ihl*4;
        ......
        // 可以看到第一个hook点是NF_INET_PRE_ROUTING
        return NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING,
                net, NULL, skb, dev, NULL,
                ip_rcv_finish);
        ......
    }

### 操作

    iptbales -L -vn --line-number
    iptables -D INPUT 7
    iptables -D FORWARD 4

这样按序号删规则很方便

[iptables 概念](http://www.zsythink.net/archives/1199)

1. iptables 是 按rules 来办事的，这些规则分别指定了源地址、目的地址、传输协议等，并按数据包与规则是否匹配采取accept、reject、drop等action
2. rule 通常不只一个，所以多个rule 组成一个链，链分为自定义链和默认链
3. 根据链生效位置、以及host是否开始ip_forward的不同

	* 到本机某进程的报文 prerouting--> input
	* 开启转发功能时，由本机转发的报文 prerouting --> forward ==> postrouting
	* 由本机的某进程发出报文 output --> postrouting

4. 自定义链允许我们以自定义名称组织相关的规则，要被默认链引用，才可以生效

		Chain INPUT (policy ACCEPT)
		target     prot opt source               destination
		cali-INPUT  all  --  anywhere             anywhere             /* cali:Cz_u1IQiXIMmKD4c */

		Chain cali-INPUT (1 references)
		target     prot opt source               destination
		ACCEPT     all  --  anywhere             anywhere             /* cali:i7okJZpS8VxaJB3n */ mark match 0x1000000/0x1000000
		DROP       ipencap--  anywhere             anywhere             /* cali:p8Wwvr6qydjU36AQ */ /* Drop IPIP packets from non-Calico hosts */ ! match-set cali4-all-hosts src

## 引用

http://fishcried.com/2016-02-09/openvswitch-ops-guide/