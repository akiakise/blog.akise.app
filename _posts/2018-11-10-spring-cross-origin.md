---
title: Spring 中的跨域问题
copyright: true
date: 2018-11-10 11:56:08 +0800
categories: [Java]
tags: [java, spring, cross region]
---

项目做前后端分离，遇到了一个很常见的问题：跨域。想着每次遇到都要搜索解决，而且搜到的文章给出的解决方案又千奇百怪，不一定合适，于是萌生了总结一下的想法。

<!-- more -->

## 问题由来

跨站 HTTP 请求（Cross-site HTTP Request）是指发起请求的资源所在的 domain 与该请求所指向的 domain 不同的 HTTP 请求。比如，域名 abc(`www.abc.com`) 的某个标签引用了域名 cde(`www.cde.com`) 的某资源，域名 abc 的 Web 应用就会导致浏览器发起一个跨站 HTTP 请求。

这种方式极大地方便了 Web 开发，然而，出于安全考虑（主要是防范 csrf），浏览器会限制从脚本内发起的跨源 HTTP 请求。例如 `XMLHttpRequest` 和 `Fetch API` 遵循同源策略。这意味着使用这些 API 的 Web 应用程序只能从加载应用程序的同一个域请求 HTTP 资源。

然而如果我们在本地开发前后端分离程序，前端请求本机上的后端 API 却由于浏览器限制而请求失败，十分不便。

### CORS

跨域资源共享（CORS）机制允许 Web 应用服务器进行跨域访问控制，从而使跨域数据传输得以安全进行。现代浏览器支持在 API 容器中（例如 `XMLHttpRequest` 或 `Fetch`）使用 CORS，以降低跨域 HTTP 请求中所带来的风险。

### 功能概述

跨域资源共享标准新增了一组 HTTP 首部字段，允许服务器声明哪些源站有权限通过浏览器访问哪些资源。另外，标准要求，对那些可能对服务器产生副作用的 HTTP 请求方法（特别是 GET 以外的 HTTP 请求，或者搭配某些 MIME 类型的 POST 请求），浏览器必须首先用 `OPTIONS` 方法发起一个**预检请求（Preflight Request，请记住这个概念，后面会经常使用）**，从而获取服务器是否允许该跨域请求。服务器确认允许后，才发起实际的 HTTP 请求。在预检请求的返回中，服务器也可以通知客户端，是否携带身份凭证（包括 Cookie 和 HTTP 认证相关数据）。

CORS 请求失败会产生错误，但为了安全，在 JavaScript 代码层面是无法获知到底哪里出了问题。你只能查看浏览器控制台以得知具体是哪里出现了错误。

### 简单请求

某些请求不会触发 [CORS 预检请求](#预检请求)。若请求满足所有下述条件，则请求可视为“简单请求”：

- 使用下列方法之一
    + GET
    + HEAD
    + POST
- 首部字段在 **对 CORS 安全的首部字段集合** 中：
    + Accept
    + Accept-Language
    + Content-Language
    + Content-Type（下一条为值限制）
    + DPR
    + Downlink
    + Save-Data
    + Viewport-Width
    + Width
- Content-Type 的值为下列三者之一
    + text/plain
    + multipart/form-data
    + application/x-www-form-urlencoded
- 请求中的任意 XMLHttpRequestUpload 对象均没有注册任何事件监听器
- 请求中没有使用 ReadableStream 对象

我们可以看到，简单请求的要求是十分严格的，这篇博客我们不会过多讨论简单请求。

### 预检请求

与上一节的简单请求不同，预检请求要求必须首先使用 `OPTIONS` 方法发起一个预检请求到服务器，以获知服务器是否允许该实际请求。预检请求的使用，可以避免跨域请求对服务器的用户数据产生未预期的影响。

## 模拟跨域问题

### 浏览器端

使用 JavaScript 请求本地 API 即可：

```html
<!DOCTYPE html>
<html>
    <head>
        <title>Web</title>
    </head>
    <body>
        <div>
            <label for="name">Name: </label>
            <input id="name" type="text" value="xlui" autofocus/>
            <input type="button" onclick="postData()" value="Submit">
        </div>
        <br><br><br>
        <div>
            Response:<br>
            <textarea name="response" id="response" cols="30" rows="5"></textarea>
        </div>
        <script>
            function postData() {
                var i = document.getElementById("name").value;
                var resp = document.getElementById("response");
                console.log('try to submit ' + i);
                var xhr = new XMLHttpRequest();
                xhr.open('post', 'http://127.0.0.1:8080/', true)
                xhr.setRequestHeader('content-type', 'application/json')
                xhr.onreadystatechange = function() {
                    if (xhr.readyState === 4) {
                        console.log(xhr.responseText);
                        resp.value = xhr.responseText;
                    }
                }
                xhr.send(JSON.stringify({
                    'username': i
                }))
            }
        </script>
    </body>
</html>
```

### 服务器端

从简单做起，我们首先写一个简单的 Servlet 服务器：

```java
@WebServlet(name = "MainServlet", urlPatterns = "/")
public class MainServlet extends HttpServlet {
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        doGet(request, response);
    }

    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        // 提取用户名
        BufferedReader reader = new BufferedReader(new InputStreamReader(request.getInputStream()));
        String input;
        StringBuilder builder = new StringBuilder();
        while ((input = reader.readLine()) != null) {
            builder.append(input);
        }
        input = builder.toString();
        String username;
        try {
            username = input.split(":")[1].replace("}", "").replace("\"", "");
        } catch (Exception e) {
            username = "defaultUsername";
        }
        // 以上部分代码是为了从浏览器发送的数据中提取出 username 字段，不用认真阅读

        User user = new User(username, "pass");
        response.setContentType("application/json;charset=UTF-8");
        try (PrintWriter out = response.getWriter()) {
            out.println(user);
        }
    }
}
```

## Servlet 中的跨域问题

Servlet 中解决跨域问题很简单，利用 `Filter` 即可。根据上面的原理，在发送跨域请求之前，浏览器会首先以 `OPTIONS` 方法发送一个预检请求，如果允许跨域，则继续发送跨域请求，如果不允许，则阻止后续请求，并在 conole 打印错误。

我们定制 `Filter`：

```java
@WebFilter(filterName = "CorsFilter", urlPatterns = "/*")
public class CorsFilter implements Filter {
    public void destroy() {
    }

    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain) throws ServletException, IOException {
        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) resp;
        if (request.getMethod().equals("OPTIONS")) {
            response.setHeader("Access-Control-Allow-Origin", "*");
            response.setHeader("Access-Control-Allow-Methods", "OPTIONS, GET, POST, PUT, DELETE");
            // -1 表示不缓存，正数值表示缓存预检请求的 秒 数
            // 在预检请求缓存的有效期内，后续的跨域请求不需要再发送预检请求
            response.setHeader("Access-Control-Max-Age", "-1"); 
            response.setHeader("Access-Control-Allow-Headers", "Authorization,x-requested-with,content-type");
            response.setHeader("Access-Control-Allow-Credentials", "true");
            return;
        }
        response.setHeader("Access-Control-Allow-Origin", "*");
        chain.doFilter(req, resp);
    }

    public void init(FilterConfig config) throws ServletException {
    }
}
```

对 OPTIONS 的处理上：因为这个请求只是为了检查服务器是否支持跨域，所以我们在设置完相应的 Header 之后可以直接 `return`，即不需要进行后续处理。

对其他 HTTP 方法，必须附带 `Access-Control-Allow-Origin`，否则虽然请求能够成功完成，浏览器会阻止结果的显示。可以注释掉 `chain.doFilter` 上一行的设置 Header 语句重新运行，在浏览器控制台的 Network 中可以看到请求返回的数据，但是不会显示在网页，并且控制台也会正常报错。说明浏览器阻止了 XMLHttpRequest 结果的显示。

## Spring Mvc 中的跨域问题

解决起来跟上面的类似，还是利用 Filter，不过鉴于这个问题的常见，Spring 为我们提供了一个实现好的 CorsFilter：`org.springframework.web.filter.CorsFilter`，我们只需要提供必要的参数，然后使用这个 Filter 即可。

示例代码我使用 Spring Boot + Spring Mvc，下面是 Controller：

```java
@RestController
public class WebController {
    @RequestMapping(value = "/", method = RequestMethod.POST, consumes = MediaType.APPLICATION_JSON_UTF8_VALUE)
    public User index(@RequestBody @NotNull User user) {
        return new User(user.getUsername(), "spring-mvc-pass");
    }
}
```

如果不加其他措施我们是不能通过跨域访问 `http://127.0.0.1:8080/` 的，下面我们将 Spring 提供的 Filter  加入示例项目：

```java
@Configuration
public class CorsConfig {
    @Bean
    public CorsFilter corsFilter() {
        final CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins(List.of("*"));
        configuration.setAllowedMethods(List.of("OPTIONS", "POST", "GET"));
        configuration.setAllowedHeaders(List.of("*"));
        configuration.setMaxAge(-1L);
        configuration.setAllowCredentials(true);
        final UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return new CorsFilter(source);
    }
}
```

重新运行项目，已经可以成功从浏览器端发送跨域请求并接收响应了。

简单查看 CorsFilter 的源码，实现思想与我们自己实现的大体相同。具体来说是：首先通过 `AntPathMatcher` 获取到我们用 `source.registerCorsConfiguration` 注册的配置，然后进行预处理：

1. 如果请求不是 Cors 请求（请求头包含 `Origin` 字段），则跳过 CorsFilter
1. 如果有其他 Filter 已经在请求头中添加了 `Access-Control-Allow-Origin` 字段，跳过 CorsFilter
1. 如果是同源请求，跳过 CorsFilter
1. 如果配置为空，并且该请求是预检请求（Preflight Request，请求头有 `Origin` 字段、请求是 `OPTIONS` 方法，请求头不包含 `Access-Control-Allow-Origin` 字段），则拒绝该请求
1. 如果配置为空，并且该请求不是预检请求，则跳过 CorsFilter

预处理之后进行跨域权限检查：

1. 检查 Origin 是否符合配置，不符合则拒绝请求
1. 检查请求 Method 是否符合配置，不符合则拒绝请求
1. 检查 Headers 是否符合配置，不符合则拒绝请求

然后设置响应：

1. setAccessControlAllowOrigin
1. setAccessControlAllowMethods（仅预检请求）
1. setAccessControlAllowHeaders（仅预检请求，并且配置了 Allowed Headers）
1. setAccessControlExposeHeaders（配置了 Expose Headers）
1. setAccessControlAllowCredentials（配置了 Allowed Credentials）
1. setAccessControlMaxAge（仅预检请求，并且配置了 Max Age）

可以看出来，比起我们自己设计 CorsFilter，Spring 提供的 CorsFilter 还自己根据配置进行了各种检查，而我们在使用的时候只需要传入自定义的配置即可，极大地简化了开发难度同时又增加了灵活性。

## Spring 4.2 以后的跨域问题

Spring Framework 4.2 版本开始原生支持跨域，相较于之前配置 Filter 的方式，Spring 提供了一个注解 `@CorssOrigin` 来简化配置。

我们只需要将 `@CrossOrigin` 添加在需要的 API 或者 Controller 上，然后设置其参数即可完成配置：

```java
@CrossOrigin(
        origins = "*",
        methods = {RequestMethod.OPTIONS, RequestMethod.GET, RequestMethod.POST},
        allowedHeaders = {"Authorization", "Content-Type"},
        maxAge = -1L,
        allowCredentials = "false"
)
@RequestMapping(value = "/", method = RequestMethod.POST, consumes = MediaType.APPLICATION_JSON_UTF8_VALUE)
public User index(@RequestBody @NotNull User user) {
    return new User(user.getUsername(), "spring-mvc-pass");
}
```

或者直接使用默认配置

```java
@CrossOrigin
@RequestMapping(value = "/", method = RequestMethod.POST, consumes = MediaType.APPLICATION_JSON_UTF8_VALUE)
public User index(@RequestBody @NotNull User user) {
    return new User(user.getUsername(), "spring-mvc-pass");
}
```

我们还可以通过 Configuration 来手动配置全局规则：

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/conf")
                .allowedOrigins("*")
                .allowedMethods("OPTIONS", "GET", "POST");
    }
}
```

```java
@RequestMapping(value = "/conf", method = RequestMethod.POST, consumes = MediaType.APPLICATION_JSON_UTF8_VALUE)
public User config(@RequestBody @NotNull User user) {
    return new User(user.getUsername(), "spring-mvc-pass");
}
```

## 参考链接

- [HTTP 访问控制（CORS）](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS)
