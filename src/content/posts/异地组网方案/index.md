---
title: 异地组网方案
published: 2024-10-18 
description: '使用tailscale来异地组网'
image: "./cover.jpg"
tags: [tailscale,openwrt,ipv6]
category: 'tailscale'
draft: false 
lang: ''
---

::github{repo="tailscale/tailscale"}

[tailscale](https://tailscale.com/) 可以快速的搭建起一个虚拟局域网，特别是当你有公网IPv4或者Ipv6的时候。

本文讨论通过软路由实现无感的异地组网，这也是体验最佳的方案。

最终实现的效果为多个不同局域网下的设备可以直接使用内网IP互相访问。

:::note[注意事项]
tailscale是一款处于更新中的软件，本文写于2024-10-18

路由器为 iStoreOS 22.03.5 2023121510，tailscale版本为1.32.3-1 (OpenWrt)
:::

首先将每个路由器的LAN网段修改为不冲突的，如：192.168.3.0/24 ; 192.168.7.0/24。

然后分别安装tailscale：
```shell 
opkg update && opkg install tailscale
```

使用如下命令，开启tailscale并登录。
```shell
tailscale up --accept-routes --advertise-exit-node --advertise-routes=192.168.3.0/24
```

其中```--accept-routes```表示接受作为子网路由器的节点的通告路由。

将```--advertise-routes=192.168.3.0/24```中通告的网段换为本路由器的LAN网段。

在所有路由器上操作后，在路由器下的设备即可通过内网IP直接访问不同网段的局域网设备。

![ping](ping.png)