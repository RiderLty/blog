---
title: 初始化
published: 2024-10-17
description: '关于此博客'
image: "./cover.jpg"
tags: [blog,markdown,vscode]
category: '网站相关'
draft: false 
lang: ''
---
# Link Start ！

关于此bolg相关信息。

## 作用：

用于记录自己折腾的一些东西，以便于在忘记了差不多的时候，还能明白当初的设计逻辑。

当然还有自己踩过的坑。

如果我记录的这些信息能够帮助到其他人，那就更好了。

## 搭建：

搭建倒是很是简单，参考 [github.com/saicaca/fuwari](https://github.com/saicaca/fuwari)

按照README的指引，初始化下仓库依赖，然后```pnpm dev --host 0.0.0.0```就可以开启dev服务器。

内容使用MarkDown实现，使用vscode即可一站式完成编写与预览。

目前使用的是NAS上的LXC容器，vscode远程连过去。

提交导github后，自动触发cloudflare pages 部署工作流。