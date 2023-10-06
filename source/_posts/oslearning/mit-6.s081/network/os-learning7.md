---
title: Network
date: 2023-10-05 14:56:42
tags: 操作系统，计算机基础, C
categories: 操作系统
---

## 网络协议

Browser => LAN => Http server(本地通信由以太网负责)

但是这不能建立一个简单的网络，如果希望建立一个庞大的网络，需要网络层协议：

Broswer => LAN1 => Router => LAN2 => http server(远程通信由IP层负责)

### 以太网协议

```
#define ETHADDR_LEN 6
struct eth {
	uint8 dhost [ETHADDR_LEN];
	uint8 shost [ETHADDR_LEN];
	uint16 type // 它的上层协议类型
} __attributr__((packed));

#define ETHTYPE_IP 0x0800 // ip
#define ETHTYPE_ARP 0x0806 // arp


#[payload][T16*8bit][S48bit][D48bit], 前24bit是制造商编号，后24bit是网卡编号]#
#表示包的开始和结束，#是给硬件识别的标志


ARP, Request who-has 10.0.2.15 tell 10.0.0.2, length 28
0x0000: ffff ffff ffff(48bit, 这里表示广播地址) 5255 0a00 0202(src host mac) 0806(16 bit type) 0001(payload)
0x0010: 0800 0604 0001(这里0001~这里，共64 bytes是hrd + pro + hln + pln + op) 5255 0a00 0202(sender ethnet addr) 0a00 0202(sip, 10.0.2.2)
0x0020: 0000 0000 0000(target ethnet addr, unknown when request) 0a00 020f(tip, 10.0.2.15)
```



### ARP协议

```


ARP, Reply 10.0.2.15 is-at 52:44:00:12:34:56, length 28
0x0000: ffff ffff ffff(48bit, 这里表示广播地址) 5255 0a00 0202(src host mac) 0806(16 bit type) 0001(payload)   .....RU.....(.表示没有对应的ascii)
0x0010: 0800 0604 0002 5254 0a00 0202 0a00 0202
0x0020: 5255 0a00 0202 0a00 0202


struct arp {
	uint16 hrd
	uint16 pro
	uint8 hln
	uint8 pln
	uint16 op
	
	char sha[ETHADDR_LEN]
	uint32 sip
	char tha[ETHADDR_LEN]
	uint132 tip
} __attribute__((packed))

#define ARP_HRD_ETHER 1 // ethernet
enum {
	ARP_OP_REQUEST = 1;
	ARP_OP_REPLY = 2;
}

#[arp][T16*8bit][S48bit][D48bit] 以太网的有效载荷就是arp数据包，紧跟在以太网报头之后



[[[[[DNS]UDP] protol IP type ETH]]
发送上往下构建包，type和protocol告知下面该用什么协议去检查和理解payload
```

<img src="C:\Users\Mmjiang\Desktop\oslearning\无标题-2023-08-26-1164.png" alt="无标题-2023-08-26-1164" style="zoom:50%;" />

### IP协议

```
// an IP packet header (comes after an Ethernet)
struct ip {
	uint8 ip_vhl; // version << 5 | header length >> 2
	uint8 ip_tos; // type of service
	uint16 ip_len; // total length
	uint16 ip_id; // identification
	uint16 ip_off; // fragment offset field
	uint8 ip_ttl; // time to live
	uint8 ip_p; // protocol,上层协议是什么
	uint8 ip_sum; // checksum
	uint8 ip_src, ip_dst;
}

#define IPPROTO_TCP 6
#define IPPROTO_UDP 17

ffff ffff ffff 5254 0012 3456 0800 [4500
002f 0000 0000 6411(11是10进制的17表示UDP) 4eae(checksum) 0a00(10.0.2.15) 020f 0a00(10.0.2.2)
0202] 07d0 6403 001b 0000 6120 6d55 7373

```



### UDP协议

```
// udp packet header (comes after an tp header)

struct udp {
	uint16 sport; // src port
	uint16 dport; // dst port
	uint16 ulen; // length, including udp header, not including ip header
	uint16 sum // checksum
}

syscall socket
sockets(53)


ffff ffff ffff 5254 0012 3456 0800 4500
002f 0000 0000 6411 4eae 0a00 020f 0a00
0202 [07d0(src) 6403(dst) 001b(len) 0000(checksum) 6120(ascii) 6d55(ascii) 7373(ascii)... // a.message.from.xv6
```



`每个协议都有最大的数据长度，这是为什么？`

- 数据在沿着线路传播的时候，数据量越大，损失的几率越高，噪音越大，对于合理大小的校验和应该是16或者32位，它限制了包的大小。


- 另外，数据越大，需要的路由器缓存也就越大，需要高质量的硬件支持



<img src="C:\Users\Mmjiang\Desktop\oslearning\无标题-2023-08-26-1165.png" alt="无标题-2023-08-26-1165" style="zoom:50%;" />

## 网络协议栈

<img src="C:\Users\Mmjiang\Desktop\oslearning\无标题-2023-08-26-1163.png" alt="无标题-2023-08-26-1163" style="zoom:50%;" />



## 网络协议硬件中的协作模式

在单核处理器上，中断拥有最高的优先级执行度，所以当中断发生时，其他进程被暂时pending；但是在多核处理器上，将有更多的并行性，其他处理器上的进程不会被打扰，甚至可以读取网络线程处理好的数据缓存。另外，这网络协议栈中，去恶劣很常见，是w为了应对网络的临时性突变，比如网络线程处理很慢，但是这个时候有大量数据发送过来。

Q：同意网卡可以同时接收和发送吗？

A：可以，如果发送和接收的是同一个网卡的情况下，比如回环地址

<img src="C:\Users\Mmjiang\Desktop\oslearning\无标题-2023-08-26-1162.png" alt="无标题-2023-08-26-1162" style="zoom:50%;" />



经典网卡模式：数据到了，存放到网卡的缓存中，同时网卡驱动从NIC buffer拷贝数据到主机内存，然后触发中断。

缺点：由于NIC和内存的通过控制总线交互耗时比较长，导致处理比较慢。

<img src="C:\Users\Mmjiang\Desktop\oslearning\无标题-2023-08-26-1161.png" style="zoom:50%;" />



现代大部分处理器(E1000)：初始化的时候，网卡就知道了DMA ring(DMA是主机内存的概念，而非网卡内的，它是计算机内部用于存放网络数据包的一段特定内存，是一个指针数组)的地址，后续有数据写入或者读取都是和DMA ring交互的，没读写一次，记住下一个DMA ring的位置

<img src="C:\Users\Mmjiang\Desktop\oslearning\无标题-2023-08-26-1160.png" style="zoom:50%;" />





<img src="C:\Users\Mmjiang\Desktop\oslearning\无标题-2023-08-26-1159.png" alt="无标题-2023-08-26-1159" style="zoom:50%;" />





简单画图描述了下课堂里提到的相关论文的一幅图，横坐标是网卡输入速率，纵坐标是cpu处理后由网卡转发的速率，可以观察no pulling前面半部分，输入速率和输出速率持平，可是到了5000之后就是下降趋势了 (这里输出最大是5000packets/s，也就是网络线程最大处理速度是200 packets/ms)，这是为什么？

考虑单核情况下，会有两个任务

- 接收网卡数据传入带来的中断
- 处理数据的处理和转发的网络线程

一个是网卡接收到数据后，触发中断程序，同时cpu也在处理之前的输入，所以每次有新的输入进来的时候，都会触发中断，而拼盘的中断也是有代价的，这样就导致了转发任务一直主语饥饿状态，这就是课堂里所说的live lock

`如何解决?`

第一个中断到达的时候，运行interrupt routine，但是interrupt routine不从网卡上赋值数据包，它唤醒网络线程，同时设置网卡的中断为禁用状态，伪代码如下：

```
Loop
 pull a few packets(eg: 5)
 process
 if none
 	enable INTR
 	sleep
```

使用了这种不断轮询的策略后，如上图所示，不会出现网络线程饥饿的现象，但是如果网卡速率过大的话，还是会发生丢包的。



## Homework

e1000_transmit

`tx_ring` 中的每个元素长这样：

![img](https://pic2.zhimg.com/80/v2-4ad1343146b023e24d6c407ebf348d35_1440w.jpg)

```
int
e1000_transmit(struct mbuf *m)
{
  //
  // Your code here.
  //
  // the mbuf contains an ethernet frame; program it into
  // the TX descriptor ring so that the e1000 sends it. Stash
  // a pointer so that it can be freed after sending.
  //
  // ask the E1000 for the TX ring index at which 
  // it's expecting the next packet, by reading the E1000_TDT control register.
  acquire(&e1000_lock);
  uint32 index = regs[E1000_TDT];
  // the E1000 hasn't finished the corresponding previous transmission request, so return an error.
  if (!(tx_ring[index].status & E1000_TXD_STAT_DD)) {
    release(&e1000_lock);
    return -1;
  }
  // free the last mbuf that was transmitted from that descriptor (if there was one).
  if (tx_mbufs[index]) {
    mbuffree(tx_mbufs[index]);
  }
  // stash away a pointer to the mbuf for later freeing.
  tx_mbufs[index] = m;
  memset(&tx_ring[index],0,sizeof(tx_ring[index]));
  // fill in the descriptor. m->head points to the packet's content in memory, and m->len is the packet length.
  tx_ring[index].addr = (uint64)m->head;
  tx_ring[index].length = m->len;
  // set the necessary cmd flags.
  tx_ring[index].cmd = E1000_TXD_CMD_RS | E1000_TXD_CMD_EOP;
  // update the ring position by adding one to E1000_TDT modulo TX_RING_SIZE.
  regs[E1000_TDT] = (index + 1) % TX_RING_SIZE;
  release(&e1000_lock);
  return 0;
}
```

[MIT 6.S081 2020 LAB11记录 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/351563871)