+++
date = "2016-10-29T22:31:40+08:00"
draft = false
title = "windows通过linux上网"
thumbnail = "images/linux/iptables.jpg"

categories = ["linux"]

+++

# 背景
女朋友笔记本的无线网卡突然不工作了，尝试有线还是好的，但是网线连接到路由器不够长。我本人台式机装的是archlinux，一块有线网卡，一块USB无线网卡。作为一个标准的四眼码农，关键时候怎能不好好表现一下。

# 通过共享linux有线网络实线windows上网
* archlinux台式机一台，双网卡，一块有线网卡（没用），一块USB无线网卡（正常上网）
* windows7笔记本一台，双网卡, 一块有线网卡（没用），一块无线网卡（已坏）
* 双绞线一根

# 基本配置
1. 用网线连接两块有线网卡
1. 配置ip转发`sysctl net.ipv4.ip_forward=1`
1. 配置nat伪装`iptables -t nat -A POSTROUTING -j MASQUERADE`
1. 分配网有线网卡在`192.168.1.0/24`网段，无线默认是在`192.168.0.0/24`网段
1. 分配linux有线网卡ip为`192.168.1.2/24`
1. 配置windows ip: `192.168.1.3`，mask: `255.255.255.0`, gateway: `192.168.1.2`
1. 配置windows dns为linux dns

# 观察网络
在linux上观察windows ping百度的网络数据：

    [root@arch ~]# tcpdump -i any -nn icmp
    tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
    listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
    23:00:03.298518 IP 192.168.1.3 > 115.239.210.27: ICMP echo request, id 1, seq 54, length 40
    23:00:03.298560 IP 192.168.0.114 > 115.239.210.27: ICMP echo request, id 1, seq 54, length 40
    23:00:03.307090 IP 115.239.210.27 > 192.168.0.114: ICMP echo reply, id 1, seq 54, length 40
    23:00:03.307120 IP 115.239.210.27 > 192.168.1.3: ICMP echo reply, id 1, seq 54, length 40

