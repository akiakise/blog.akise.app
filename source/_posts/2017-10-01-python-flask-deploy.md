---
title: Centos7 系统使用 Gunicorn、Supervisor、Nginx 部署使用了工厂模式的 Flask 项目
copyright: true
date: 2017-10-01 14:21:30
categories: [Python]
tags: [python, flask, deployment]
---

项目后端从原始 socket 模式切换到了 RESTful API，考虑到项目的复杂度不高，于是我决定采用 Flask 来实现，本文记录一下基于 Gunicorn、Supervisor 和 Nginx 的最终的部署过程。

<!-- more -->

我们使用一个简单的 Flask Demo 来跑通整个流程。

## Flask

项目结构：

```
|-REST-Server
  |-app
    |-__init__.py
    |-api_0_1_0
      |-__init__.py
      |-views.py
  |-config.py
  |-manage.py
  |-wsgi.py
```

> 考虑到版本向下兼容，同时为了测试方便，使用不同的目录区分不同的 API 版本。在 `app/__init__.py` 中的 `create_app()` 函数中注册所有受支持的 api 蓝本（依据不同的 api 版本，在注册的时候设置不同的前缀，本例设置的前缀是：`/api/v0.1.0`），这样就做到了同时支持新旧功能。

`app` 文件夹是项目主体
`manage.py` 主要是本地测试使用
`config.py` 是配置文件
`wsgi.py` 是`部署文件`，其内容是：

```python
from app import create_app
app = create_app()

if __name__ == '__main__':
    app.run()
```

## Gunicorn

Gunicorn (独角兽)是一个高效的 Python WSGI Server，通常用它来运行 WSGI 应用（由我们编写的符合 WSGI 标准的后端服务）或者 WSGI 框架(如 Django，Paster 等)，其地位相当于 Java 中的 Tomcat。

使用 pip 安装：

```bash
pip install gunicorn
```

在项目 root 目录下（有 `wsgi.py` 的那个目录）运行：

```bash
gunicorn -w 2 -b 127.0.0.1:5000 wsgi:app
```

- -w 后接工作线程数
- -b 后接绑定地址，即本机访问地址
- wsgi 即第一步中的部署文件
- app 是部署文件中的全局变量 app

运行后，在本地用 `curl localhost:5000/api/v0.1.0`（该命令受你的配置影响。`/api/v0.1.0`是我注册蓝本时添加的前缀，如果你注册时并没有添加，请不要使用此链接）即可成功访问。

## Supervisor

利用 yum 可以直接安装 Supervisor，需要注意 Supervisor 是直接运行在系统中的，所以**不要在虚拟环境中安装**。 而且 Supervisor 也只支持 Python2。

```bash
yum install supervisor
```

创建配置文件：`/etc/supervisord.d/server.ini`

```conf
[program:server]
directory=/项目目录
command=/虚拟环境目录/bin/gunicorn -w 2 -b 127.0.0.1:5000 wsgi:app
autostart=true
autorestart=true
user=用户
startsecs=3
startretries=5
redirect_stderr=true
stdout_logfile_maxbytes=50MB
stdout_logfile_backups=10
stdout_logfile=/opt/server.log
```

修改 `/etc/supervisord.conf` 将配置文件包含进来：

```conf
[include]
files = supervisord.d/*.ini
```

在我写这篇文章的时候，通过 yum 安装的 Supervisor 的配置文件已经默认是上边的格式，所以你只需要在 `/etc/supervisord.d/` 下创建你的 `程序.ini` 即可。

配置完成后启动：

```bash
supervisord -c /etc/supervisord.conf
```

## Nginx

关于正向代理和反向代理的概念只是不同位置请求网络资源的不同说法，你用笔记本上网浏览网页通过的一个代理就叫正向代理，如果互联网过来的一个请求访问一个 IDC 机房的一个内网 web 服务，这个服务由 nginx 转进来就叫做反向代理，nginx 一般用于反向代理和负载均衡，这个 IDC 机房的 web 服务也可以直接用公网IP访问，但是为了解决高并发负载均衡的问题引入了 nginx 反向代理这个概念。

一个服务器只能对外暴露一个 80 端口，而我们有多个服务同时在跑，因此我们就需要借助 Nginx 反向代理，复用 80 端口，由 Nginx 完成 [客户端 -> 域名 -> 服务器 -> 具体服务] 中，服务器接收到请求后路由给特定服务的工作。

```
server {
    listen 80 default_server;
    server_name _;
    #index index.html index.htm index.php;

    # 重要的是这部分
    location / {
        proxy_pass      http://127.0.0.1:5000/;
        proxy_redirect  off;

        proxy_set_header    Host                $host;
        proxy_set_header    X-Real-IP           $remote_addr;
        proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Proto   $scheme;
    }

    location ~ .*\.(gif|jpg|jpeg|png|bmp|swf)$
    {
        expires      30d;
    }

    location ~ .*\.(js|css)?$
    {
        expires      12h;
    }

    location ~ /.well-known {
        allow all;
    }

    location ~ /\.
    {
        deny all;
    }

    access_log  /home/wwwlogs/access.log;
}
```

用 `nginx -t` 检查配置无误后重启 nginx。

## 验证

打开本地浏览器，访问 `ip/api/v0.1.0`（后面的 `/api/v0.1.0` 基于你自己的配置，如果你在注册蓝本时没有设置前缀，请不要添加），成功看到 `Hello World`。
