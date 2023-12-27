---
title: CentOS7 配置网卡端口镜像
---

# 背景

最近一直在研究旁路监测，需要设置一个源端口镜像给两个目的端口（分别接两个监测设备），无奈ip-com交换机没配置明白，研究下使用软件实现暂时代替。

 

# 环境

发行版、内核、iptables版本信息如下

```
[root@ted ~]# uname -a
Linux ted 3.10.0-327.el7.x86_64 #1 SMP Thu Nov 19 22:10:57 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
[root@ted ~]# cat /etc/redhat-release 
CentOS Linux release 7.2.1511 (Core) 
[root@ted ~]# rpm -qi iptables
Name        : iptables
Version     : 1.4.21
Release     : 17.el7
Architecture: x86_64
```

 

# 配置

部署图大致如下

![](/images/[CentOS7 配置网卡端口镜像]/495966-20161228001347195-532046690.png)

办公网某主机IP: 192.168.118.1（演示时用的VMware的NAT模式，宿主机）

监测服务器A网卡eth0:192.168.118.134（演示时用的VMware的NAT模式，VM）

监测服务器A网卡eth1:192.168.12.13（演示时用的VMware的VMnet9，VM）

监测服务器B网卡eth0:192.168.12.12（演示时用的VMware的VMnet9，VM）

 

 

**在监测服务器A上执行如下命令**

查看路由设置

[![复制代码](img/[CentOS7 配置网卡端口镜像]/copycode.gif)](javascript:void(0);)

```
[root@ted ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.118.2   0.0.0.0         UG    100    0        0 eth0
192.168.12.0    0.0.0.0         255.255.255.0   U     0      0        0 eth1
192.168.118.0   0.0.0.0         255.255.255.0   U     100    0        0 eth0
```

[![复制代码](img/[CentOS7 配置网卡端口镜像]/copycode.gif)](javascript:void(0);)

所有192.168.12.0网段的流量都走eth1网卡（此网卡与服务器B的eth0直连）

 

配置只需一条命令

```
[root@ted ~]# iptables -I PREROUTING -t mangle -i eth0 -j TEE --gateway 192.168.12.12
[root@ted ~]# iptables-save
```

所有eth0的进口流量包都会被复制一份发到192.168.12.12上（从路由表上看出，先走eth1网卡，再到服务器B的eth0）

 

# 使用

在服务器A上用tcpdump抓包（开两个终端）

```
[root@ted ~]# tcpdump -i eth0 tcp port 80 -w A-eth0.pcap
[root@ted ~]# tcpdump -i eth0 tcp port 80 -w A-eth1.pcap
```

 

在服务器B上用tcpdump抓包

```
[root@min-base ~]# tcpdump -i eth0 tcp port 80 -w B-eth0.pcap
```

 

 在服务器A上开httpd服务器，用办公网主机访问该服务

![](/images/[CentOS7 配置网卡端口镜像]/495966-20161228002703398-1948062865.png)

 

# 结果

监测服务器A、B都得到了镜像流量。

这里为了演示，使用“办公机访问服务器A的httpd服务”代替“办公网镜像流量”，实际情况应该是服务器A网卡eth0是不配置IP的。

![](/images/[CentOS7 配置网卡端口镜像]/495966-20161228004526867-1564865189.png)

 

 

缺点

iptables-TEE实现端口镜像会改变源MAC、目的MAC

TODO

这里没有实现回包镜像，我再研究研究。。。

在服务器A上再执行一条iptables规则

```
[root@ted ~]# iptables -I POSTROUTING -t mangle -o eth0 -j TEE --gateway 192.168.12.12
```

**把从eth0出去的流量包都镜像一份发到192.168.12.12上就可以啦~~**