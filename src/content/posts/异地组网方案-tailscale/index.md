---
title: 异地组网方案-tailscale
published: 2024-10-18 
description: '使用tailscale来异地组网'
image: "./cover.jpg"
tags: [tailscale,openwrt,ipv6]
category: 'tailscale'
draft: false 
lang: ''
---

::github{repo="tailscale/tailscale"}

[tailscale](https://tailscale.com/) 可以很方便搭建起一个虚拟局域网，但是需要安装在每一台机器，且需要使用虚拟局域网的IP才能互相访问。

而安卓又无法在wifi下获取IPV6，只有用移动数据才能实现直连。

本文讨论通过软路由实现无感的异地组网，实现多个不同局域网下的设备无需安装tailscale直接使用内网IP跨网段互相访问。

:::note[注意事项]
tailscale是一款处于更新中的软件，本文部分信息可能已过时

本文中使用的路由器版本为 iStoreOS 22.03.5 2023121510，tailscale版本为 1.32.3-1 (OpenWrt)
:::

修改路由器的LAN网段，确保不冲突，如：192.168.3.0/24 ; 192.168.7.0/24。

分别在每台路由器上安装tailscale：
```shell 
opkg update && opkg install tailscale
```

使用如下命令，开启tailscale并登录。
```shell
tailscale up --accept-routes --advertise-exit-node --advertise-routes=192.168.3.0/24
```

```--accept-routes``` 接受其他节点通告的子网路由。

```--advertise-exit-node``` 开启出口节点，可以让其他设备都通过本设备访问网络，可选则是否开启。

```--advertise-routes=192.168.3.0/24``` 通告192.168.3.0/24网段通过本设备访问，根据路由器实际LAN网段修改。

在所有路由器上操作完成后，任一路由器下的设备都可通过内网IP直接访问不同网段的局域网设备。

![ping](ping.png)