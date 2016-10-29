+++
date = "2016-10-29T22:31:40+08:00"
draft = false
title = "记windows通过linux共享上网"
thumbnail = "images/linux/iptables.jpg"

categories = ["linux"]
tags = ["iptables"]

+++

# 背景
女友家用笔记本的无线网卡突然罢工了，尝试有线还是能用的。但是网线连接到路由器不够长。我本人台式机装的是archlinux，一块有线网卡，一块USB无线网卡，平时使用USB无线网卡上网。两电脑距离较近，使用网线完全合适，于是想到能不能让windows通过linux上网。作为一个爱折腾的码农，关键时候怎能不好好表现一下，说干就干。

# 环境
* windows7笔记本一台，双网卡, 一块有线网卡（没用），一块无线网卡（已坏）
* archlinux台式机一台，双网卡，一块有线网卡（没用），一块USB无线网卡（正常上网）
* 普通网线（双绞线）一根

目的：通过网线连接两块有线网卡，使windows通过linux上网

# 配置步骤
1. 用网线连接两块有线网卡
1. 开启ip转发`sysctl net.ipv4.ip_forward=1`
1. 开启nat伪装`iptables -t nat -A POSTROUTING -j MASQUERADE`
1. 配置linux有线网卡静态IP: `192.168.1.2/24`
1. 配置windows有线网卡静态IP: `192.168.1.3`，MASK: `255.255.255.0`, GATEWAY: `192.168.1.2`
1. 配置windows的DNS为linux机器上的DNS或者google公共DNS: 8.8.8.8

# 观察网络
在linux上观察windows ping百度的网络数据，其中192.168.0.114是linux上USB无线网卡的IP地址：

    [root@arch ~]# tcpdump -i any -nn icmp
    tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
    listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
    23:00:03.298518 IP 192.168.1.3 > 115.239.210.27: ICMP echo request, id 1, seq 54, length 40
    23:00:03.298560 IP 192.168.0.114 > 115.239.210.27: ICMP echo request, id 1, seq 54, length 40
    23:00:03.307090 IP 115.239.210.27 > 192.168.0.114: ICMP echo reply, id 1, seq 54, length 40
    23:00:03.307120 IP 115.239.210.27 > 192.168.1.3: ICMP echo reply, id 1, seq 54, length 40

分析:

1. 由于windows网关为192.168.1.2，windows将数据包发送到linux有线网卡
1. 由于linux开启了ip转发，linux将数据包通过无线网卡发送到路由并启用nat伪装
1. 接着路由将回应包发送到linux无线网卡
1. 此时linux知道该包是自己nat伪装的，将数据包通过有线网卡转发到windows

# 总结
* windows与linux通过网线组成的局域网不能和无线网处于同一网段
* 必须配置windows的网关为linux上有线网卡的IP地址

