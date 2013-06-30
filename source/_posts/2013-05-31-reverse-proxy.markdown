---
layout: post
title: "ssh+nginx反向代理访问内网Web Service"
date: 2013-05-31 14:14
comments: true
categories: [linux, misc]
---

在一般的网络环境下，互联网无法直接访问到局域网的计算机，如果你想在家里访问公司局域网的计算机或者在公司想访问家里的计算机，那么需要借助特定的工具，这里以ssh+nginx反向代理访问局域网计算机的web服务器为例。
假定你有一台具有公网IP的服务器S1(211.83.241.83)以及一台在局域网开启web服务的计算机S2(172.16.18.3)，如下图所示:

![reverse-proxy-net.png](/images/reverse-proxy-net.png)

###概述

S2通过ssh连接S1开启反向隧道，它在服务器上开启一个监听端口12345同时在将该端口影射在本地web服务端口8080上，S1上开启ngix反向代理将80端口代理到12345上，用户通过访问S1时nginx将请求代理到12345端口上，ssh通过建立好的隧道将请求传输到S2，S2从隧道中接收请求并转发到监听8080端口的web服务，并将结果通过隧道返回给S1，S1从隧道获取结果返回给nginx，nginx再将请求结果返回给用户，从而访问到局域网内的web服务。

###配置

1. 在你所需要访问的web service的内网服务器S2上执行以下命令:
    {% codeblock %}
    ssh -f -N -R <服务器端口>:<本地IP>:<本地端口> <服务器地址>
    {% endcodeblock %}

例如:
    
    {% codeblock %}
    ssh -f -N -L 12345:127.0.0.1:8080 211.83.241.83
    {% endcodeblock %}

该命令执行后会将远程服务器S1(211.83.241.83:12345)映射到本地端口8080

2. 在公网服务器S1上开启nginx反向代理:

    例如:

    {% codeblock %}
    server {
        listen       80;
        server_name  localhost;
        location / {
            proxy_pass  http://127.0.0.1:12345;
            proxy_set_header   Host             $host:80;
            proxy_set_header   X-Real-IP        $remote_addr;
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
            proxy_set_header Via    "nginx";
        }
    }
    {% endcodeblock %}
