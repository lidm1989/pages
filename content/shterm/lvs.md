+++
date = "2016-10-16T17:16:41+08:00"
draft = false
title = "lvs"

categories = ["linux"]

+++

# 背景
提高业务系统的处理能力及可靠性

* 集群：方案采用多节点集群
* 负载均衡：调度器
* 高可用：HA

# 方案
pacemaker + lvs + ldirectord

由于项目的主要负载是rdp及ssh协议，所以lb采用lvs。同时使用pacemaker保证lvs的可靠性。

集群由lvs进行负载均衡高度，使用pacemaker来保证lvs的高可用。
为了节约成本，lvs由pacemaker随机选取集群节点中的一个节点来运行。调度策略采用lvs的dr模式，调度算法采用lc。

# 实施
## lvs资源
## ldirectord

## lvs要求real server必须禁止arp响应而virtual server必须支持arp响应，pacemaker定义资源运行在Master/Slave模式并与ldirectord绑定关系。

# 遗留问题
* real server上tcp通过vip连不出来，只能连接到本机
* virtial server上tcp通过vip如果刚好调度到本机的话是可以的，如果调度到real server上tcp超时

该可能跟策略路由有关系

# QA
## 为什么不用keepalived？
由于项目中还用其它服务要要进行集群管理，pacemaker更灵活。

## 为什么使用ldirectord？
使用ldirectord来更新lvs的高度规则，简化配置管理。

## 为什么使用ansible?
为了简化集群部署，使用ansible部署工具实现集群一键部署的目标。
