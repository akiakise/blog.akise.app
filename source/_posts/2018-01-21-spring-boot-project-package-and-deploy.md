---
title: Spring Boot 项目打包并部署到 Tomcat、Tomcat 同时部署多应用
copyright: true
date: 2018-01-21 21:02:42 +0800
categories: [Java]
tags: [java, spring boot, deployment]
---

Spring Boot 项目开发完毕后，需要部署到 tomcat 服务器下，鉴于经常忘记部署流程，特地写了一篇博客来记录。

<!-- more -->

# 打包为 war

## 修改 packaging

基于 Intellij IDEA 构建项目有一个好处是大多数东西它已经自动帮你设置好了，不需要太多修改的地方。

修改 `pom.xml` 中的打包格式：

`<packaging>jar</packaging>` --> `<packaging>war</packaging>`

## 插件与组件

有的博客中提到了 build 组件和 tomcat 插件（`pom.xml` 中)，在 Intellij IDEA 生成的 pom 中并没有这些东西，所以可以直接跳过。

## 注册启动类

修改 Application 类，继承 `SpringBootServletInitializer` 并重写 `configure` 方法，在方法中注册启动类：

```java
@SpringBootApplication
public class Application extends SpringBootServletInitializer {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        return builder.sources(Application.class);
    }
}
```

## 打包

选择 Intellij IDEA 的 Build -> Build Artifacts -> ProjectName: war -> Build，就会在项目根目录的 target 文件夹下生成：项目名+版本号.war。

# Tomcat 同时部署多应用

有时候受限于服务器资源，我们可能希望 Tomcat 同时运行多个应用。有两种解决方案：一种是单一 tomcat，通过配置文件同时服务多个应用；另一种是多个 tomcat，各个应用互不影响，但是比较麻烦。我们采用第一种方案。

Tomcat 默认的配置文件在 `/path/to/tomcat/conf/server.xml`

单一 tomcat 运行多应用的关键就是在 server.xml 中配置多个 Service。

## 默认的 Tomcat Service

```xml
<Service name="Catalina">
    <Connector port="8080" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443" />
    <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
    <Engine name="Catalina" defaultHost="localhost">
        <Realm className="org.apache.catalina.realm.LockOutRealm">
            <Realm className="org.apache.catalina.realm.UserDatabaseRealm" resourceName="UserDatabase"/>
        </Realm>
        <Host name="localhost"  appBase="webapps" unpackWARs="true" autoDeploy="true">
            <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs" prefix="java_dx_style_log" suffix=".txt" pattern="%h %l %u %t &quot;%r&quot; %s %b" />
        </Host>
    </Engine>
</Service>
```

要让 Tomcat 同时运行多应用，我们只需要新增 Service。

## 新增 Service

新增 Service 有几点需要注意的：

- Service name 不能与原来的 `Catalina` 相同
- HTTP port 和 AJP port 不能与原来的相同
- Engine name 不能与原来的相同
- Host 的 appBase 属性不能与原来的相同

以下是一个示例：

```xml
<Service name="newService">
    <Connector port="8081" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443" />
    <Connector port="8010" protocol="AJP/1.3" redirectPort="8443" />
    <Engine name="newService" defaultHost="localhost">
        <Realm className="org.apache.catalina.realm.LockOutRealm">
            <Realm className="org.apache.catalina.realm.UserDatabaseRealm" resourceName="UserDatabase" />
        </Realm>
        <Host name="localhost" appBase="newService" unpackWARs="true" autoDeploy="true">
            <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs" prefix="eros_dx_style_log" suffix=".txt" pattern="%h %l %u %t &quot;%r&quot; %s %b" />
        </Host>
    </Engine>
</Service>
```

修改完成后重启 Tomcat，部署新应用到上面配置文件指示的 `/path/to/tomcat/newService` 文件夹下即可。
