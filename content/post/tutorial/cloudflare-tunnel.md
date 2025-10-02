---
date: "2025-10-02T22:46:14+08:00"
draft: false
title: "使用 Cloudflare Tunnel 实现内网穿透"
slug: "cloudflare-tunnel"
description: "这个不需要了, 指 Tailscale Funnel"
image:
categories:
    - 教程
tags:
    - 网络
    - 软路由
    - 内网穿透
license: GNU General Public License v3.0
---

## 使用步骤

### 开通 Tunnel

在 Cloudflare Dashboard 中找到 **Zero Trust**, 进入零信任服务首页. 侧栏找到**网络**栏目中的 **Tunnels**. 选择使用 `cloudflared` 创建隧道. 复制创建的命令, 其中包含 tunnel token.

### 配置软路由

在软路由编译时启用 `luci-app-cloudflared` 或者安装对应插件, 然后在 OpenWrt 侧栏找到 **VPN** 栏目中的 **Cloudflare 零信任隧道**, 勾选**启用**, 并粘贴上一步复制的 token. 点击保存并应用. 等服务启动后, 应该可以在 Cloudflare Tunnel 面板中看到 Tunnel 状态变为健康.

### 配置域名

Tunnel 只能绑定已托管在 Cloudflare 的域名. 要将域名映射到内网中的服务, 应该添加**已发布应用程序路由**. 在已发布应用程序路由界面, 点击添加已发布应用程序路由, 输入一个你想要的子域, 然后在服务类型中选择相应的服务(通常为 HTTP), URL 则可以填写**局域网中可用的 URL**!

也就是说, 部署在软路由上的 Tunnel 不仅可以穿透自身, 还可以将内网中的服务暴露到公网上, 这真是太方便了. 你可以在 URL 中填写 `192.168.114.114:11451`, 甚至是 `homo.lan`, 哪怕这并不是路由器本体的地址, 但只要在内网中可以访问, 通通都可以穿透出来!
