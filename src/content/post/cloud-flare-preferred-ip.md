---
title: "关于CloudFlare优选ip"
publishDate: "19 Sep 2024"
description: "CloudFlare优选ip实践和原理"
tags: ["CloudFlare"]
draft: false
---

## 前言

前段时间通过[华为开发者大赛](https://jontyding.com/posts/i-got-free-huawei-vps/)白嫖了一台机器，部署了一些简单的服务，其中包括一个[探针](https://probe.jontyding.com)，为了防止ip暴露，选择躲在了CloudFlare背后，但是通过CloudFlare CDN后，访问速度下降了一截，于是有了这篇文章，通过优选ip的方式提升访问速度，并且简单介绍一下CloudFlare CDN的工作方式，如有错误或补充，请给我提出[issues](https://github.com/langchou/langchou.github.io/issues/new)


## CDN工作方式


### 最原始部署方式
假设你有一个服务器，IP是1.2.3.4，并且部署了一个qinglong面板服务，通过一些操作后，你可以通过1.2.3.4:9527访问此qinglong面板，但你不希望通过ip+端口的形式进行访问，因为很不好记，于是你买了一个域名 baidu.com，并且安装了nginx进行反代理，此时你可以通过qinglong.baidu.com访问1.2.3.4:9527服务，并且此时，你会在DNS服务商处配置一条DNS A记录，从qinglong.baidu.com 指向 1.2.3.4

并且此时你通过ping qinglong.baidu.com，也可以看到IP是1.2.3.4


### 通过CloudFlare代理
在该qinglong面板运行了一段时间后，你发现CloudFlare可以提供免费的https证书，并且可以隐藏你的真实IP 1.2.3.4，并且CloudFlare可以为你的站点提供一些防护，于是你把你的baidu.com域名的DNS服务器改到了CloudFlare，并且设置了一条DNS记录：

qinglong.baidu.com 指向 1.2.3.4，并且你开启了CloudFlare的小黄云，也就是CDN代理

现在你可仍然可以通过qinglong.baidu.com访问你的qinglong服务，并且通过ping的形式，你会发现IP从1.2.3.4变成了111.222.222.222，此时你意识到，你对qinglong.baidu.com的访问逻辑变成了： 
qinglong.baidu.com -> CloudFlare(111.222.222.222) -> 1.2.3.4

也就是说，你访问你的域名时，是通过访问CloudFlare提供的网络，并由CloudFlare去与你的真实IP地址去交互。但是此时通过CloudFlare后，你发现访问速度好像慢了一些，但是你觉得也还行，毕竟CloudFlare提供了免费的https证书


### 使用CloudFlare for Saas
你的qinglong服务在安稳运行了一周后，突然有一天你发现qinglong.baidu.com无法访问了，ping了一下后也没有ping通，网上搜了一下后，可能是因为最近在开两会，导致111.222.222.222这个IP被GFW墙了，但是这个IP是CLoudFlare分配的，你尝试去CloudFlare上找这个IP进行更换，找了半天发现好像无法更换，但是你听说taobao.com也是用的CloudFlare，并且一直都可以访问，而且他的访问速度好像比你的要快，那是否可以借用分配给taobao.com的IP去访问CloudFlare呢？

## 设置优选IP

显然是可以的，不过你需要准备两个域名

- 一个是你的baidu.com
- 一个是用于Saas回源的域名

### 设置回源域名

![回源地址DNS设置](https://img.jontyding.com/jonty-imgs/2024/09/a204994713e984d21a2740713ed094e2.png)

首先给回源域名添加一个A记录，解析到真实的服务器IP地址，服务器IP是什么，只有这个回源域名知道，它的工作也只起到回源的作用


### 添加回退源


![设置回退源](https://img.jontyding.com/jonty-imgs/2024/09/d1b4d13edfce90ce7a14e6789bcd34bf.png)

进入CloudFlare管理界面，在图示区域添加回退源，也就是刚刚的回退域名的A记录，回退源生效需要1-2分钟。未开通CloudFlare Saas的同学需要开通一下，添加自定义主机名小于100不会收费


### 检查回退源是否有效
回退源状态：有效，即代表添加成功，此时访问 origin.20010521.xyz，你会看到如下界面

![回退源生效](https://img.jontyding.com/jonty-imgs/2024/09/24377a5ed841b20ad364da2283014b13.png)

此时回退源生效，但并未设置自定义主机名，所以回退源还不知道要回源哪个主机到服务器IP

### 设置自定义主机名

![添加主机名](https://img.jontyding.com/jonty-imgs/2024/09/cab4bbf2fad51d18b7bffd37cfc55178.png)

自定义主机名：填写你想解析的域名，可以为一级或二级，这里我填写了本站域名

### 验证域名所有权

这里我们需要验证域名的所有权

你需要去你的DNS解析面板里添加这两条TXT的解析，但不应该包含主域名

例如复制下来是_acme-challenge.jontyding.com，你应该删去jontyding.com，即为_acme-challenge

添加好之后稍等片刻,等待CF服务器去验证

![验证完成](https://img.jontyding.com/jonty-imgs/2024/09/72f74694e39570fa6a4eb49e78ddfff6.png)

### 添加解析

验证完成后

最后将我们需要访问的域名，CNAME解析到我们的CDN记录上即可

这里通过增加一条CNAME记录，把我们需要的域名直接解析到了csgo.com上



## 对比

最后，使用itDog对优选IP前后进行对比

![优选前](https://img.jontyding.com/jonty-imgs/2024/09/5ab6ac74d74c567d5f5908e7d1448a45.png)

![优选后](https://img.jontyding.com/jonty-imgs/2024/09/10294f079077c3ad8dbccc7d50d08439.png)