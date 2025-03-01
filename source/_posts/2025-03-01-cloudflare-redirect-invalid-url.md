---
title: 利用 Cloudflare 重定向规则自动处理非法 URL
tags: [Cloudflare]
categories: Technology
date: 2025-03-01 19:13:18
description: 网上分享网站链接时偶尔会漏加空格，这时多数平台都会将原始链接+跟着的内容识别为新链接，用户点进来是 404 非常尴尬，我们借助 Cloudflare 的重定向规则来解决此问题
cover: /img/2025/cloudflare-redirect-invalid-url/cover.png
banner: /img/2025/cloudflare-redirect-invalid-url/cover.png
topic: play
---

在前文讲了如何使用 Cloudflare 重定向规则实现近似“无端口”访问带 IP 的服务（{% post_link cloudflare-redirect-rule %}）后，我们继续利用重定向规则来解决另一个常见的链接错误的问题。

# 问题

当我们在社交/论坛分享链接时，如果没有在链接后加上空格，那么链接后边的内容有很大的概率就会错误地被拼接上来，例如我们分享本文链接时结尾没加空格拼上了 `，测试test`：

{% link https://blog.akise.app/posts/cloudflare-redirect-invalid-url/，测试test 利用 Cloudflare 重定向规则自动处理非法 URL %}

点击这个链接就会报 404 错误。

# 解决方案

## Hexo ❌

Hexo 本身是一个静态博客，本站托管在 GitHub Pages 上，因此我们没有任何动态的手段来识别错误 URL 并进行处理。

## Cloudflare ✔

Cloudflare 支持针对请求的 URL 做重定向规则，看起来可以满足我们的诉求。那么首先我们明确下需要实现的功能：**我们需要在博客链接包含错误字符时将其重定向到正常的链接**。

假如错误的链接为：{% note https://blog.akise.app/posts/cloudflare-redirect-invalid-url/，测试test %}

我们要将其重定向为：{% note https://blog.akise.app/posts/cloudflare-redirect-invalid-url/ %}

在 Cloudflare 添加如下的重定向配置：

{% image /img/2025/cloudflare-redirect-invalid-url/config.png 重定向配置 %}

参数说明：

- 如果传入请求匹配...：自定义筛选表达式
- URI 完整：通配符匹配 `https://blog.akise.app/posts/*/*`，精确匹配博客 URL 格式，其中第一个 `*` 匹配文档，第二个 `*` 匹配多余的字符
- URI 路径：结尾不是 `/`，避免类似于 `https://blog.akise.app/posts/cloudflare-redirect-invalid-url/` 的链接无限循环重定向
- URL 重定向：动态重定向，表达式 `wildcard_replace(http.request.uri.path, r"/posts/*/*", r"/posts/${1}/")`，状态代码 `301`

### URL 匹配

通过通配符和排除规则，针对如下 URL 会走到不同的处理逻辑：

- `https://blog.akise.app/posts/cloudflare-redirect-invalid-url/`: 由于结尾是 `/` 跳过 Cloudflare 处理
- `https://blog.akise.app/posts/cloudflare-redirect-invalid-url/，测试test`: 匹配成功，需要重定向
- `https://blog.akise.app/posts/cloudflare-redirect-invalid-url`: 由于缺少结尾的 `/` 跳过 Cloudflare 处理，Hexo 会自动在结尾加一个 `/`

### 重定向

重定向规则使用 `wildcard_replace`，它会匹配第一个参数，并将其中的 `*` 编号替换到第二个参数中的 `${数字}`，在 `wildcard_replace(http.request.uri.path, r"/posts/*/*", r"/posts/${1}/")` 规则下针对如下入参：

{% note https://blog.akise.app/posts/cloudflare-redirect-invalid-url/，测试test %}

它会返回：

{% note https://blog.akise.app/posts/cloudflare-redirect-invalid-url/ %}

完美满足我们的诉求。

### 实验

下面我们拼上其他字符进行试验：

{% link https://blog.akise.app/posts/cloudflare-redirect-invalid-url/，测试 拼上了“，测试”的URL %}

点击上述 URL 就不会再报 404 了，完美。

# 结语

可以看到，Cloudflare 重定向规则还是很强大的，通过简单的配置就让我们对域名的访问流程有了较大的定制化能力，Cloudflare 还在测试一项名为 [Snippets](https://developers.cloudflare.com/rules/snippets/) 来直接通过 JavaScript 代码干涉访问流程，这样会更加简单便捷，不过目前仅限付费用户可用。

{% folding 附言 %}
为了实现本文中部分 URL 访问 404、其他 URL 访问重定向的效果，我还在规则中另外增加了“URL 结尾不是 test”的规则 😊
{% endfolding %}
