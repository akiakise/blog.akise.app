---
title: Clash.Meta DNS 配置指南
tags: [Clash.Meta, Mihomo, DNS]
date: 2025-02-23 11:49:12
categories: Technology
description: Clash DNS 配置指南，详细描述 Clash 中 DNS 解析流程与推荐配置
cover: /img/2025/clash-dns-configure/cover.png
banner: /img/2025/clash-dns-configure/cover.png
topic: play
---

随着互联网技术的发展与广告联盟的互联互通，大家对隐私的重视程度也越来越高，而 DNS 作为网络基础设施，其{% mark 泄露 color:error %}的风险与泄露后导致的问题影响也越来越大。泄露可能涉及的问题主要有：用户通过虚拟专用网络或网络代理请求的域名被记录或审查、用户浏览历史被记录和分析用于广告投放或数据挖掘。同时 ISP 的 DNS 也天生自带污染和和劫持，并不推荐使用。

{% quot A DNS leak is a security flaw that allows DNS requests to be revealed to ISP DNS servers, despite the use of a VPN service to attempt to conceal them. Although primarily of concern to VPN users, it is also possible to prevent it for proxy and direct internet users. %}

[虚空终端](https://wiki.metacubex.one/)作为开源项目[原神](https://yuanshen.com)的二次开发版本，提供了更强的 DNS 处理能力，通过简单且合理的配置，可以尽最大可能规避 DNS 污染、劫持、泄露问题（不能做 100% 保证，毕竟不清楚还有没有未知的手段）。

# DNS 解析流程

下图来自虚空终端 wiki: [DNS 解析流程](https://wiki.metacubex.one/config/dns/diagram/)

{% image /img/2025/clash-dns-configure/cover.png DNS Resolve %}

# 配置指南

我们直接给出推荐配置，并结合上边的解析流程来详细分析：

```yaml
geodata-mode: true 
geodata-loader: memconservative
geo-auto-update: true
geo-update-interval: 24
geox-url:
  geoip: "https://testingcf.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/geoip.dat"
  geosite: "https://testingcf.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/geosite.dat"
  mmdb: "https://testingcf.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/country.mmdb"
  asn: "https://github.com/xishang0128/geoip/releases/download/latest/GeoLite2-ASN.mmdb"
dns:
  enable: true
  prefer-h3: false # DOH 优先使用 http/3
  use-hosts: true # 是否回应配置中的 hosts，默认 true
  use-system-hosts: false # 是否查询系统 hosts，默认 true，我们不用系统 hosts
  listen: 0.0.0.0:53 # DNS 服务监听，仅支持 udp
  ipv6: true # ipv6 解析开关
  default-nameserver: # 默认 DNS, 用于解析 DNS 服务器 的域名，必须为 IP, 可为加密 DNS
    - 1.2.4.8
    - tls://223.5.5.5
  enhanced-mode: fake-ip # or reder-host
  fake-ip-range: 172.29.0.1/16 # Fake-IP 解析地址池
  fake-ip-filter: # Fake-IP 过滤，列表中的域名返回真实 IP
    - '+.lan'
    - '+.in-addr.arpa'
  # 配置后面的nameserver、fallback和nameserver-policy向dns服务器的连接过程是否遵守遵守rules规则
  # 如果为false（默认值）则这三部分的dns服务器在未特别指定的情况下会直连
  # 如果为true，将会按照rules的规则匹配链接方式（走代理或直连），如果有特别指定则任然以指定值为准
  # 仅当proxy-server-nameserver非空时可以开启此选项, 强烈不建议和prefer-h3一起使用
  # 此外，这三者配置中的dns服务器如果出现域名会采用default-nameserver配置项解析，也请确保正确配置default-nameserver
  respect-rules: false
  # 配置查询域名使用的 DNS 服务器
  nameserver-policy:
    "+.lan,+.in-addr.arpa": system
    # 广告拦截，效果有限，请使用浏览器扩展等更有效的手段
    "geosite:category-ads-all": rcode://success
    # 禁止 apple 更新检测
    "geosite:apple-update": rcode://success
    "rule-set:direct":
      # 阿里 DNS
      - tls://223.5.5.5
      - tls://223.6.6.6
      - 'https://223.5.5.5/dns-query#h3=true'
      - 'https://223.6.6.6/dns-query#h3=true'
    "geosite:private,onedrive,microsoft@cn,apple@cn,steam@cn,google@cn,jetbrains@cn,category-games@cn,cn":
      # 阿里 DNS
      - tls://223.5.5.5
      - tls://223.6.6.6
      - 'https://223.5.5.5/dns-query#h3=true'
      - 'https://223.6.6.6/dns-query#h3=true'
  proxy-server-nameserver:
    - 'https://223.5.5.5/dns-query#h3=true'
    - 'https://223.6.6.6/dns-query#h3=true'
  nameserver:
    # Cloudflare
    - 'tls://1.1.1.1#PROXY'
    - 'tls://1.0.0.1#PROXY'
    - 'https://1.1.1.1/dns-query#PROXY&h3=true'
    - 'https://1.0.0.1/dns-query#PROXY&h3=true'
    # Google
    - 'tls://8.8.8.8#PROXY'
    - 'tls://8.8.4.4#PROXY'
    - 'https://8.8.8.8/dns-query#PROXY&h3=true'
    - 'https://8.8.4.4/dns-query#PROXY&h3=true'
    # 101
    - 'tls://101.101.101.101#PROXY'
    - 'https://101.101.101.101/dns-query#PROXY'
    # dns.sb
    - 'tls://185.222.222.222#PROXY'
    - 'tls://45.11.45.11#PROXY'
    - 'https://185.222.222.222/dns-query#PROXY'
    - 'https://45.11.45.11/dns-query#PROXY'
rule-providers:
  direct:
    behavior: classical
    type: inline
    payload:
      # CDN
      - 'DOMAIN,download-cdn.jetbrains.com'
      # Clash Dashboard & Converter
      - 'DOMAIN,clash.razord.top'
      - 'DOMAIN,yacd.haishan.me'
      - 'DOMAIN,nexconvert.com'
      # DDNS
      - 'DOMAIN-SUFFIX,ip.sb'
      # time
      - 'DOMAIN,time.android.com'
  proxy:
    behavior: classical
    type: inline
    payload:
      # 这些域名虽然在国内有接入点，但是直连体验并不好，因此强制代理
      - 'DOMAIN,dockerstatic.com'
      - 'DOMAIN,fonts.googleapis.com'
      - 'DOMAIN,fonts.gstatic.com'
      - 'DOMAIN,www.gstatic.com'
  reject:
    behavior: classical
    type: inline
    payload:
      # niconico 广告
      - 'DOMAIN-SUFFIX,ads.nicovideo.jp'
      # mumu 模拟器广告
      - 'DOMAIN-SUFFIX,mumu.nie.netease.com'
      # postman 更新
      - 'DOMAIN-SUFFIX,dl.pstmn.io'
      - 'DOMAIN-SUFFIX,sync-v3.getpostman.com'
      - 'DOMAIN-SUFFIX,getpostman.com'
rules:
  # 个人规则，1.2.3.4 用作示例
  - IP-CIDR,1.2.3.4/32,PROXY,no-resolve
  # 规则修正
  - RULE-SET,reject,REJECT
  - RULE-SET,proxy,PROXY
  - RULE-SET,direct,DIRECT

  # 广告过滤
  - GEOSITE,category-ads-all,REJECT
  # 强制直连
  - GEOSITE,private,DIRECT
  - GEOSITE,onedrive,DIRECT
  # @cn 为该规则内的域名在中国大陆有接入点，可直连
  - GEOSITE,microsoft@cn,DIRECT
  - GEOSITE,apple@cn,DIRECT
  - GEOSITE,steam@cn,DIRECT
  - GEOSITE,google@cn,DIRECT
  - GEOSITE,jetbrains@cn,DIRECT
  - GEOSITE,category-games@cn,DIRECT
  # 流媒体
  - GEOSITE,biliintl,国际流媒体
  - GEOSITE,youtube,国际流媒体
  # 强制代理
  - GEOSITE,google,PROXY
  - GEOSITE,twitter,PROXY
  - GEOSITE,pixiv,PROXY
  - GEOSITE,category-scholar-!cn,PROXY
  - GEOSITE,geolocation-!cn,PROXY
  # 兜底强制直连
  - GEOSITE,cn,DIRECT

  # GEOIP 规则
  - GEOIP,private,DIRECT
  - GEOIP,telegram,PROXY
  - GEOIP,JP,PROXY
  - GEOIP,CN,DIRECT

  # 兜底代理
  - MATCH,PROXY
```

在这个配置中，最最核心的就是利用 `GEOSITE` 来进行域名规则的判断，以下我们分为几种情况来分析其解析流程。

## 1. 直接通过 ip 访问

直接通过 `1.2.3.4` 访问会命中 `IP-CIDR,1.2.3.4/32,PROXY,no-resolve` 规则，它会跳过 DNS 解析流程并直接交给对应的策略处理（这里配置为 PROXY 策略）。

假设我们同时有一个域名 `example.com` 也指向了 `1.2.3.4`，那我们访问域名的时候并不会走到这条规则，因为规则添加了 `no-resolve` 条件，除非是直接通过 IP 访问，否则直接跳过此规则。

## 2. 访问境内域名，命中 GEOSITE 规则

当访问的境内域名命中 `GEOSITE` 规则时，由于指定了 `DIRECT` 策略，虚空终端会进入 DNS 解析流程，结合配置文件来说流程如下：

1. 查询 DNS 缓存，如果查到，拿出对应的 IP 并直接建立连接
2. 匹配 `nameserver-policy`，我们将所有的境内规则的 `GEOSITE` 也都配置在了这里，因此肯定会命中这条规则。在 `nameserver-policy` 内容里我们配置了 AliDNS 的 DoT 和 DoH，虚空终端会并发查询这些 nameserver 并返回最快返回的结果
3. 拿到域名解析结果，直接建立连接，并保存 DNS 缓存

## 3. 访问境外域名，命中 GEOSITE 规则

当访问的境外域名命中 `GEOSITE` 规则时，由于指定了 `PROXY` 策略，因此虚空终端会直接将域名解析与请求处理路由到 `PROXY` 节点，跳过自身所有的 DNS 解析流程。

## 4. 访问任意域名，未命中境内或境外 GEOSITE 规则

境内境外 `GEOSITE` 规则未命中时，规则解析一路往下走到 `GEOIP` 规则，`GEOIP` 规则判断前需要先拿到域名对应的 IP，因此虚空终端会发起一次 DNS 解析流程：

1. 查询 DNS 缓存，如果查到，拿出对应的 IP 交给 `GEOIP` 规则判断
2. 匹配 `nameserver-policy`，我们在这里配置的 `GEOSITE` 规则跟 `rules` 里用的 `GEOSITE` 规则是一致的，因此也无法命中
3. 兜底使用 `nameserver`，在内容里我们配置了常用的境外 DoT 和 DoH，并且指定这些 DNS 查询时使用 `PROXY` 节点以加速（境外 DNS 境内直接访问延迟很高），同时配置了 `proxy-server-nameserver` 来避免鸡蛋问题（考虑到解析速度配置成了 AliDNS，无奈之举），另外节点如果支持 H3 则尽可能使用 H3（这点存疑，境内到境外 H3 稳定性不确定）
4. 拿到域名解析结果，交给 `GEOIP` 规则判断，并保存 DNS 缓存
5. 如果 `GEOIP` 规则判断直连，则直接建立连接，如果 `GEOIP` 规则未命中，则走到兜底使用代理建立连接

可以看到，我们兜底使用远程节点来解析未命中规则的域名，我个人倾向于假设未命中规则的都是境外域名，因此都由境外 DNS 解析来避免国内 DNS 保留记录。当然这里也可以改为使用境内 DNS 解析，两种策略任君自选。

## 5. 规则修正

规则库不是万能的，因此必然会出现某些域名分流错误的情况，我们通过补充的 `direct`、`proxy`、`reject` 三个 `RULE-SET` 来前置修正规则。

## 补充说明: rule-providers

我们将强制修正的规则通过 `rule-providers` 维护，这样做的好处是可以在 `rules` 和 `dns` 中使用 `rule-set` 引用同一份配置，无需重复编写。在配置内容上使用 `inline` + `classical` 配合的形式，使得我们无需依赖远程仓库（旧做法是使用 GitHub 仓库或者 gist 托管），同时又可以使用 `DOMAIN-SUFFIX`、`DOMAIN-KEYWORD` 等特性，调整成本非常低。

## 补充说明: default-nameserver、fallback

`default-nameserver`: 用于解析 `nameserver`、`fallback`、`nameserver-policy` 中通过域名指定的 DNS 服务器，我们使用 IP 配置 DoT 和 DoH 节点，因此不再需要此配置。
`fallback`: 虚空终端之前 Clash 使用的策略，逻辑比较复杂，不再推荐，使用更加简单易懂的 `nameserver-policy` 替代。

## 补充说明: GEOIP + no-resolve 是错误用法

网上存在很多在 `GEOIP,CN,DIRECT` 后加上 `no-resolve` 的，这实际上是一个{% mark 错误 color:error %}的用法，`GEOIP` 类规则就是要解析出 IP 后再进行 IP 判定的，加了 `no-resolve` 后如果不是直接通过 IP 访问的话，这条规则就失效了。

同时，虚空终端的规则处理流程是从上往下严格有序的，如果遇到了 IP 类规则（GEOIP、IP-CIDR、IP-CIDR6 等）会尝试进行 DNS 解析拿到 IP 再判断这条规则，如果某个 IP 类规则不带 `no-resolve` 并且放的比较靠前，那么必然会产生多余的 DNS 解析。

因此，这里的正确用法是{% mark 将所有的 IP 类规则（GEOIP、IP-CIDR、IP-CIDR6 等）统一放到域名规则之后 color:green %}，如配置中将所有的 `GEOIP` 规则都放到了最后边，同时针对需要直接指定 IP 访问的（配置示例中为 `1.2.3.4`）规则添加 `no-resolve` 来避免多余的 DNS 解析。

# 总结

通过上述的配置与解析，相信你已经对虚空终端的处理流程了如指掌，如果配置过程中遇到什么问题或者与文章不符的情况欢迎评论。
