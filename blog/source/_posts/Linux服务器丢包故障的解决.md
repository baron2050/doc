---
title: Linux服务器丢包故障的解决
---

我们使用Linux作为服务器操作系统时，为了达到高并发处理能力，充分利用机器性能，经常会进行一些内核参数的调整优化，但不合理的调整常常也会引起意想不到的其他问题，本文就一次Linux服务器丢包故障的处理过程，结合Linux内核参数说明和TCP/IP协议栈相关的理论，介绍一些常见的丢包故障定位方法和解决思路。

在开始之前，我们先用一张图解释 linux 系统接收网络报文的过程。

1. 首先网络报文通过物理网线发送到网卡
2. 网络驱动程序会把网络中的报文读出来放到 ring buffer 中，这个过程使用 DMA（Direct Memory Access），不需要 CPU 参与
3. 内核从 ring buffer 中读取报文进行处理，执行 IP 和 TCP/UDP 层的逻辑，最后把报文放到应用程序的 socket buffer 中
4. 应用程序从 socket buffer 中读取报文进行处理

![image1](/images/Linux服务器丢包故障的解决/GetImage.ashx)

在接收 UDP 报文的过程中，图中任何一个过程都可能会主动或者被动地把报文丢弃，因此丢包可能发生在网卡和驱动，也可能发生在系统和应用。

之所以没有分析发送数据流程，一是因为发送流程和接收类似，只是方向相反；另外发送流程报文丢失的概率比接收小，只有在应用程序发送的报文速率大于内核和网卡处理速率时才会发生。

本篇文章假定机器只有一个名字为 eth0 的 interface，如果有多个 interface 或者 interface 的名字不是 eth0，请按照实际情况进行分析。

NOTE：文中出现的 RX（receive） 表示接收报文，TX（transmit） 表示发送报文。

## 问题现象

本次故障的反馈现象是：从办公网访问公网服务器不稳定，服务器某些端口访问经常超时，但Ping测试显示客户端与服务器的链路始终是稳定低延迟的。

通过在服务器端抓包，发现还有几个特点：

- 从办公网访问服务器有多个客户端，是同一个出口IP，有少部分是始终能够稳定连接的，另一部分间歇访问超时或延迟很高
- 同一时刻的访问，无论哪个客户端的数据包先到达，服务端会及时处理部分客户端的SYN请求，对另一部分客户端的SYN包“视而不见”，如tcpdump数据所示，源端口为56909的SYN请求没有得到响应，同一时间源端口为50212的另一客户端SYN请求马上得到响应。

```
$ sudo tcpdump -i eth0 port 22 and "tcp[tcpflags] & (tcp-syn) != 0"18:56:37.404603 IP CLIENT.56909 > SERVER.22: Flags [S], seq 1190606850, win 29200, options [mss 1448,sackOK,TS val 198321481 ecr 0,nop,wscale 7], length 018:56:38.404582 IP CLIENT.56909 > SERVER.22: Flags [S], seq 1190606850, win 29200, options [mss 1448,sackOK,TS val 198321731 ecr 0,nop,wscale 7], length 018:56:40.407289 IP CLIENT.56909 > SERVER.22: Flags [S], seq 1190606850, win 29200, options [mss 1448,sackOK,TS val 198322232 ecr 0,nop,wscale 7], length 018:56:44.416108 IP CLIENT.56909 > SERVER.22: Flags [S], seq 1190606850, win 29200, options [mss 1448,sackOK,TS val 198323234 ecr 0,nop,wscale 7], length 018:56:45.100033 IP CLIENT.50212 > SERVER.22: Flags [S], seq 4207350463, win 65535, options [mss 1366,nop,wscale 5,nop,nop,TS val 821068631 ecr 0,sackOK,eol], length 018:56:45.100110 IP SERVER.22 > CLIENT.50212: Flags [S.], seq 1281140899, ack 4207350464, win 27960, options [mss 1410,sackOK,TS val 1709997543 ecr 821068631,nop,wscale 7], length 018:56:52.439086 IP CLIENT.56909 > SERVER.22: Flags [S], seq 1190606850, win 29200, options [mss 1448,sackOK,TS val 198325240 ecr 0,nop,wscale 7], length 018:57:08.472825 IP CLIENT.56909 > SERVER.22: Flags [S], seq 1190606850, win 29200, options [mss 1448,sackOK,TS val 198329248 ecr 0,nop,wscale 7], length 018:57:40.535621 IP CLIENT.56909 > SERVER.22: Flags [S], seq 1190606850, win 29200, options [mss 1448,sackOK,TS val 198337264 ecr 0,nop,wscale 7], length 018:57:40.535698 IP SERVER.22 > CLIENT.56909: Flags [S.], seq 3621462255, ack 1190606851, win 27960, options [mss 1410,sackOK,TS val 1710011402ecr 198337264,nop,wscale 7], length 0
```

如果能排除网卡或者驱动丢包可能的话，linux系统丢包的原因相对就很多，常见的有：UDP 报文错误、防火墙、UDP buffer size 不足、系统负载过高等，这里对这些丢包原因进行分析。

## 名词解释

```
# ifconfig em2em2       Link encap:Ethernet  HWaddr AC::3D:A9::0Dinet addr:211.211.211.211  Bcast:211.211.211.255  Mask:255.255.255.0UP BROADCAST RUNNING MULTICAST  MTU:  Metric:RX packets: errors: dropped: overruns: frame:TX packets: errors: dropped: overruns: carrier:collisions: txqueuelen:RX bytes: ( (1.3 TiB)Memory:94b00000-94b20000
```

- RX errors: 表示总的收包的错误数量，这包括 too-long-frames 错误，Ring Buffer 溢出错误，crc 校验错误，帧同步错误，fifo overruns 以及 missed pkg 等等。
- RX dropped: 表示数据包已经进入了 Ring Buffer，但是由于内存不够等系统原因，导致在拷贝到内存的过程中被丢弃。
- RX overruns: 表示了 fifo 的 overruns，这是由于 Ring Buffer(aka Driver Queue) 传输的 IO 大于 kernel 能够处理的 IO 导致的，而 Ring Buffer 则是指在发起 IRQ 请求之前的那块 buffer。很明显，overruns 的增大意味着数据包没到 Ring Buffer 就被网卡物理层给丢弃了，而 CPU 无法即使的处理中断是造成 Ring Buffer 满的原因之一，上面那台有问题的机器就是因为 interruprs 分布的不均匀(都压在 core0)，没有做 affinity 而造成的丢包。
- RX frame: 表示 misaligned 的 frames。

对于 TX 的来说，出现上述 counter 增大的原因主要包括 aborted transmission, errors due to carrirer, fifo error, heartbeat erros 以及 windown error，而 collisions 则表示由于 CSMA/CD 造成的传输中断。

dropped与overruns的区别 dropped，表示这个数据包已经进入到网卡的接收缓存fifo队列，并且开始被系统中断处理准备进行数据包拷贝（从网卡缓存fifo队列拷贝到系统内存），但由于此时的系统原因（比如内存不够等）导致这个数据包被丢掉，即这个数据包被Linux系统丢掉。 overruns，表示这个数据包还没有被进入到网卡的接收缓存fifo队列就被丢掉，因此此时网卡的fifo是满的。为什么fifo会是满的？因为系统繁忙，来不及响应网卡中断，导致网卡里的数据包没有及时的拷贝到系统内存，fifo是满的就导致后面的数据包进不来，即这个数据包被网卡硬件丢掉。所以，个人觉得遇到overruns非0，需要检测cpu负载与cpu中断情

## 排查过程

服务器能正常接收到数据包，问题可以限定在两种可能：部分客户端发出的数据包本身异常；服务器处理部分客户端的数据包时触发了某种机制丢弃了数据包。因为出问题的客户端能够正常访问公网上其他服务，后者的可能性更大。

有哪些情况会导致Linux服务器丢弃数据包？

### 确认有 UDP 丢包发生

要查看网卡是否有丢包，可以使用 ethtool -S eth0 查看，在输出中查找 bad 或者 drop 对应的字段是否有数据，在正常情况下，这些字段对应的数字应该都是 0。如果看到对应的数字在不断增长，就说明网卡有丢包。

另外一个查看网卡丢包数据的命令是 ifconfig，它的输出中会有 RX(receive 接收报文)和 TX（transmit 发送报文）的统计数据：

```
~# ifconfig eth0...RX packets 3553389376  bytes 2599862532475 (2.3 TiB)RX errors 0  dropped 1353  overruns 0  frame 0TX packets 3479495131  bytes 3205366800850 (2.9 TiB)TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0...
```

此外，linux 系统也提供了各个网络协议的丢包信息，可以使用 netstat -s 命令查看，加上 –udp 可以只看 UDP 相关的报文数据：

```
[root@holodesk02 GOD]# netstat -s -uIcmpMsg:InType0: 3InType3: 1719356InType8: 13InType11: 59OutType0: 13OutType3: 1737641OutType8: 10OutType11: 263Udp:517488890 packets received2487375 packets to unknown port received.47533568 packet receive errors147264581 packets sent12851135 receive buffer errors0 send buffer errorsUdpLite:IpExt:OutMcastPkts: 696InBcastPkts: 2373968InOctets: 4954097451540OutOctets: 5538322535160OutMcastOctets: 79632InBcastOctets: 934783053InNoECTPkts: 5584838675
```

对于上面的输出，关注下面的信息来查看 UDP 丢包的情况：

- packet receive errors 不为空，并且在一直增长说明系统有 UDP 丢包
- packets to unknown port received 表示系统接收到的 UDP 报文所在的目标端口没有应用在监听，一般是服务没有启动导致的，并不会造成严重的问题
- receive buffer errors 表示因为 UDP 的接收缓存太小导致丢包的数量

**NOTE： 并不是丢包数量不为零就有问题，对于 UDP 来说，如果有少量的丢包很可能是预期的行为，比如丢包率（丢包数量/接收报文数量）在万分之一甚至更低。**

### 网卡或者驱动丢包

之前讲过，如果 `ethtool -S eth0` 中有 `rx_***_errors` 那么很可能是网卡有问题，导致系统丢包，需要联系服务器或者网卡供应商进行处理。

```
# ethtool -S eth0 | grep rx_ | grep errorsrx_crc_errors: 0rx_missed_errors: 0rx_long_length_errors: 0rx_short_length_errors: 0rx_align_errors: 0rx_errors: 0rx_length_errors: 0rx_over_errors: 0rx_frame_errors: 0rx_fifo_errors: 0
```

`netstat -i` 也会提供每个网卡的接发报文以及丢包的情况，正常情况下输出中 error 或者 drop 应该为 0。

如果硬件或者驱动没有问题，一般网卡丢包是因为设置的缓存区（ring buffer）太小，可以使用 ethtool 命令查看和设置网卡的 ring buffer。

`ethtool -g` 可以查看某个网卡的 ring buffer，比如下面的例子

```
# ethtool -g eth0Ring parameters for eth0:Pre-set maximums:RX:4096RX Mini:0RX Jumbo:0TX:4096Current hardware settings:RX:256RX Mini:0RX Jumbo:0TX:256
```

Pre-set 表示网卡最大的 ring buffer 值，可以使用 `ethtool -G eth0 rx 8192` 设置它的值。

### UDP 报文错误

如果在传输过程中UDP 报文被修改，会导致 checksum 错误，或者长度错误，linux 在接收到 UDP 报文时会对此进行校验，一旦发明错误会把报文丢弃。

如果希望 UDP 报文 checksum 及时有错也要发送给应用程序，可以在通过 socket 参数禁用 UDP checksum 检查：

```
int disable = 1;setsockopt(sock_fd, SOL_SOCKET, SO_NO_CHECK, (void*)&disable, sizeof(disable)
```

### UDP buffer size 不足

linux 系统在接收报文之后，会把报文保存到缓存区中。因为缓存区的大小是有限的，如果出现 UDP 报文过大（超过缓存区大小或者 MTU 大小）、接收到报文的速率太快，都可能导致 linux 因为缓存满而直接丢包的情况。

在系统层面，linux 设置了 receive buffer 可以配置的最大值，可以在下面的文件中查看，一般是 linux 在启动的时候会根据内存大小设置一个初始值。

- /proc/sys/net/core/rmem_max：允许设置的 receive buffer 最大值
- /proc/sys/net/core/rmem_default：默认使用的 receive buffer 值
- /proc/sys/net/core/wmem_max：允许设置的 send buffer 最大值
- /proc/sys/net/core/wmem_dafault：默认使用的 send buffer 最大值

但是这些初始值并不是为了应对大流量的 UDP 报文，如果应用程序接收和发送 UDP 报文非常多，需要讲这个值调大。可以使用 sysctl 命令让它立即生效：

```
sysctl -w net.core.rmem_max=26214400 # 设置为 25M
```

也可以修改 `/etc/sysctl.conf` 中对应的参数在下次启动时让参数保持生效。

如果报文报文过大，可以在发送方对数据进行分割，保证每个报文的大小在 MTU 内。

另外一个可以配置的参数是 `netdev_max_backlog`，它表示 linux 内核从网卡驱动中读取报文后可以缓存的报文数量，默认是 1000，可以调大这个值，比如设置成 2000：

```
sudo sysctl -w net.core.netdev_max_backlog=2000
```

### 系统负载过高

系统 CPU、memory、IO 负载过高都有可能导致网络丢包，比如 CPU 如果负载过高，系统没有时间进行报文的 checksum 计算、复制内存等操作，从而导致网卡或者 socket buffer 出丢包；memory 负载过高，会应用程序处理过慢，无法及时处理报文；IO 负载过高，CPU 都用来响应 IO wait，没有时间处理缓存中的 UDP 报文。

linux 系统本身就是相互关联的系统，任何一个组件出现问题都有可能影响到其他组件的正常运行。对于系统负载过高，要么是应用程序有问题，要么是系统不足。对于前者需要及时发现，debug 和修复；对于后者，也要及时发现并扩容。

### 应用丢包

上面提到系统的 UDP buffer size，调节的 sysctl 参数只是系统允许的最大值，每个应用程序在创建 socket 时需要设置自己 socket buffer size 的值。

linux 系统会把接受到的报文放到 socket 的 buffer 中，应用程序从 buffer 中不断地读取报文。所以这里有两个和应用有关的因素会影响是否会丢包：socket buffer size 大小以及应用程序读取报文的速度。

对于第一个问题，可以在应用程序初始化 socket 的时候设置 socket receive buffer 的大小，比如下面的代码把 socket buffer 设置为 20MB：

```
uint64_t receive_buf_size = 20*1024*1024;  //20 MBsetsockopt(socket_fd, SOL_SOCKET, SO_RCVBUF, &receive_buf_size, sizeof(receive_buf_size));
```

如果不是自己编写和维护的程序，修改应用代码是件不好甚至不太可能的事情。很多应用程序会提供配置参数来调节这个值，请参考对应的官方文档；如果没有可用的配置参数，只能给程序的开发者提 issue 了。

很明显，增加应用的 receive buffer 会减少丢包的可能性，但同时会导致应用使用更多的内存，所以需要谨慎使用。

另外一个因素是应用读取 buffer 中报文的速度，对于应用程序来说，处理报文应该采取异步的方式

### 包丢在什么地方

想要详细了解 linux 系统在执行哪个函数时丢包的话，可以使用 dropwatch 工具，它监听系统丢包信息，并打印出丢包发生的函数地址：

```
# dropwatch -l kasInitalizing kallsyms dbdropwatch> startEnabling monitoring...Kernel monitoring activated.Issue Ctrl-C to stop monitoring1 drops at tcp_v4_do_rcv+cd (0xffffffff81799bad)10 drops at tcp_v4_rcv+80 (0xffffffff8179a620)1 drops at sk_stream_kill_queues+57 (0xffffffff81729ca7)4 drops at unix_release_sock+20e (0xffffffff817dc94e)1 drops at igmp_rcv+e1 (0xffffffff817b4c41)1 drops at igmp_rcv+e1 (0xffffffff817b4c41)
```

通过这些信息，找到对应的内核代码处，就能知道内核在哪个步骤中把报文丢弃，以及大致的丢包原因。

此外，还可以使用 linux perf 工具监听 kfree_skb（把网络报文丢弃时会调用该函数） 事件的发生：

```
sudo perf record -g -a -e skb:kfree_skbsudo perf script
```

关于 perf 命令的使用和解读，网上有很多文章可以参考。

### 关于UDP丢包的总结

- UDP 本身就是无连接不可靠的协议，适用于报文偶尔丢失也不影响程序状态的场景，比如视频、音频、游戏、监控等。对报文可靠性要求比较高的应用不要使用 UDP，推荐直接使用 TCP。当然，也可以在应用层做重试、去重保证可靠性
- 如果发现服务器丢包，首先通过监控查看系统负载是否过高，先想办法把负载降低再看丢包问题是否消失
- 如果系统负载过高，UDP 丢包是没有有效解决方案的。如果是应用异常导致 CPU、memory、IO 过高，请及时定位异常应用并修复；如果是资源不够，监控应该能及时发现并快速扩容
- 对于系统大量接收或者发送 UDP 报文的，可以通过调节系统和程序的 socket buffer size 来降低丢包的概率
- 应用程序在处理 UDP 报文时，要采用异步方式，在两次接收报文之间不要有太多的处理逻辑

### 防火墙拦截

服务器端口无法连接，通常就是查看防火墙配置了，虽然这里已经确认同一个出口IP的客户端有的能够正常访问，但也不排除配置了DROP特定端口范围的可能性。

如果系统防火墙丢包，表现的行为一般是所有的 UDP 报文都无法正常接收，当然不排除防火墙只 drop 一部分报文的可能性。

如果遇到丢包比率非常大的情况，请先检查防火墙规则，保证防火墙没有主动 drop UDP 报文。

**如何确认**

查看iptables filter表，确认是否有相应规则会导致此丢包行为：

```
$ sudo iptables-save -t filter
```

这里容易排除防火墙拦截的可能性。

### 连接跟踪表溢出

除了防火墙本身配置DROP规则外，与防火墙有关的还有连接跟踪表nf_conntrack，Linux为每个经过内核网络栈的数据包，生成一个新的连接记录项，当服务器处理的连接过多时，连接跟踪表被打满，服务器会丢弃新建连接的数据包。

**如何确认**

通过dmesg可以确认是否有该情况发生：

```
$ dmesg |grep nf_conntrack
```

如果输出值中有“nf_conntrack: table full, dropping packet”，说明服务器nf_conntrack表已经被打满。

通过/proc文件系统查看nf_conntrack表实时状态：

```
# 查看nf_conntrack表最大连接数$ cat /proc/sys/net/netfilter/nf_conntrack_max65536# 查看nf_conntrack表当前连接数$ cat /proc/sys/net/netfilter/nf_conntrack_count7611
```

当前连接数远没有达到跟踪表最大值，排除这个因素。

**如何解决**

如果确认服务器因连接跟踪表溢出而开始丢包，首先需要查看具体连接判断是否正遭受DOS攻击，如果是正常的业务流量造成，可以考虑调整nf_conntrack的参数：

`nf_conntrack_max`决定连接跟踪表的大小，默认值是65535，可以根据系统内存大小计算一个合理值：`CONNTRACK_MAX = RAMSIZE(in bytes)/16384/(ARCH/32)`，如32G内存可以设置1048576；

`nf_conntrack_buckets`决定存储`conntrack`条目的哈希表大小，默认值是`nf_conntrack_max`的1/4，延续这种计算方式：`BUCKETS = CONNTRACK_MAX/4`，如32G内存可以设置262144；

`nf_conntrack_tcp_timeout_established`决定ESTABLISHED状态连接的超时时间，默认值是5天，可以缩短到1小时，即3600。

```
$ sysctl -w net.netfilter.nf_conntrack_max=1048576$ sysctl -w net.netfilter.nf_conntrack_buckets=262144$ sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=3600
```

### Ring Buffer溢出

排除了防火墙的因素，我们从底向上来看Linux接收数据包的处理过程，首先是网卡驱动层。

如下图所示，物理介质上的数据帧到达后首先由NIC（网络适配器）读取，写入设备内部缓冲区Ring Buffer中，再由中断处理程序触发Softirq从中消费，Ring Buffer的大小因网卡设备而异。当网络数据包到达（生产）的速率快于内核处理（消费）的速率时，Ring Buffer很快会被填满，新来的数据包将被丢弃。

![image2](/images/Linux服务器丢包故障的解决/GetImage.ashx)

**如何确认**

通过ethtool或/proc/net/dev可以查看因Ring Buffer满而丢弃的包统计，在统计项中以fifo标识：

```
$ ethtool -S eth0|grep rx_fiforx_fifo_errors: 0$ cat /proc/net/devInter-|   Receive                                                |  Transmitface |bytes    packets errs drop fifo frame compressed multicast|bytes    packets errs drop fifo colls carrier compressedeth0: 17253386680731 42839525880    0    0    0     0          0 244182022 14879545018057 41657801805    0    0    0     0       0         0
```

可以看到服务器的接收方向的fifo丢包数并没有增加，这里自然也排除这个原因。

**如何解决**

如果发现服务器上某个网卡的fifo数持续增大，可以去确认CPU中断是否分配均匀，也可以尝试增加Ring Buffer的大小，通过ethtool可以查看网卡设备Ring Buffer最大值，修改Ring Buffer当前设置：

```
# 查看eth0网卡Ring Buffer最大值和当前设置$ ethtool -g eth0Ring parameters for eth0:Pre-set maximums:RX:     4096RX Mini:    0RX Jumbo:   0TX:     4096Current hardware settings:RX:     1024RX Mini:    0RX Jumbo:   0TX:     1024# 修改网卡eth0接收与发送硬件缓存区大小$ ethtool -G eth0 rx 4096 tx 4096Pre-set maximums:RX:     4096RX Mini:    0RX Jumbo:   0TX:     4096Current hardware settings:RX:     4096RX Mini:    0RX Jumbo:   0TX:     4096
```

### netdev_max_backlog溢出

netdev_max_backlog是内核从NIC收到包后，交由协议栈（如IP、TCP）处理之前的缓冲队列。每个CPU核都有一个backlog队列，与Ring Buffer同理，当接收包的速率大于内核协议栈处理的速率时，CPU的backlog队列不断增长，当达到设定的netdev_max_backlog值时，数据包将被丢弃。

**如何确认**

通过查看/proc/net/softnet_stat可以确定是否发生了netdev backlog队列溢出：

```
$ cat /proc/net/softnet_stat01a7b464 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 0000000001d4d71f 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 000000000349e798 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000017e0826 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
```

其中： 每一行代表每个CPU核的状态统计，从CPU0依次往下； 每一列代表一个CPU核的各项统计：第一列代表中断处理程序收到的包总数；第二列即代表由于netdev_max_backlog队列溢出而被丢弃的包总数。 从上面的输出可以看出，这台服务器统计中，并没有因为netdev_max_backlog导致的丢包。

**如何解决**

netdev_max_backlog的默认值是1000，在高速链路上，可能会出现上述第二列统计不为0的情况，可以通过修改内核参数net.core.netdev_max_backlog来解决：

```
$ sysctl -w net.core.netdev_max_backlog=2000
```

### 反向路由过滤

反向路由过滤机制是Linux通过反向路由查询，检查收到的数据包源IP是否可路由（Loose mode）、是否最佳路由（Strict mode），如果没有通过验证，则丢弃数据包，设计的目的是防范IP地址欺骗攻击。rp_filter提供了

**三种模式供配置：**

- 0 - 不验证
- 1 - RFC3704定义的严格模式：对每个收到的数据包，查询反向路由，如果数据包入口和反向路由出口不一致，则不通过
- 2 - RFC3704定义的松散模式：对每个收到的数据包，查询反向路由，如果任何接口都不可达，则不通过

**如何确认**

查看当前rp_filter策略配置：

```
$ cat /proc/sys/net/ipv4/conf/eth0/rp_filter
```

如果这里设置为1，就需要查看主机的网络环境和路由策略是否可能会导致客户端的入包无法通过反向路由验证了。

从原理来看这个机制工作在网络层，因此，如果客户端能够Ping通服务器，就能够排除这个因素了。

**如何解决**

根据实际网络环境将rp_filter设置为0或2：

```
$ sysctl -w net.ipv4.conf.all.rp_filter=2
```

或

```
$ sysctl -w net.ipv4.conf.eth0.rp_filter=2
```

### 半连接队列溢出

半连接队列指的是TCP传输中服务器收到SYN包但还未完成三次握手的连接队列，队列大小由内核参数tcp_max_syn_backlog定义。

当服务器保持的半连接数量达到tcp_max_syn_backlog后，内核将会丢弃新来的SYN包。

**如何确认**

通过dmesg可以确认是否有该情况发生：

```
$ dmesg | grep "TCP: drop open request from"
```

半连接队列的连接数量可以通过netstat统计SYN_RECV状态的连接得知

```
$ netstat -ant|grep SYN_RECV|wc -l0
```

大多数情况下这个值应该是0或很小，因为半连接状态从第一次握手完成时进入，第三次握手完成后退出，正常的网络环境中这个过程发生很快，如果这个值较大，服务器极有可能受到了SYN Flood攻击。

**如何解决**

tcp_max_syn_backlog的默认值是256，通常推荐内存大于128MB的服务器可以将该值调高至1024，内存小于32MB的服务器调低到128，同样，该参数通过sysctl修改：

```
$ sysctl -w net.ipv4.tcp_max_syn_backlog=1024
```

另外，上述行为受到内核参数tcp_syncookies的影响，若启用syncookie机制，当半连接队列溢出时，并不会直接丢弃SYN包，而是回复带有syncookie的SYC+ACK包，设计的目的是防范SYN Flood造成正常请求服务不可用。

```
$ sysctl -w net.ipv4.tcp_syncookies=1net.ipv4.tcp_syncookies = 1
```

### PAWS

PAWS全名Protect Againest Wrapped Sequence numbers，目的是解决在高带宽下，TCP序列号在一次会话中可能被重复使用而带来的问题。

![image2](/images/Linux服务器丢包故障的解决/GetImage.ashx)

如上图所示，客户端发送的序列号为A的数据包A1因某些原因在网络中“迷路”，在一定时间没有到达服务端，客户端超时重传序列号为A的数据包A2，接下来假设带宽足够，传输用尽序列号空间，重新使用A，此时服务端等待的是序列号为A的数据包A3，而恰巧此时前面“迷路”的A1到达服务端，如果服务端仅靠序列号A就判断数据包合法，就会将错误的数据传递到用户态程序，造成程序异常。

PAWS要解决的就是上述问题，它依赖于timestamp机制，理论依据是：在一条正常的TCP流中，按序接收到的所有TCP数据包中的timestamp都应该是单调非递减的，这样就能判断那些timestamp小于当前TCP流已处理的最大timestamp值的报文是延迟到达的重复报文，可以予以丢弃。在上文的例子中，服务器已经处理数据包Z，而后到来的A1包的timestamp必然小于Z包的timestamp，因此服务端会丢弃迟到的A1包，等待正确的报文到来。

PAWS机制的实现关键是内核保存了Per-Connection的最近接收时间戳，如果加以改进，就可以用来优化服务器TIME_WAIT状态的快速回收。

TIME_WAIT状态是TCP四次挥手中主动关闭连接的一方需要进入的最后一个状态，并且通常需要在该状态保持2*MSL（报文最大生存时间），它存在的意义有两个：

1.可靠地实现TCP全双工连接的关闭：关闭连接的四次挥手过程中，最终的ACK由主动关闭连接的一方（称为A）发出，如果这个ACK丢失，对端（称为B）将重发FIN，如果A不维持连接的TIME_WAIT状态，而是直接进入CLOSED，则无法重传ACK，B端的连接因此不能及时可靠释放。

2.等待“迷路”的重复数据包在网络中因生存时间到期消失：通信双方A与B，A的数据包因“迷路”没有及时到达B，A会重发数据包，当A与B完成传输并断开连接后，如果A不维持TIME_WAIT状态2*MSL时间，便有可能与B再次建立相同源端口和目的端口的“新连接”，而前一次连接中“迷路”的报文有可能在这时到达，并被B接收处理，造成异常，维持2*MSL的目的就是等待前一次连接的数据包在网络中消失。

TIME_WAIT状态的连接需要占用服务器内存资源维持，Linux内核提供了一个参数来控制TIME_WAIT状态的快速回收：tcp_tw_recycle，它的理论依据是：

在PAWS的理论基础上，如果内核保存Per-Host的最近接收时间戳，接收数据包时进行时间戳比对，就能避免TIME_WAIT意图解决的第二个问题：前一个连接的数据包在新连接中被当做有效数据包处理的情况。这样就没有必要维持TIME_WAIT状态2*MSL的时间来等待数据包消失，仅需要等待足够的RTO（超时重传），解决ACK丢失需要重传的情况，来达到快速回收TIME_WAIT状态连接的目的。

但上述理论在多个客户端使用NAT访问服务器时会产生新的问题：同一个NAT背后的多个客户端时间戳是很难保持一致的（timestamp机制使用的是系统启动相对时间），对于服务器来说，两台客户端主机各自建立的TCP连接表现为同一个对端IP的两个连接，按照Per-Host记录的最近接收时间戳会更新为两台客户端主机中时间戳较大的那个，而时间戳相对较小的客户端发出的所有数据包对服务器来说都是这台主机已过期的重复数据，因此会直接丢弃。

**如何确认**

通过netstat可以得到因PAWS机制timestamp验证被丢弃的数据包统计：

```
$ netstat -s |grep -e "passive connections rejected because of time stamp" -e "packets rejects in established connections because of timestamp”387158 passive connections rejected because of time stamp825313 packets rejects in established connections because of timestamp
```

通过sysctl查看是否启用了tcp_tw_recycle及tcp_timestamp:

```
$ sysctl net.ipv4.tcp_tw_recyclenet.ipv4.tcp_tw_recycle = 1$ sysctl net.ipv4.tcp_timestampsnet.ipv4.tcp_timestamps = 1
```

这次问题正是因为服务器同时开启了tcp_tw_recycle和timestamps，而客户端正是使用NAT来访问服务器，造成启动时间相对较短的客户端得不到服务器的正常响应。

**如何解决**

如果服务器作为服务端提供服务，且明确客户端会通过NAT网络访问，或服务器之前有7层转发设备会替换客户端源IP时，是不应该开启tcp_tw_recycle的，而timestamps除了支持tcp_tw_recycle外还被其他机制依赖，推荐继续开启：

```
$ sysctl -w net.ipv4.tcp_tw_recycle=0$ sysctl -w net.ipv4.tcp_timestamps=1
```

### 怎么知道为什么数据包被丢弃

dropwatch

通过谷歌搜索，发现一个很酷的工具叫 dropwatch 。 没有现成的 Ubuntu 安装软件包，但可以通过 github 下载：

https://github.com/pavel-odintsov/drop_watch

以下是我可以编译的说明：

```
sudo apt-get install -y libnl-3-dev libnl-genl-3-dev binutils-dev libreadline6-devgit clone https://github.com/pavel-odintsov/drop_watch.gitcd drop_watch/srcmake
```

这里是输出！ 它告诉我哪个内核函数丢失数据包，酷！

```
sudo ./dropwatch -l kasInitalizing kallsyms dbdropwatch> startEnabling monitoring...Kernel monitoring activated.Issue Ctrl-C to stop monitoring1 drops at tcp_v4_do_rcv+cd (0xffffffff81799bad)10 drops at tcp_v4_rcv+80 (0xffffffff8179a620)1 drops at sk_stream_kill_queues+57 (0xffffffff81729ca7)4 drops at unix_release_sock+20e (0xffffffff817dc94e)1 drops at igmp_rcv+e1 (0xffffffff817b4c41)1 drops at igmp_rcv+e1 (0xffffffff817b4c41)
```

perf

用perf监控丢弃的数据包

还有另一个很酷的方法，用来调试发生什么。

可以使用 perf 监视 kfree_skb 事件，这将告诉你什么时候丢弃数据包（内核堆栈发生的地方）：

```
sudo perf record -g -a -e skb:kfree_skbsudo perf script
```

### 扩展阅读

还有这两个很酷的文章：

监控和调优Linux网络堆栈：接收数据

https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/

监控和调优Linux网络堆栈：发送数据

https://blog.packagecloud.io/eng/2017/02/06/monitoring-tuning-linux-networking-stack-sending-data/

### 结论

Linux提供了丰富的内核参数供使用者调整，调整得当可以大幅提高服务器的处理能力，但如果调整不当，就会引进莫名其妙的各种问题，比如这次开启tcp_tw_recycle导致丢包，实际也是为了减少TIME_WAIT连接数量而进行参数调优的结果。我们在做系统优化时，时刻要保持辩证和空杯的心态，不盲目吸收他人的果，而多去追求因，只有知其所以然，才能结合实际业务特点，得出最合理的优化配置。