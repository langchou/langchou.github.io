---
title: "使用CloudFlare Argo Tunnel穿透内网服务"
publishDate: "19 Sep 2024"
description: "CloudFlare Argo Tunnel穿透"
tags: ["CloudFlare", "Argo Tunnel"]
draft: false
---

## 背景
依旧是CloudFlare的一个教程，关于使用Argo Tunnel来穿透一些本地没有公网的服务

## 准备材料
一个域名，并解析到CloudFlare

## 配置CloudFlare Argo Tunnel


### 安装CloudFlare CLI
如果未安装`cloudflared`，按照以下步骤安装，其他系统请查询相关资料

Ubuntu/Debian
``` bash
sudo apt-get update && sudo apt-get install cloudflare
```

安装好后，使用一下命令登陆到CloudFlare账号

### 登陆CloudFlare账号
``` bash
cloudflared tunnel login 
```
这个命令会打开一个链接，在浏览器中进行验证登陆CLoudFlare账号


### 创建一个Argo Tunnel
接下来，创建一个Argo Tunnel，用来连接你的服务器上的服务

``` bash
cloudflared tunnel create server-tunnel
```

上述的命令会穿件一个名为server-tunnel的隧道，并会返回给你一个隧道id

例如 `Created tunnel server-tunnel with id 0ba13a92-eqwe-asdb-aa71-cqwdklj204c8`

记录下这个id，后续会使用

## 配置Argo Tunnel代理到服务

假设你的 服务运行在本地 9527 端口上，你需要创建一个配置文件来配置隧道代理

创建一个config.yml文件，例如放在`/etc/cloudflared/`目录下

``` bash
sudo nano /etc/cloudflared/config.yml
```
然后将以下内容填入`config.yml`文件中

``` yaml
tunnel: server-tunnel # 使用生成的隧道ID
credentials-file: /root/.cloudflared/your-tunnel-id.json # 替换为你的隧道凭证路径

ingress:
  - hostname: server.qqq.com  # 二级域名
    service: http://localhost:9527 # 服务的端口
  - service: http_status:404
```

### 添加CNAME

使用以下命令给二级域名 server.qqq.com 添加一条CNAME记录，指向你本地的server

``` bash 
cloudflared tunnel route dns server-tunnel server.qqq.com
```

### 启动Argo Tunnel

``` bash
sudo nano /etc/systemd/system/cloudflared.service
```

编辑 cloudflared.service 文件，然后写入一下内容，需要根据自己的路径和配置进行调整

``` ini
[Unit]
Description=cloudflared service
After=network.target

[Service]
Type=simple
Restart=on-failure
ExecStart=/usr/local/bin/cloudflared tunnel --no-autoupdate run <your-tunnel-name>
User=root
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

### 重新加载 systemd 守护进程
``` bash
sudo systemctl daemon-reload
```

启动`cloudflared`服务并设置开机自动启动：
``` bash
sudo systemctl start cloudflared
sudo systemctl enable cloudflared
```
