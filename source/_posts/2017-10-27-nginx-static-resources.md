---
title: Nginx 自定义静态资源位置
copyright: true
date: 2017-10-27 20:43:18 +0800
categories: [Nginx]
tags: [nginx, linux]
---

> nginx [Engine X] is an HTTP and reverse proxy server, a mail proxy server, and a generic TCP/UDP proxy server, originally written by [Igor Sysoev](http://sysoev.ru/en/).

<!-- more -->

## 起因

项目后端之前变更为了 Flask 框架 + RESTful API 模式，然后通过 Gunicorn + Supervisor + Nginx 部署在 CentOS 7 服务器上。最近客户端需要访问服务器提供的静态资源（图片），我在 Flask 中配置了静态资源后客户端却无法访问。

项目结构：

```
├─app
│  ├─api_dev
│  ├─api_error
│  ├─api_stable
│  ├─static
│  │  └─images
│  └─templates
├─migrations
└─test
```

在 `app/static/images` 文件夹下有一个 `test.png` 测试图片。在 Win10 本地运行项目，访问 `http://127.0.0.1:5000/static/images/test.png` 可以正常看到测试图片，但是当部署到服务器后，无法正常访问图片。

## 排查原因

在 Win10 和手机浏览器访问 `https://域名.com/static/images/test.png` 都显示 `404 Not Found`。

在云服务器使用命令：

```bash
curl 127.0.0.1:5000/static/images/test.png
```

发现结果是乱码：

![curl](/img/tech/server_curl_test.png "curl")

证明成功访问到了图片资源。

以上测试说明，服务器端代码没有问题。那么问题一定出在 Gunicorn + Supervisor + Nginx 其中的一环，其他两个组件不涉及 URL 路由，那就一定是 Nginx 配置的问题了。

## Nginx 配置

由于我是用的是 lnmp 一键安装包，所以 Nginx 配置是自动生成的，检查配置文件 `/use/local/nginx/conf/vhost/域名.com.conf` 发现并没有对静态资源做相应的配置。我们之前出错的页面显示的是 `404 Not Found` 说明 Nginx 并没有找到我们的静态文件。

因为使用了 vhost，所以配置文件中默认的 root 地址并不是我们项目的地址，我首先尝试修改 root 值为 `/path/to/project/root/app` 但是没有解决问题。

因为是 Flask 项目，使用 Nginx 做反向代理，所以我尝试这在 proxy_pass 中配置了静态资源的路径，依然不行。

## 解决问题

通过 Google 发现，需要自己配置静态文件的路径。在参考了 StackOverflow 上部分配置文件的内容后终于成功了！

```nginx
location ^~ /static {
    root /path/to/project/root/app/;
    expires 30d;
}
```

还记得我的目录树吗？不记得了可以往上翻。请记住，如果你这样写，root 的路径一定是 static 目录的上层目录。

## 总结

出现问题，一步一步排查缩小可能出现问题的范围，发现问题原因，尝试手动解决，谷歌、StackOverflow、官方文档，解决问题。按照着这个步骤走下去，大部分 Linux 上的问题都可以成功解决的吧！
