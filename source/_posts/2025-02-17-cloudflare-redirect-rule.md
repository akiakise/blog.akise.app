---
title: 利用 Cloudflare 重定向规则摆脱端口访问服务
tags: [Cloudflare]
categories: Technology
date: 2025-02-17 10:32:09
description: 在 80/443 端口无法顺畅提供服务时往往需要暴露其他端口，而服务数量多了之后端口记忆也是一个问题，本文通过借助 Cloudflare 的功能来实现“无端口”访问非 80/443 端口的服务。
cover: /img/2025/cloudflare-redirect-rule/cover.png
banner: /img/2025/cloudflare-redirect-rule/cover.png
topic: play
---

如果你拥有一个国内服务器或者你的宽带有公网 IP，那么你一定知道在当前环境下尝试通过 80/443 提供服务有多困难。多数情况下我们只能另选其他端口，然而非 80/443 端口在访问时必须准确写出端口，这在服务数量众多的情况下成本非常高。

本文通过借助 Cloudflare 的源站规则来实现“无端口”访问非 80/443 端口的服务，至少可以简化记忆端口的成本。

# 端口映射 - 放开外网访问

假设我们在内网存在数个服务：

1. 服务1 vaultwarden，在 192.168.10.2，端口为 8081
2. 服务2 navidrome，在 192.168.10.3，端口为 8082

我们首先在路由器的外网端口映射配置映射规则，确保外网可以直接通过 `公网IP+8081` 访问到服务 1、通过 `公网IP+8082` 访问到服务 2：

{% image /img/2025/cloudflare-redirect-rule/ddns-forward.png 端口映射配置 %}

**这是一切的前提**，如果你配置了端口映射规则之后在外网通过 `公网IP+端口` 无法访问你内网的服务，那么你的公网 IP 是有问题的。

# DDNS - 跟内网服务绑定域名

**在 DDNS 中跟本地 IP 绑定多个域名**：假设服务 1 为 vaultwarden，服务 2 为 Navidrome，则我们使用 DDNS 同时将公网 IP 注册到两个域名下：

1. vaultwarden.example.com -> 公网IP
2. navidrome.example.com -> 公网IP

# 创建规则

进入 Cloudflare 后台选中你的域名后转到{% mark 规则 %}页面，创建一个{% mark 源服务器规则 %}:

{% image /img/2025/cloudflare-redirect-rule/create-rule.png 创建规则 %}

然后填充表单：

{% image /img/2025/cloudflare-redirect-rule/edit-rule.png 编辑规则 %}

接下来，在 DNS 页面填入我们刚才在规则页面写的主机名，并打开{% mark Proxy status color:yellow %}配置：

{% image /img/2025/cloudflare-redirect-rule/dns.png DNS 配置 %}

针对上边我们举例的两个服务，那就是创建两个规则：

1. 规则1，匹配主机名 vaultwarden.example.com，映射到端口 8081
2. 规则2，匹配主机名 navidrome.example.com，映射到端口 8082

我们以访问 vaultwarden.example.com 为例：

1. 浏览器访问 vaultwarden.example.com
2. Cloudflare 匹配到源服务器规则，将请求内部路由到 vaultwarden.example.com:8081
3. Cloudflare 解析 vaultwarden.example.com 得到公网 IP
4. Cloudflare 通过公网IP + 8081 访问服务

这样，我们就实现了 DDNS + 端口隐藏。
