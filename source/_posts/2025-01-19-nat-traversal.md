---
title: 内网穿透
description: 详细描述多种内网穿透方案的实现细节，包括基于 FRP 的内网穿透、基于 DDNS 的公网 IPv4 和 IPv6 穿透。
date: 2025-01-19 12:59:28
categories: [Technology]
tags:
  - NAS
  - FRP
---

# 前言

内网穿透的概念不再赘述，通常我们是希望能够在外出、旅行等不在家情况下也能访问家庭网络（例如查看 NAS 中的照片视频、远程连接内网设备进行办公等），而基于家庭网络环境，内网穿透又有几种不同的方案：
1. 无公网 IP 无云服务器：Tailscale、ZeroTier 等虚拟组网工具
2. 无公网 IP 有云服务器：FRP
3. 有公网 IPv4：DDNS
4. 有公网 IPv6：DDNS

通过 Tailscale 或 ZeroTier 等组网工具配合路由器可以实现较为无感的内网穿透，但是这些工具在打洞失败时会走中转服务器，而中转服务器到国内的延迟往往是无法接受的（虽说可以自行部署中转服务器，但是这样还不知直接用 FRP）。我个人此前用过 ZeroTier，配置起来非常简单，但是在客户端使用时会占用 VPN，无法跟其他 VPN 并存，因此本文不再介绍这部分。

本文以我的家庭网络环境为例，有一些需要联系运营商开启（例如光猫桥接），有一些需要自身硬件支持（例如路由器支持特定配置功能），可能在读者自身的网络环境中无法完美复刻，但是管中窥豹已经足够，读者可以结合自身的网络环境选取合适的方案。

# 概览

多数内网穿透都是直接将内网服务穿透后暴露至公网，虽说内网服务也会有各自的密码，但是总会有未知的安全漏洞、我们也有可能误将无密码/弱密码服务暴露出去，最好还是在内网服务前加一层 VPN。本文中的所有方案我们都采用 shadowsocks 作为内网服务的入口，预检所有流量，同时也实现流量的端到端加密：

![NAT Traversal with VPN](/img/2025/nat-traversal/with-vpn.png)

# FRP

针对无公网 IP 有云服务器的情况，我们通过 FRP 来实现内网穿透：

![FRP](/img/2025/nat-traversal/frp.png)

## ss-server 配置

增加配置文件：

```bash
mkdir -p /docker/config/ss
vim /docker/config/ss/config.json
```

编辑配置文件内容：

```json
{
    "server":"0.0.0.0",
    "server_port":12300,
    "password":"lalala",
    "timeout":300,
    "method":"chacha20-ietf-poly1305"
}
```

使用 docker 方便管理服务：

```bash
docker run --name ss -p 12300:12300 -v /docker/config/ss:/etc/shadowsocks-rust -d teddysun/shadowsocks-rust
```

## 中转服务器配置

假设中转服务器 IP 为 `1.2.3.4`，首先我们通过脚本安装 frp：

```bash
wget https://raw.githubusercontent.com/mvscode/frps-onekey/master/install-frps.sh -O ./install-frps.sh
chmod +x ./install-frps.sh
./install-frps.sh install
```

跟着提示安装即可，期间脚本会生成 frps 的 token，这个我们需要记录下。假设 frps 的端口为 `12500`，token 为 `YOURFRPSTOKEN`。

配置 systemd 管理 frps 服务：

```bash
vim /etc/systemd/system/frps.service
```

添入以下内容

```conf
[Unit]
Description=frps daemon

[Service]
Type=simple
ExecStart=/usr/local/frps/frps -c /usr/local/frps/frps.toml

[Install]
WantedBy=multi-user.target
```

加载 systemd 服务并启动：

```bash
systemctl daemon-reload
systemctl start frps
systemctl enable frps
```

检查服务运行状态：

```bash
systemctl status frps
```

现在 frps 会随着机器启动自动运行了。

## 内网配置

新增配置文件：

```bash
mkdir -p /docker/config/frpc
vim /docker/config/frpc/frpc.toml
```

配置文件内容：

```toml
serverAddr = "1.2.3.4"
serverPort = 12500
auth.token = "YOURFRPSTOKEN" # 替换为中转服务器的 token

[[proxies]]
name = "shadowsocks"
type = "tcp"
localPort = 12300
remotePort = 12400
```

使用 docker 方便管理服务：

```bash
docker run --name frpc --net host -v /docker/config/frpc/frpc.toml:/frp/frpc.toml -d stilleshan/frpc
```

这样，我们内网启动了 frpc，它会通过端口 `12500` 和 token `YOURFRPSTOKEN` 连接到公网中转服务器 `1.2.3.4`，告诉公网服务器监听自己 `12400` 端口的流量，并将发往这个端口的流量转发至 frpc 所在机器的 12300 端口。前文我们在 12300 端口部署了 shadowsocks，于是流量会经过 shaodowsocks 后进入内网。 

## 客户端配置

客户端通过指定 IP 规则来实现透明代理，以 Quantumult X 配置举例：

```conf
[server_local]
shadowsocks=1.2.3.4:12400, method=chacha20-ietf-poly1305, password=lalala, fast-open=false, udp-relay=false, tag=ss

[filter_local]
# 我的内网网段为 192.168.10.x 
ip-cidr, 192.168.10.0/24, ss
ip-cidr, 192.168.0.0/16, direct
```

这样，客户端在访问 192.168.10.x 时流量就会通过 ss-client 加密后传递给中转服务器，并经由 FRP 传递给内网 ss-server，ss-server 解密后再传递给内网服务。

## 优缺点

FRP 方案的优点是不需要公网 IP，任何网络环境都可以部署，缺点是需要一台云服务器用于流量中转，同时该方案的网络带宽也受限于云服务器和家庭网络的上传上限，即 `MAX(带宽) = MIN(云服务器上传, 家宽上传)`。

# DDNS + IPv4

有公网 IP 的情况下，通过 DDNS 实现内网穿透最为简单：

![DDNS](/img/2025/nat-traversal/ddns.png)

## ss-server 配置

增加配置文件：

```bash
mkdir -p /docker/config/ss
vim /docker/config/ss/config.json
```

编辑配置文件内容：

```json
{
    "server":"0.0.0.0",
    "server_port":12300,
    "password":"lalala",
    "timeout":300,
    "method":"chacha20-ietf-poly1305"
}
```

使用 docker 方便管理服务：

```bash
docker run --name ss -p 12300:12300 -v /docker/config/ss:/etc/shadowsocks-rust -d teddysun/shadowsocks-rust
```

## DDNS 脚本

在内网服务（NAS、路由器）上新增 DDNS 脚本（也可以用现有的 DDNS 组件，原理是一样的）：

```bash
#!/bin/sh
ipv4=$(curl -sS ipv4.ip.sb)
echo '['`date +"%Y-%m-%d %H:%M:%S"`']' "current ipv4: $ipv4"
config_json=$(curl -sS -X GET "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/dns_records/<RECORD_ID>" -H "X-Auth-Email: <YOUR_EMAIL>" -H "X-Auth-Key: <YOUR_AUTH_KEY>" -H "Content-Type: application/json")
config_ipv4=$(echo $config_json | sed 's/.*content":"\(.*\)","proxiable.*/\1/g')
if [[ -z $config_ipv4 ]];
then
    echo '['`date +"%Y-%m-%d %H:%M:%S"`']' "current ipv4 is empty"
    exit 1
fi
echo '['`date +"%Y-%m-%d %H:%M:%S"`']' "config ipv4: $config_ipv4"
if [ "$ipv4" == "$config_ipv4" ]; then
    echo '['`date +"%Y-%m-%d %H:%M:%S"`']' "current ipv4 and config ipv4 equals, skip"
else
    echo '['`date +"%Y-%m-%d %H:%M:%S"`']' $(curl -sS -X PUT "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/dns_records/<RECORD_ID>" \
        -H "X-Auth-Email: <YOUR_EMAIL>" \
        -H "X-Auth-Key: <YOUR_AUTH_KEY>" \
        -H "Content-Type: application/json" \
        -d '{"type":"A","name":"<YOUR_DOMAIN_NAME>","content":"'$ipv4'","ttl":120, "proxied":false}')	
fi
```

脚本中 `<>` 中的都是变量，需要替换下：
- ZONE_ID：在 Cloudflare 域名界面右下角的“区域ID”
- RECORD_ID：替换 ZONE_ID 后访问 `https://dash.cloudflare.com/api/v4/zones/<ZONE_ID>/dns_records?per_page=200&order=type&direction=asc`，找到 name 匹配你域名的记录的 `id` 字段
- YOUR_EMAIL：你的 Cloudflare 账户邮箱
- YOUR_AUTH_KEY：在 Cloudflare 域名界面右下角的“获取您的 API 令牌”，生成一个可以操作 DNS 的 API 令牌，使用“编辑区域 DNS”模板即可
- YOUR_DOMAIN_NAME：你的 DDNS 要绑定的域名

注意，脚本会先比对当前数据是否一致，所以变量中 RECORD_ID 和 YOUR_DOMAIN_NAME 是一一对应的，即 RECORD_ID 应该就是 YOUR_DOMAIN_NAME 对应的 id。

## 流量转发配置

此种方案外网流量直达我们的光猫/路由器（桥接），因此我们需要配置下流量转发将流量路由到提供 ss-server 的机器上：

![DDNS Forward](/img/2025/nat-traversal/ddns-forward.png)

## 客户端配置

客户端通过指定 IP 规则来实现透明代理，以 Quantumult X 配置举例：

```conf
[server_local]
shadowsocks=ddns.domain.com:12300, method=chacha20-ietf-poly1305, password=lalala, fast-open=false, udp-relay=false, tag=ss

[filter_local]
# 我的内网网段为 192.168.10.x 
ip-cidr, 192.168.10.0/24, ss
ip-cidr, 192.168.0.0/16, direct
```

这样，客户端在访问 192.168.10.x 时流量就路由到 ss，客户端通过域名解析出当前的公网 IP，然后 ss-client 将流量加密后直接发往光猫/路由器（桥接）的 12300 端口，光猫/路由器（桥接）再将流量路由到 ss-server，ss-server 将流量解密后路由到内网 192.168.10.x 机器上。

## 优缺点

DDNS 方案的优点是不需要中转，流量直达家宽，能轻松跑到家宽上传上限，同时增加了 ss 作为安全性、流量加密保障，缺点是公网 IP 难求。

# DDNS + IPv6

受限于当前公网 IPv4 越来越难拿到，能给每一粒沙子都分配一个 IP 的 IPv6 成了另一个可行方案：

![DDNS](/img/2025/nat-traversal/ddns.png)

依然是这张图，不过内部用于流量传递的变成了 IPv6。

## ss-server 配置

增加配置文件：

```bash
mkdir -p /docker/config/ss
vim /docker/config/ss/config.json
```

编辑配置文件内容：

```json
{
    "server":"0.0.0.0",
    "server_port":12300,
    "password":"lalala",
    "timeout":300,
    "method":"chacha20-ietf-poly1305"
}
```

使用 docker 方便管理服务：

```bash
docker run --name ss -p 12300:12300 -v /docker/config/ss:/etc/shadowsocks-rust -d teddysun/shadowsocks-rust
```

## DDNS 脚本

在内网服务（NAS、路由器）上新增 DDNS 脚本（也可以用现有的 DDNS 组件，原理是一样的）：

```bash
ipv6=$(curl -sS ipv6.ip.sb)
echo '['`date +"%Y-%m-%d %H:%M:%S"`']' "current ipv6: $ipv6"
config_json_v6=$(curl -sS -X GET "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/dns_records/<RECORD_ID_V6>" -H "X-Auth-Email: <YOUR_EMAIL>" -H "X-Auth-Key: <YOUR_AUTH_KEY>" -H "Content-Type: application/json")
config_ipv6=$(echo $config_json_v6 | sed 's/.*content":"\(.*\)","proxiable.*/\1/g')
if [[ -z $config_ipv6 ]];
then
    echo '['`date +"%Y-%m-%d %H:%M:%S"`']' "current ipv6 is empty"
    exit 1
fi
echo '['`date +"%Y-%m-%d %H:%M:%S"`']' "config ipv6: $config_ipv6"
if [ "$ipv6" == "$config_ipv6" ]; then
    echo '['`date +"%Y-%m-%d %H:%M:%S"`']' "current ipv6 and config ipv6 equals, skip"
else
    echo '['`date +"%Y-%m-%d %H:%M:%S"`']' $(curl -sS -X PUT "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/dns_records/<RECORD_ID_V6>" \
        -H "X-Auth-Email: <YOUR_EMAIL>" \
        -H "X-Auth-Key: <YOUR_AUTH_KEY>" \
        -H "Content-Type: application/json" \
        -d '{"type":"AAAA","name":"<YOUR_DOMAIN_NAME_V6>","content":"'$ipv6'","ttl":120, "proxied":false}')	
fi
```

脚本中 `<>` 中的都是变量，需要替换下：
- ZONE_ID：在 Cloudflare 域名界面右下角的“区域ID”
- RECORD_ID_V6：替换 ZONE_ID 后访问 `https://dash.cloudflare.com/api/v4/zones/<ZONE_ID>/dns_records?per_page=200&order=type&direction=asc`，找到 name 匹配你域名的记录的 `id` 字段
- YOUR_EMAIL：你的 Cloudflare 账户邮箱
- YOUR_AUTH_KEY：在 Cloudflare 域名界面右下角的“获取您的 API 令牌”，生成一个可以操作 DNS 的 API 令牌，使用“编辑区域 DNS”模板即可
- YOUR_DOMAIN_NAME_V6：你的 DDNS 要绑定的域名

注意，脚本会先比对当前数据是否一致，所以变量中 RECORD_ID_V6 和 YOUR_DOMAIN_NAME_V6 是一一对应的，即 RECORD_ID_V6 应该就是 YOUR_DOMAIN_NAME_V6 对应的 id。

## 防火墙放行配置

在 IPv6 情况下，流量转发已经没有必要，因为内网设备分配的也是公网 IP，我们需要做的是放行 ss-server 所在机器的防火墙。但是内网设备分配的公网 IP 也会动态变化，我们应该如何配置防火墙才能在 IP 变化时不需要调整配置呢？可以用 [EUI-64 地址](https://networklessons.com/ipv6/ipv6-eui-64-explained)。

首先找到 ss-server 所在机器的 IPv6 地址，假设为 `2001:2002:2003:2004:****:**ff:fe**:****`，记下 `****:**ff:fe**:****` 部分，使用 EUI-64 `<需要暴露的主机的后缀>/::ffff:ffff:ffff:ffff`。

假设我们的 IPv6 地址为 `2001:2002:2003:2004:1:20ff:fe25:2025`，则对应 EUI-64 地址为 `1:20ff:fe25:2025/::ffff:ffff:ffff:ffff`，将这个地址填入防火墙放行即可：

![IPv6 Firewall Allow](/img/2025/nat-traversal/ipv6-firewall-allow.png)

注意：**千万不要关闭 IPv6 防火墙，除非你明确知道这有什么后果！！！！！**

## 客户端配置

客户端通过指定 IP 规则来实现透明代理，以 Quantumult X 配置举例：

```conf
[server_local]
shadowsocks=ddnsv6.domain.com:12300, method=chacha20-ietf-poly1305, password=lalala, fast-open=false, udp-relay=false, tag=ss

[filter_local]
# 我的内网网段为 192.168.10.x 
ip-cidr, 192.168.10.0/24, ss
ip-cidr, 192.168.0.0/16, direct
```

这样，客户端在访问 192.168.10.x 时流量就路由到 ss，客户端通过域名解析出当前的公网 IPv6，然后 ss-client 将流量加密后直接发往光猫/路由器（桥接）的 12300 端口，光猫/路由器（桥接）再将流量路由到 ss-server，ss-server 将流量解密后路由到内网 192.168.10.x 机器上。

注意，代理软件不能关闭 IPv6，否则域名解析不出来结果。

## 优缺点

IPv6 方案的优点与 IPv4 相同，都是流量直达家宽，并且 IPv6 不需要额外向运营商申请公网 IPv4，缺点是国内的 IPv6 网络质量堪忧。

# 狡兔三窟

分布式系统中常说异地多活、容灾等概念，我们内网穿透也可以三种方案并存，以备不时之需，每种方案的配置细节与原理都写在上边了，任君组合 ~

# 我的网络环境

我所在的地区是可以下发动态公网 IP 的，所以我主要使用 DDNS + IPv4 的形式，不过我也配置了另外两种方案作为备份，以下是我的家庭网络拓扑：

![My NAT Traversal](/img/2025/nat-traversal/my.png)

# 总结

本文我们介绍了内网穿透的几种方案，在各式各样工具的加持下，内网穿透现在已经不再是一个难题了。不过网络传输的稳定性依然是一个大问题，家宽上传更是万年的 30M，不过技术发展日新月异，这些问题迟早会被解决，期待那一天的到来！
