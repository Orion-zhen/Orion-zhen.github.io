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

### Split DNS

有时候你可能会想在一个配置文件中统一写 tunnel 的域名, 但希望在家的时候就不用再经过一次 tunnel 中转了. 这时候可以使用软路由的 Split DNS 功能, 在软路由上将那个域名直接指向你内网中的设备. 在内网设备上通过 Nginx 来做反向代理, 将对域名的请求转发到相应的端口上. 唯一的代价是部署证书会比较麻烦. 如果你能接受在家里的浏览器上打不开对应的域名, 那也无所谓. 至少我是这样来处理我的自部署大模型 API 的.

例如, 在软路由上将 `api.example.com` 的域名指向内网的 `homo.lan`, 然后在 `homo.lan` 上启用 Nginx, 监听 `80` 和 `443` 端口, 将对 `api.example.com` 的请求转发到实际部署服务的端口.

### 申请证书

可以通过 `certbot` 申请一份 SSL 证书. 首先在 Cloudflare 官网上申请一个 API token. Cloudflare 首页 -> 用户头像 -> 配置文件 -> API 令牌.

选择创建令牌, 选择**编辑区域 DNS**, 在区域资源中选择特定的域名, 之后会得到一个 API token.

在电脑上安装 `certbot` 和对应的 Cloudflare certbot 插件. 创建文件 `cloudflare.ini`:

```ini
dns_cloudflare_api_token = 你的Cloudflare_API_Token
```

然后申请证书:

```bash
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials cloudflare.ini \
  --no-eff-email \
  -d example.com \
  -d '*.example.com'
```

成功后会得到:

```text
/etc/letsencrypt/live/example.com/fullchain.pem
/etc/letsencrypt/live/example.com/privkey.pem
```

于是可以在 Nginx 中使用了:

```conf
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
```
