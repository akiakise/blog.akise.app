---
title: Maven 中的 Scope
copyright: true
date: 2018-10-27 14:33:31 +0800
categories: [Technology]
tags: [maven, java]
---

Maven 中 scope 的取值一共有 `compile`、`test`、`runtime`、`provided`、`system` 这几种，其实没有必要专门写一篇博客来讲。但是最近遇到了一个相关的坑，其实也不能说是坑，主要原因是自己对这些东西一知半解，所以还是有必要深究一下。

<!-- more -->

## compile

`compile` 是默认的 scope，即不显式声明条件下 scope 就是 `compile`。`compile` 表示该依赖需要参与项目编译、测试、运行等所有步骤中，是最强的依赖，打包时通常需要包含进去。

## test

`test` 表示依赖项目仅仅参与测试相关的工作，包含测试代码的编译、执行，但是原项目的编译、运行均不参与其中。

## runtime

`runtime` 表示依赖不会参与项目的编译，不过会参与到后续的测试与运行环节中。与 `compile` 相比只是少了编译阶段，那为什么要提供这个 scope 呢？

StackOverflow 上有一个经典回答：[https://stackoverflow.com/a/12273576/7944150](https://stackoverflow.com/a/12273576/7944150)。简单翻译下，声明 scope 为 `runtime` 的依赖通常是需要动态加载的代码，比如 JDBC 驱动，或者其他运行时通过 Java 反射调用加载的类。将这种依赖设置为 `runtime` 可以避免代码中意外的依赖，同时避免依赖传递。

**注意，原回答中关于传递依赖的说法是错误的**，参考 [Dependency Mechanism](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html)，本文下方也会给出这个传递依赖表。

## provided

`provided` 表明打包时可以不将依赖打包进去，JDK 或者容器会提供该依赖，典型的就是 servlet-api.jar，Tomcat 容器或者其他容器一般会提供这个 jar，所以我们就不需要将其声明为 `compile`。后一种做法还可能导致依赖冲突。

## system

`system` 表明该依赖存在于本地系统，需要提供 JAR 路径，比较少用。

## 依赖传递

存在 A -> B -> C，项目 A 依赖于项目 B，B 又依赖于 C。当知道 B 在 A 中的 scope 时，如何确定 C 的 scope？

1. C 在 B 中 scope 为 `test` 或 `provided` 时，C 直接丢弃，A 不依赖于 C
1. 否则 A 依赖于 C，C 在 A 中的 scope 由 C 在 B 中的 scope 决定

| B在A中的scope \ C在B中的scope | compile | provided | runtime | test |
| - | - | - | - | - |
| compile | compile | - | **runtime** | - |
| provided | provided | provided | provided | - |
| runtime | runtime | - | runtime | - |
| test | test | - | test | - |

表格中交叉部分的单元格即为 C 在 A 中的 scope，单元格取值为 “-” 则代表 C 不能被 A 传递依赖。

## 遇到的问题

我最近在学 Netty，编解码部分用到了 JBoss Marshalling，这个编解码框架有一个通用的 API 以及有几个不同的实现：

通用 API 为：`org.jboss.marshalling >> jboss-marshalling`，
可选的实现有：

- org.jboss.marshalling >> jboss-marshalling-river
- org.jboss.marshalling >> jboss-marshalling-osgi
- org.jboss.marshalling >> jboss-marshalling-serial

我的项目用到了 `serial` 版本的实现，但是在添加依赖的时候失误将其 scope 设置成了 `test`，造成一直无法成功编码、解码。定位问题的方法是将编码解码方法单独摘出来调用，发现这一行返回了 null：

```java
final MarshallerFactory factory = Marshalling.getProvidedMarshallerFactory("serial");
```

于是想到 serial 的依赖可能没有成功导入....

在将其依赖重设为 `compile` 之后项目成功跑起来了。在补充了 scope 相关的知识后我认为 `serial` 正确的 scope 应该是 `runtime`，因为这个依赖存在的目的就是通过反射被加载，编译期是不需要的。修改 scope 之后果然不出所料。

所以 `runtime` 存在的意义就是将抽象与实现解耦，我们可以以 `compile` 依赖一个 API，然后通过 `runtime` 依赖其多种不同的实现，根据需要进行使用。

以上为个人理解，如有不符之处，欢迎指出 :-)

## 引申

综合考虑以上几种 scope 的特性，当我们想在 Spring Boot 项目中全局排除某一个依赖时，应该如何处理？

<details>
<summary>查看答案</summary>
答案是在依赖声明时将其直接声明为 <code class="language-plaintext highlighter-rouge">provided</code>，由于 <code class="language-plaintext highlighter-rouge">provided</code> 代表着由 JDK 或者容器提供该依赖，而 Spring Boot 项目本身不需要容器即可运行（或者说它本身已经带了一个容器），所以将依赖设为 <code class="language-plaintext highlighter-rouge">provided</code> 可以确保该依赖不会被编译也不会被打包进最终的 jar/war，从而实现全局排除的效果。
</details>
