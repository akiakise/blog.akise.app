---
title: Nginx 提供文件下载服务
copyright: true
date: 2017-11-30 17:21:54 +0800
categories: [Nginx]
tags: [nginx, linux]
---

有时候我们可能需要提供一些配置文件或安装包的下载链接，这种场景使用 CDN 有些杀鸡用牛刀，通过 Eginx 配置可以简单快速的提供功能。

<!-- more -->

Nginx 配置：

```nginx
server {
    listen 80;
    server_name download.akise.app;
    root /path/to/download.akise.app;

    location / {
        index index.html;
    }

    location /app {
        default_type application/octet-stream;
    }
}
```

文件夹配置：

```bash
mkdir /path/to/download.akise.app/app
touch /path/to/download.akise.app/app/t.apk
```

浏览器访问 `http://download.akise.app/app/t.apk` 即会自动开始下载。
