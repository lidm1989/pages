+++
date = "2016-10-16T17:16:41+08:00"
draft = false
title = "lvs"
thumbnail = "images/shterm/lvs.png"

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
由于lvs要求real server必须禁止arp响应而virtual server必须支持arp响应，所以pacemaker定义资源运行在Master/Slave模式并与ldirectord绑定关系。

### lvs资源
lvs运行在master/slave模式。master节点在指定网卡上配置vip，响应arp，并启用转发；slave节点在网卡lo上配置vip，不响应arp。

响应arp需要通知路由更新arp表，由程序send_arp向路由发送更新消息，该程序由resource-agents的IPaddr中提取，IPaddr位于/usr/lib/ocf/resource.d/heartbeat/IPaddr。

以下agent脚本由pacemaker提供的Stateful模板改编而来，模版默认位于/usr/lib/ocf/resource.d/pacemaker/Stateful。

**文件名：lvs**

    #!/bin/sh
    #
    #
    #	Example of a stateful OCF Resource Agent. 
    #
    # Copyright (c) 2006 Andrew Beekhof
    #
    # This program is free software; you can redistribute it and/or modify
    # it under the terms of version 2 of the GNU General Public License as
    # published by the Free Software Foundation.
    #
    # This program is distributed in the hope that it would be useful, but
    # WITHOUT ANY WARRANTY; without even the implied warranty of
    # MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
    #
    # Further, this software is distributed without any warranty that it is
    # free of the rightful claim of any third person regarding infringement
    # or the like.  Any license provided herein, whether implied or
    # otherwise, applies only to this software file.  Patent licenses, if
    # any, provided herein do not apply to combinations of this program with
    # other software, or any other product whatsoever.
    #
    # You should have received a copy of the GNU General Public License
    # along with this program; if not, write the Free Software Foundation,
    # Inc., 59 Temple Place - Suite 330, Boston MA 02111-1307, USA.
    #

    #######################################################################
    # Initialization:

    : ${OCF_FUNCTIONS=${OCF_ROOT}/resource.d/heartbeat/.ocf-shellfuncs}
    . ${OCF_FUNCTIONS}
    : ${__OCF_ACTION=$1}
    CRM_MASTER="${HA_SBIN_DIR}/crm_master -l reboot"

    NIC=$OCF_RESKEY_nic
    VIP=$OCF_RESKEY_vip
    PREFIX=$OCF_RESKEY_vip_prefix
    PREFIX=32
    PORT=80

    #######################################################################

    meta_data() {
        cat <<END
    <?xml version="1.0"?>
    <!DOCTYPE resource-agent SYSTEM "ra-api-1.dtd">
    <resource-agent name="Stateful" version="1.0">
    <version>1.0</version>

    <longdesc lang="en">
    This is an example resource agent that impliments two states
    </longdesc>
    <shortdesc lang="en">Example stateful resource agent</shortdesc>

    <parameters>

    <parameter name="state" unique="1">
    <longdesc lang="en">
    Location to store the resource state in
    </longdesc>
    <shortdesc lang="en">State file</shortdesc>
    <content type="string" default="${HA_VARRUN}/Stateful-{OCF_RESOURCE_INSTANCE}.state" />
    </parameter>

    <parameter name="nic">
    <longdesc lang="en">network interface card</longdesc>
    <shortdesc lang="en">nic</shortdesc>
    <content type="string" default="eth0"/>
    </parameter>

    <parameter name="vip" required="1">
    <longdesc lang="en">virtual ip</longdesc>
    <shortdesc lang="en">vip</shortdesc>
    <content type="string"/>
    </parameter>

    <parameter name="vip_prefix">
    <longdesc lang="en">virtual ip prefix</longdesc>
    <shortdesc lang="en">prefix</shortdesc>
    <content type="string" default="32"/>
    </parameter>

    </parameters>

    <actions>
    <action name="start"   timeout="20" />
    <action name="stop"    timeout="20" />
    <action name="monitor" depth="0"  timeout="20" interval="10" role="Master"/>
    <action name="monitor" depth="0"  timeout="20" interval="10" role="Slave"/>
    <action name="meta-data"  timeout="5" />
    <action name="validate-all"  timeout="30" />
    </actions>
    </resource-agent>
    END
        exit $OCF_SUCCESS
    }

    #######################################################################

    lvs_cleanup() {
        ip addr del dev lo $VIP/$PREFIX
        ip addr del dev $NIC $VIP/$PREFIX

        sysctl net.ipv4.conf.all.forwarding=0

        sysctl net.ipv4.conf.all.arp_ignore=0
        sysctl net.ipv4.conf.all.arp_announce=0
        sysctl net.ipv4.conf.lo.arp_ignore=0
        sysctl net.ipv4.conf.lo.arp_announce=0

        return $OCF_SUCCESS
    }

    lvs_master() {
        ip addr del dev lo $VIP/$PREFIX
        ip addr add dev $NIC $VIP/$PREFIX

        sysctl net.ipv4.conf.all.forwarding=1

        sysctl net.ipv4.conf.all.arp_ignore=0
        sysctl net.ipv4.conf.all.arp_announce=0
        sysctl net.ipv4.conf.lo.arp_ignore=0
        sysctl net.ipv4.conf.lo.arp_announce=0

        ipvsadm -C

        /usr/libexec/heartbeat/send_arp -i 200 -r 5  $NIC $VIP auto not_used not_used

        return $OCF_SUCCESS
    }

    lvs_slave() {
        ip addr add dev lo $VIP/$PREFIX
        ip addr del dev $NIC $VIP/$PREFIX

        sysctl net.ipv4.conf.all.forwarding=1

        sysctl net.ipv4.conf.all.arp_ignore=1
        sysctl net.ipv4.conf.all.arp_announce=2
        sysctl net.ipv4.conf.lo.arp_ignore=1
        sysctl net.ipv4.conf.lo.arp_announce=2

        ipvsadm -C

        return $OCF_SUCCESS
    }

    stateful_usage() {
        cat <<END
    usage: $0 {start|stop|promote|demote|monitor|validate-all|meta-data}

    Expects to have a fully populated OCF RA-compliant environment set.
    END
        exit $1
    }

    stateful_update() {
        echo $1 > ${OCF_RESKEY_state}
    }

    stateful_check_state() {
        target=$1
        if [ -f ${OCF_RESKEY_state} ]; then
        state=`cat ${OCF_RESKEY_state}`
        if [ "x$target" = "x$state" ]; then
            return 0
        fi

        else
        if [ "x$target" = "x" ]; then
            return 0
        fi
        fi

        return 1
    }

    stateful_start() {
        stateful_check_state master
        if [ $? = 0 ]; then
            # CRM Error - Should never happen
        return $OCF_RUNNING_MASTER
        fi
        stateful_update slave
        lvs_slave
        $CRM_MASTER -v ${slave_score}
        return 0
    }

    stateful_demote() {
        stateful_check_state 
        if [ $? = 0 ]; then
            # CRM Error - Should never happen
        return $OCF_NOT_RUNNING
        fi
        stateful_update slave
        lvs_slave
        $CRM_MASTER -v ${slave_score}
        return 0
    }

    stateful_promote() {
        stateful_check_state 
        if [ $? = 0 ]; then
        return $OCF_NOT_RUNNING
        fi
        stateful_update master
        lvs_master
        $CRM_MASTER -v ${master_score}
        return 0
    }

    stateful_stop() {
        lvs_cleanup
        $CRM_MASTER -D
        stateful_check_state master
        if [ $? = 0 ]; then
            # CRM Error - Should never happen
        return $OCF_RUNNING_MASTER
        fi
        if [ -f ${OCF_RESKEY_state} ]; then
        rm ${OCF_RESKEY_state}
        fi
        return 0
    }

    stateful_monitor() {
        stateful_check_state "master"
        if [ $? = 0 ]; then
        if [ $OCF_RESKEY_CRM_meta_interval = 0 ]; then
            # Restore the master setting during probes
            $CRM_MASTER -v ${master_score}
        fi
        return $OCF_RUNNING_MASTER
        fi

        stateful_check_state "slave"
        if [ $? = 0 ]; then
        if [ $OCF_RESKEY_CRM_meta_interval = 0 ]; then
            # Restore the master setting during probes
            $CRM_MASTER -v ${slave_score}
        fi
        return $OCF_SUCCESS
        fi

        if [ -f ${OCF_RESKEY_state} ]; then
        echo "File '${OCF_RESKEY_state}' exists but contains unexpected contents"
        cat ${OCF_RESKEY_state}
        return $OCF_ERR_GENERIC
        fi
        return 7
    }

    stateful_validate() {
        exit $OCF_SUCCESS
    }

    : ${slave_score=5}
    : ${master_score=10}

    : ${OCF_RESKEY_CRM_meta_interval=0}
    : ${OCF_RESKEY_CRM_meta_globally_unique:="true"}

    if [ "x$OCF_RESKEY_state" = "x" ]; then
        if [ ${OCF_RESKEY_CRM_meta_globally_unique} = "false" ]; then
        state="${HA_VARRUN}/Stateful-${OCF_RESOURCE_INSTANCE}.state"
        
        # Strip off the trailing clone marker
        OCF_RESKEY_state=`echo $state | sed s/:[0-9][0-9]*\.state/.state/`
        else 
        OCF_RESKEY_state="${HA_VARRUN}/Stateful-${OCF_RESOURCE_INSTANCE}.state"
        fi
    fi

    case $__OCF_ACTION in
    meta-data)	meta_data;;
    start)		stateful_start;;
    promote)	stateful_promote;;
    demote)		stateful_demote;;
    stop)		stateful_stop;;
    monitor)	stateful_monitor;;
    validate-all)	stateful_validate;;
    usage|help)	stateful_usage $OCF_SUCCESS;;
    *)		stateful_usage $OCF_ERR_UNIMPLEMENTED;;
    esac

    exit $?

### ldirectord
ldirectord需从官网下载安装。

由于ldirectord没有提供相应agent，我们这里也要增加相应agent。agent脚本可由pacemaker提供的模板改编而来，模版默认位于/usr/lib/ocf/resource.d/pacemaker/Dummy。

这里为了简单，使用了systemd的agent。

**文件名： ldirectord.service**

    [Unit]
    Description=ldirectord.service
    After=network.target

    [Service]
    Type=forking
    PIDFile=/var/run/ldirectord.ldirectord.pid
    ExecStart=/sbin/ldirectord restart
    ExecStop=/sbin/ldirectord stop
    ExecReload=/sbin/ldirectord reload

    [Install]
    WantedBy=multi-user.target

### inventory
资产文件设置了三个主机，使用cluster用户进行部署。请根据实际情况进行调整。

**文件名： hosts**

    [nodes]
    node1 ip=192.168.122.111 ansible_user=cluster
    node2 ip=192.168.122.112 ansible_user=cluster
    node3 ip=192.168.122.113 ansible_user=cluster

    [nodes:vars]
    vip=192.168.122.100
    port=80

### ldirectord.cf.j2
ldirectord配置文件，请根据实际情况进行调整。

**文件名： ldirectord.cf.j2**

    checktimeout=3
    checkinterval=10
    autoreload=yes
    quiescent=yes

    virtual={{ vip }}:{{ port }}
    {% for host in groups['nodes'] %}
        real={{ hostvars[host].ip }}:{{ port }} gate
    {% endfor %}
        checkport={{ port }}
        checktype=connect
        service=simpletcp
        scheduler=lc
        protocol=tcp

### playbook
ansible使用pcs对pacemaker进行配置

这里密码写死123456。网卡为eth0。vip为192.168.122.100。

**文件名： lvs.yml**

    ---
    - hosts: nodes
      become: yes
      tasks:
      - name: Set a password for the hacluster user
        user: name=hacluster password=$6$rounds=656000$JBKx3dx31baJQ.zA$uBwQqkpLY/0SV7JxEHdKGcXfjQx6NZgvKoiZ/Z13NTFjn96UyTAqEKYjaGhJbY1lnT/F2c6zSITD8gpAEiMIe. update_password=always
      - name: Enable pcs
        service: name=pcsd.service state=started enabled=yes
      - name: Authenticate the hacluster user
        command: /usr/sbin/pcs cluster auth {{ groups['nodes'] | join(' ') }} -u hacluster -p 123456
        run_once: true
      - name: Generate and synchronize the corosync
        command: /usr/sbin/pcs cluster setup --name mycluster {{ groups['nodes'] | join(' ') }}
        run_once: true
      - name: Enable corysync
        service: name=corosync.service state=started enabled=yes
      - name: Enable pacemaker
        service: name=pacemaker.service state=started enabled=yes
      - name: Set property stonith
        command: /usr/sbin/pcs property set stonith-enabled=false
        run_once: true
      - name: Set rosource defaults
        command: /usr/sbin/pcs resource defaults resource-stickiness=100
        run_once: true

      - name: Create lvs agent
        copy: src={{ playbook_dir }}/lvs dest=/usr/lib/ocf/resource.d/heartbeat/lvs mode=755
      - name: Create lvs resource
        command: /usr/sbin/pcs resource create lvs ocf:heartbeat:lvs params nic=eth0 vip=192.168.122.100 --master
        run_once: true

      - name: Create ldirectord.cf
        template: src={{ playbook_dir }}/ldirectord.cf.j2 dest=/etc/ha.d/ldirectord.cf     
      - name: Create ldirectord.service
        copy: src={{ playbook_dir }}/ldirectord.service dest=/etc/systemd/system/ldirectord.service
      - name: Create ldirectord resource
        command: /usr/sbin/pcs resource create ldirectord systemd:ldirectord
        run_once: true

      - name: Create colocation constraint
        command: /usr/sbin/pcs constraint colocation add ldirectord with master lvs-master
        run_once: true
      - name: Create order constraint
        command: /usr/sbin/pcs constraint order promote lvs-master then ldirectord
        run_once: true

# 部署
ansible的配置这里省略了。部署命令：

    ansible-playbook -i hosts lvs.yml -v

# 验证
在各机器启用验证实例：

    nc -l 80 -e /usr/bin/hostname -k

在其它节点验证：

    curl 192.168.122.100

# 遗留问题
* real server上不要连接vip，tcp通过vip连不出来，只能连接到本机。
* virtial server上不要连接vip，tcp通过vip如果刚好调度到本机可以连通，如果调度到real server上tcp超时。

以上两点可能跟策略路由有关系，该问题可由应用程序解决。

* 断网卡时vip可能挂不上**（硬伤）**。corosync能检测到，但pacemaker没有动作。

这点是硬伤，还需要进一步调查。

# Q&A
### 为什么使用ldirectord？
使用ldirectord来更新lvs的高度规则，简化配置管理。

### 为什么不用keepalived？
如果只使用lvs调度，由于keepalived集成了ldirectord与pacemaker的功能，更简单。

但由于我们项目中还有其它服务要进行集群管理，使用pacemaker更灵活。

### 为什么使用ansible?
为了简化集群部署，使用ansible部署工具实现集群一键部署的目标，如上只要一条命令就可以部署集群。
