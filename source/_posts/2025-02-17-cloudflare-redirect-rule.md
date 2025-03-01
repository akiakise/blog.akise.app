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

本文通过借助 Cloudflare 的重定向规则来实现“无端口”访问非 80/443 端口的服务，至少可以简化记忆端口的成本。

# 创建规则

进入 Cloudflare 后台选中你的域名后转到{% mark 规则 %}页面:

{% image /img/2025/cloudflare-redirect-rule/create-rule.png 创建规则 %}

然后填充表单：

{% image /img/2025/cloudflare-redirect-rule/edit-rule.png 编辑规则 %}

接下来，在 DNS 页面填入我们刚才在规则页面写的主机名，并打开{% mark Proxy status color:yellow %}配置：

{% image /img/2025/cloudflare-redirect-rule/dns.png DNS 配置 %}

# 实际访问

创建完成后稍等一段时间让 DNS 生效，我们再访问 `https://d.akise.app` 就会自动跳转到 `http://192.168.10.20:9000`，实现了“无端口”访问（其实就是把记录端口的事情交给 Cloudflare 来做了）:

{% image /img/2025/cloudflare-redirect-rule/result.png 实现效果 %}

你还可以在填充表单时选中 `Preserve query string` 来让跳转后的链接保留 URL query string，以便无缝提供 API 服务，不过我并没有这样的需求，暂时没有验证过这块。