---
title: 利用 Travis CI 自动测试、部署 Java Web 项目到 Tomcat
copyright: true
date: 2018-05-27 14:52:15 +0800
categories: [Technology]
tags: [java, ci]
---

![Travis CI](/img/tech/travis.png "Travis CI")

<!-- more -->

上一个项目的部署测试流程是：本地写完，本地测试，打包为 war 上传到服务器，服务器部署到 Tomcat 指定目录下，重启 Tomcat。这一套流程下来少说十分钟，而且如果刚上传完发现有 bug 的话，还要本地改完重新再来一遍。

重复这样的过程让人心神俱疲，好在现在已经有成熟的解决方案如 Travis CI、Jenkins等，今天我们就尝试着在 Java Web 项目中运用 Travis CI 来自动测试、部署、重启。

Travis CI 是软件开发领域一个在线的、分布式的持续集成服务，它与 GitHub 的协作相当紧密，并且对开源项目免费。

## 注册配置 Travis

到 Travis 的官网 [https://travis-ci.org](https://travis-ci.org) 注册登录，其实是用 GitHub 做第三方授权登录，十分简单。

![Travis Register](/img/tech/travis-register.png "Travis Registeration Page")

注册成功后进入个人 Profile 页，选择一个需要集成 Travis CI 的项目，开启 Build 即可。

## 创建项目

本文我们以一个简单的 Spring Boot 项目为例：

### 项目目录树

```
│  .gitignore
│  .travis.yml
│  LICENSE
│  README.md
│
└─Hello
    │  .gitignore
    │  HelloTravisCI.iml
    │  mvnw
    │  mvnw.cmd
    │  pom.xml
    │
    ├─.idea
    │  └─....
    │
    ├─.mvn
    │  └─wrapper
    │          maven-wrapper.jar
    │          maven-wrapper.properties
    │
    ├─src
    │  ├─main
    │  │  ├─java
    │  │  │  └─me
    │  │  │      └─xlui
    │  │  │          └─spring
    │  │  │                  Application.java
    │  │  │                  HelloController.java
    │  │  │
    │  │  └─resources
    │  │      │  application.properties
    │  │      │
    │  │      ├─static
    │  │      └─templates
    │  └─test
    │      └─java
    │          └─me
    │              └─xlui
    │                  └─spring
    │                          ApplicationTests.java
    │
    └─target
        └─....
```

### HelloController

```java
package me.xlui.spring;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {
    @RequestMapping(value = "/", method = RequestMethod.GET)
    public String index() {
        return "<html>" +
                "<head><title>Test Page</title></head>" +
                "<body><div align=\"center\">Hello World!</div><br><br><div align=\"center\">This website shows you have successfully integrated <b>Travis-CI</b></div>" +
                "</body></html>";
    }
}
```

### 测试

```java
package me.xlui.spring;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
@SpringBootTest
public class ApplicationTests {

    @Test
    public void contextLoads() {
        System.out.println("This is a simple test, and you pass it.");
    }

}
```

## 添加 .travis.yml

Travis 需要根据项目中 `.travis.yml` 文件中的配置信息来执行相应的动作。

```yml
language: java
jdk:
 - openjdk8
install: cd Hello && mvn install -DskipTests=true -Dmaven.javadoc.skip=true
script: mvn test
```

注意 **install** 的一行，根据目录树，我们的项目是在 Hello 目录下的，所以 install 的时候需要先切换到 Hello 目录中，否则 Maven 会找不到 `pom.xml` 进而构建失败。

## 触发构建

提交并 push 到 GitHub 后，Travis就会自动构建这个 Maven 工程，可以在 Travis 上看到构建结果和过程中的详细输出：

![Travis Build](/img/tech/travis-build.png "Travis Build")

## 自动部署

现在 Travis 已经可以根据我们的提交自动执行构建过程了，下一步就是部署到远程服务器。 Travis 提供 `after_success` 来实现这个步骤。

不过在此之前有一件事需要处理，因为我们要部署到服务器，势必需要 Travis 登录远程服务器，因为是开源项目，我们应该怎么解决登录密码问题？

### 加密登录密码

Travis Docs 提供了这个问题的[解决方案](https://docs.travis-ci.com/user/encrypting-files/)，我们一起来实践一下：

**以下操作完全是在本地！！！**

首先通过 ruby 的 gem 安装 travis：

```bash
# 安装 ruby
sudo yum install ruby ruby-devel
# 更新 gem
sudo gem update --system
# 添加 ruby-china 源
sudo gem sources --add https://gems.ruby-china.org/
# 安装 travis
sudo gem install travis
```

接下来在命令行登录 travis：

```bash
$ travis login
We need your GitHub login to identify you.
This information will not be sent to Travis CI, only to api.github.com.
The password will not be displayed.

Try running with --github-token or --auto if you don't want to enter your password anyway.

Username: xlui
Password for xlui: *********************
Two-factor authentication code for xlui: ******
Successfully logged in as xlui!
```

输入 GitHub 用户名、密码、两步认证码（如果开启了的话）。

将目录切换到项目根目录，因为我们要让 travis 远程登录自己的服务器，所以需要将本地保存的 SSH 私钥进行加密处理。（默认大家都是通过 ssh 登录服务器）

```bash
$ cd test-travis-ci/
$ travis encrypt-file ~/.ssh/id_rsa --add
Detected repository as xlui/test-travis-ci, is this correct? |yes| yes
encrypting ~/.ssh/id_rsa for xlui/test-travis-ci
storing result as id_rsa.enc
storing secure env variables for decryption

Make sure to add id_rsa.enc to the git repository.
Make sure not to add ~/.ssh/id_rsa to the git repository.
Commit all changes to your .travis.yml.
```

这个时候查看 `.travis.yml` 会发现多出了几行：

```yml
before_install:
  - openssl aes-256-cbc -K $encrypted_9655a05d2431_key -iv $encrypted_9655a05d2431_iv
  -in id_rsa.enc -out ~\/.ssh/id_rsa -d
```

为了保证权限正常，我们多加一行设置：

```yml
before_install:
  - openssl aes-256-cbc -K $encrypted_9655a05d2431_key -iv $encrypted_9655a05d2431_iv
  -in id_rsa.enc -out ~/.ssh/id_rsa -d
  - chmod 600 ~/.ssh/id_rsa
```

同时，因为 travis 第一次登录远程服务器的时候会出现 SSH 主机验证，我们无法控制交互，所以要添加 addons 配置：

```yml
addons:
  ssh_known_hosts: your_ip_addr:port
```

注意，如果你的服务器 ssh 端口是开在 `22` 的（**强烈不推荐**），上面的值可以只是 `your_ip_addr`。如果不是开在 22 端口，就要按照上面的格式自己填充。

这样配置完成后，travis 就可以免密登录自己的服务器了。

### 部署脚本

既然已经可以免密登录服务器了，那么我们写一个部署脚本，在登录的时候自动执行即可。

```bash
echo "Auto Deploy Success" >> a.log
```

记得要给脚本添加执行权限（x）。

### 执行部署脚本

在 `.travis.yml` 中添加：

```yml
after_success:
  - ssh your-username@your_ip_addr -p your_port "./your_shell-script.sh"
```

将其中的 `your-username`, `your_id_addr`, `your_port`, `your_shell_script` 替换成自己的即可。

## 自动部署项目到 Tomcat

上面的设置完成后，我们已经可以让 travis 在测试完成后自动执行服务器上的脚本了，如果想自动部署到 Tomcat 我们只需要自己编写脚本即可。因为各自使用的 Tomcat 配置都有差别，就不再举例。

## 在项目中添加 badge

点击 Travis 网站中 `buid` badge，选择 Markdown，将代码复制到项目 README 中即可看到 `build:passing` 的Badge。

![Travis Badge](/img/tech/travis-badge.png "Travis Badge")
