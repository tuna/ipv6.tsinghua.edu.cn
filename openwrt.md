---
layout: default 
title: OpenWRT 路由器作为 IPv6 网关的配置
active_nav: home
permalink: /openwrt/
---

# OpenWRT 路由器作为 IPv6 网关的配置

OpenWRT 是一种嵌入式 Linux 操作系统，广泛应用于家用路由器/网关。本文将介绍几种在清华校园网中，使用 OpenWRT 路由器为接入设备提供 IPv6 服务的配置方法。

本文内容综合有多位贡献者，如有疑问、建议，可以参与[内容讨论](https://github.com/tuna/ipv6.tsinghua.edu.cn/issues/7)。

## IPv6 NAT

虽然 IETF 的设计中，IPv6 不再有 NAT (网络地址转换), 但 Linux 内核从 3.7 版本开始实现了 IPv6 的 NAT。早年也曾有过 NAT66 等非官方项目，但缺乏维护，目前已经不再推荐使用。
OpenWRT IPv6 NAT 配置部分，由 [@Blaok](https://github.com/Blaok) 贡献。

### 零: 检查内核模块和有用的软件包

```
ip6tables kmod-ipt-nat6 kmod-ip6tables kmod-ip6tables-extra luci-proto-ipv6 iputils-traceroute6
```

较新的OpenWrt已经内置了IPv6支持。对于较老的版本（Backfire 10.03 或 Attitude Adjustment 12.09及之前），还需要安装`kmod-ipv6`。

`kmod`开头的内核模块一般无法通过opkg直接安装，其他软件包虽然可以通过`opkg install`直接安装，但会多占路由器存储空间，推荐在编译固件时就将这些软件包都放入固件

`kmod-ipt-nat6`提供IPv6 NAT支持，`ip6tables kmod-ip6tables kmod-ip6tables-extra`等提供IPv6防火墙，`luci-proto-ipv6`为LuCI提供IPv6设置选项，`iputils-traceroute6`为IPv6提供traceroute功能(`mtr`是个不错的支持双栈的`traceroute`替代品，如果路由器存储空间够大的话)

### 壹: 打开 OpenWRT IPv6 私网地址分配

 OpenWRT默认会分配IPv6私网地址，在`Network->Interfaces`页面底下有个`Global network options`，`IPv6 ULA-Prefix`这里应该有一个随机的`fd`开头的`/64`地址，LAN客户端应该能自动获得这个地址范围内的IPv6地址，DHCPv6和SLAAC默认都开了

 为了让OpenWRT后面的设备始终能够获得IPv6网关，需要在`Network->Interfaces->LAN`下方的`DHCP Server`部分的`IPv6 Settings`部分，勾选`Always announce default router`。否则，由于默认分配的是私网地址，OpenWRT不会向下游设备公布IPv6默认路由(即网关)，可能导致路由器上IPv6连通但下游设备不通的情况 (感谢[@terro](https://github.com/terro)提醒)

### 贰: 打开 IPv6 NAT

客户端有了正确的地址以后，需要在路由器上打开IPv6 NAT。OpenWRT默认的防火墙配置不会管IPv6的nat表，可以在`/etc/firewall.user`里面加上

```bash
WAN6=eth0
LAN=br-lan
ip6tables -t nat -A POSTROUTING -o $WAN6 -j MASQUERADE
ip6tables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
ip6tables -A FORWARD -i $LAN -j ACCEPT
```

WAN6和LAN分别改成外网IPv6和内网网卡(interface)的名字，注意不是防火墙区域(zone)的名字，也不是LuCI里面`Network->Interfaces`里面看到的名字，而是`ifconfig`看到的名字

### 叁: 正确配置网关

在路由器上`ip -6 route`看一下自己的默认网关。如果获得的是

```
default from 2402:f000:x:xxxx::/64 via fe80::xxxx:xxxx:xxxx:xxxx dev eth0  proto static  metric 512
```

这样坑爹的网关，在转发NAT包的时候会有问题，需要把去掉`from 2402:f000:x:xxxx::/64`这一部分的以后的默认路由添加到路由表中。
下面有两个方法可以选择
1. 可以新建一个`/etc/hotplug.d/iface/99-ipv6`，它的内容是

```bash
#!/bin/sh
[ "$ACTION" = ifup ] || exit 0
iface=wan6
[ -z "$iface" -o "$INTERFACE" = "$iface" ] || exit 0
ip -6 route add `ip -6 route show default|sed -e 's/from [^ ]* //'`
logger -t IPv6 "Add IPv6 default route."
```

这里`iface`是LuCI里面`Network->Interfaces`里面看到的名字，一般叫wan6。这个脚本的意思是在wan6起来以后读取默认网关，把带from的内容去掉，再加到系统路由表里。记得

```
chmod +x /etc/hotplug.d/iface/99-ipv6
```

2. 新建 `/etc/odhcp6c.user`
如果上述内容不起作用（尤其是当你的 IPv6 WAN 是 wlanX 这种不能在 ifup 的时候就能获取到 IPv6 地址的情况下），你可以新建一个`/etc/odhcp6c.user`文件内容如下

```bash
#!/bin/sh
ip -6 route add `ip -6 route show default|sed -e 's/from [^ ]* //'`
logger -t IPv6 "Add IPv6 default route."
```
因为 odhcpd 负责 IPv6 地址的获取和添加，odchpd 启动时会调用 `/lib/netifd/dhcpv6.script`，在运行完毕后会执行 `/etc/odhcp6c.user`，我们借此修改路由
