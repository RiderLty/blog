---
title: 为内网应用添加域名
published: 2024-10-20
description: ''
image: 'cover.png'
tags: [openwrt,host,dnsmasp,nginx]
category: 'NAS'
draft: false 
lang: ''
---

如果你和我一样，内网部署了一堆应用，但是又经常懒得再从导航页跳转

那么你可以通过.local域名来访问你的网页应用。

使用到的工具：nginx，dnsmasq，openwrt

## 原理部分

通过在路由器上使用自自定义host文件，实现使用自定义域名访问```http://[IP]:80```。

使用nginx反向代理应用网页，仅在80端口提供服务并通过域名区分反向代理服务器。

## 配置部分

访问 “路由器后台 > 网络 > DHCP/DNS > HOST和解析文件”

点击 “添加额外的HOSTS文件”

host文件示例：
```host
192.168.3.1 router.local
192.168.3.3 nas.local
192.168.3.3 nav.local
192.168.3.3 alist.nas.local
192.168.3.3 portainer.nas.local
192.168.3.3 qbittorrent.nas.local
192.168.3.3 librespeed.nas.local
```

将对应的域名解析到部署了nginx的机器上。

可以全用软路由，但我选择分开，每个机器只反向代理自己的服务。

:::important[重要提示]
将原本部署在80端口的修改为非80端口，否则nginx将启动失败。
:::

:::tip[重启Dnsmasq]
```shell
/etc/init.d/dnsmasq restart 
```
:::

nginx添加对应的配置，模版如下：
```nginx
server {
    listen 80;#端口80无需修改
    server_name librespeed.nas.local;#这里修改为对应的域名
    location / {
        proxy_pass http://localhost:8005;#这里是对应服务的地址
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host:$server_port;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Host $host;
        proxy_redirect http:// $scheme://;
    }
}
```

从此域名直达，再也不用记端口 ✌️

在异地组网的情况下依然适用，只需要在电脑或者路由器上有对应的host条目即可。 ~~其实是我整不明白DNS~~