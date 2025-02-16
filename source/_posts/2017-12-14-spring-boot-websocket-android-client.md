---
title: SpringBoot WebSocket 发送广播，Android 接收
copyright: true
date: 2017-12-14 10:16:11 +0800
categories: [Java]
tags: [java, spring boot, websocket, android]
topic: spring
---

## WebSocket

WebSocket 为客户端、浏览器和服务端提供了双工异步通信的功能，即客户端（浏览器、Android）可以向服务器发送消息，服务器端也可以向客户端发送消息。

<!-- more -->

WebSocket 是通过一个 socket 来实现双工异步通信能力的。但是直接使用 WebSocket 协议开发程序会十分繁琐，因此我们使用它的子协议 `STOMP`，它是一个更高级别的协议。`STOMP` 协议使用一个基于帧的格式来定义消息，与 HTTP 的 request 和 response 类似（具有类似于 `@RequestMpping` 的注解 `@MessageMapping`）。

## Spring Boot 的支持

Spring Boot 对内嵌的 Tomcat、Jetty 和 Undertow 使用 WebSocket 提供了支持。

Spring Boot 为 WebSocket 提供的 starter pom 是 `spring-boot-starter-websocket`。

## 服务器端

### 使用 Intellij IDEA + maven 搭建。

spring-boot-starter 选择 Thymeleaf 和 WebSocket

![starter pom](/img/tech/spring-boot-starter.png)

### 创建拦截器

拦截器可以在 WebSocket 握手前后进行一些预设置。

HandshakeInterceptor.java

```java
package me.xlui.im.config;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.web.socket.WebSocketHandler;
import org.springframework.web.socket.server.support.HttpSessionHandshakeInterceptor;

import java.util.Map;

public class HandshakeInterceptor extends HttpSessionHandshakeInterceptor {
    private static Logger logger = LoggerFactory.getLogger("xlui");

    /**
     * WebSocket 握手前
     * <p>
     * 可以设置数据到 attributes 中，并在 WebSocketHandler 的 session 中获取
     */
    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response, WebSocketHandler wsHandler, Map<String, Object> attributes) throws Exception {
        logger.info("HandshakeInterceptor: beforeHandshake");
        logger.info("Attributes: " + attributes.toString());
        return super.beforeHandshake(request, response, wsHandler, attributes);
    }

    // WebSocket 握手后
    @Override
    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response, WebSocketHandler wsHandler, Exception ex) {
        logger.info("HandshakeInterceptor: afterHandshake");
        super.afterHandshake(request, response, wsHandler, ex);
    }
}
```

### 创建配置类

WebSocketConfig.java:

```java
package me.xlui.im.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.AbstractWebSocketMessageBrokerConfigurer;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;

@Configuration
// 启用 Websocket 的消息代理
@EnableWebSocketMessageBroker
public class WebSocketConfig extends AbstractWebSocketMessageBrokerConfigurer {
	// 注册 STOMP 协议的节点（Endpoint），并映射为指定的 URL
	// 我们使用 STOMP，所以不需要再实现 WebSocketHandler。
	// 实现 WebSocketHandler 的目的是接收和处理消息，STOMP 已经为我们做了这些。
	@Override
	public void registerStompEndpoints(StompEndpointRegistry stompEndpointRegistry) {
		// 注册 STOMP 协议的节点，并指定使用 SockJS 协议
		stompEndpointRegistry.addEndpoint("/im").addInterceptors(new HandshakeInterceptor()).withSockJS();
	}

	// 配置使用消息代理
	@Override
	public void configureMessageBroker(MessageBrokerRegistry registry) {
		// 统一配置消息代理，消息代理即订阅点，客户端通过订阅消息代理点接受消息
		registry.enableSimpleBroker("/b", "/g", "/user");

		// 配置点对点消息的前缀
		registry.setUserDestinationPrefix("/user");
	}
}
```

通过注解 `@EnableWebSocketMessageBroker` 开启使用 STOMP 协议来传输基于代理（message broker）的消息，这时控制器使用 `@MessageMapping` 就像使用 `@RequestMapping` 一样。

### 消息发送与接收类

Message.java:

```java
package me.xlui.im.message;

/**
 * 浏览器向服务端发送的消息应该用此类接受
 */
public class Message {
    private String name;

    public String getName() {
        return name;
    }
}
```

Response.java:

```java
package me.xlui.im.message;

/**
 * 服务器向客户端发送此类消息
 */
public class Response {
    private String responseMessage;

    public Response(String responseMessage) {
        this.responseMessage = responseMessage;
    }

    public String getResponseMessage() {
        return responseMessage;
    }
}
```

### 控制器

```java
package me.xlui.im.web;

import me.xlui.im.message.ChatMessage;
import me.xlui.im.message.Message;
import me.xlui.im.message.Response;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.messaging.handler.annotation.DestinationVariable;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Controller;

@Controller
public class WebSocketController {
    @Autowired
    SimpMessagingTemplate simpMessagingTemplate;

    // 当客户端向服务器发送请求时，通过 `@MessageMapping` 映射 /broadcast 这个地址
    @MessageMapping("/broadcast")
    // 当服务器有消息时，会对订阅了 @SendTo 中的路径的客户端发送消息
    @SendTo("/b")
    public Response say(Message message) {
        return new Response("Welcome, " + message.getName() + "!");
    }

    @MessageMapping("/group/{groupID}")
    public void group(@DestinationVariable int groupID, Message message) {
        Response response = new Response("Welcome to group " + groupID + ", " + message.getName() + "!");
        simpMessagingTemplate.convertAndSend("/g/" + groupID, response);
    }

    @MessageMapping("/chat")
    public void chat(ChatMessage chatMessage) {
        Response response = new Response("Receive message from user " + chatMessage.getFromUserID() + ": " + chatMessage.getMessage());
        simpMessagingTemplate.convertAndSendToUser(String.valueOf(chatMessage.getUserID()), "/msg", response);
    }
}
```

## 浏览器演示页面

静态资源放在 `src/main/resources/static` 下

### 广播 broadcast.html

```html
<!doctype html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta http-equiv="Content-Type" content="text/html;charset=UTF-8"/>
    <title>Spring Boot WebSocket 广播式</title>
</head>
<body onload="disconnect()">
<noscript>
    <h2 style="color: #ff0000;">貌似你的浏览器不支持 websocket</h2>
</noscript>
<div>
    <button id="connect" onclick="connect();">连接</button>
    <button id="disconnect" onclick="disconnect();" disabled="disabled">断开连接</button>
</div>
<div id="conversationDiv">
    <label for="name">输入你的名字：</label>
    <input type="text" id="name" placeholder="name"/>
    <button id="sendName" onclick="sendName();">发送</button>
    <p id="response"></p>
</div>
<script th:src="@{sockjs.min.js}"></script>
<script th:src="@{stomp.min.js}"></script>
<script th:src="@{jquery.js}"></script>
<script type="text/javascript">
    var stompClient = null;

    function setConnected(connected) {
        conn = $('#connect');
        disconn = $('#disconnect');
        if (connected) {
            conn.attr('disabled', 'true');
            disconn.removeAttr('disabled');
        } else {
            conn.removeAttr('disabled');
            disconn.attr('disabled', 'true');
        }
        document.getElementById('conversationDiv').style.visibility = connected ? 'visible' : 'hidden';
        $('#response').html();
    }

    function connect() {
        var socket = new SockJS("/im");
        stompClient = Stomp.over(socket);
        stompClient.connect({}, function (frame) {
            setConnected(true);
            console.log('Connected: ' + frame);
            stompClient.subscribe('/b', function (response) {
                showResponse(JSON.parse(response.body).response);
            });
        });
    }

    function disconnect() {
        if (stompClient != null) {
            stompClient.disconnect();
        }
        setConnected(false);
        console.log('Disconnected');
    }

    function sendName() {
        var name = $('#name').val();
        stompClient.send('/broadcast', {}, JSON.stringify({'name': name}));
    }

    function showResponse(message) {
        var response = $('#response');
        response.html(response.text() + '\r\n' + message);
    }
</script>
</body>
</html>
```

动态群组与点对点聊天的代码见 GitHub。

### 配置路径映射

WebMvcConfig.java:

```java
package me.xlui.im.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.ViewControllerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;

@Configuration
public class WebMvcConfig extends WebMvcConfigurerAdapter {
    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/broadcast").setViewName("/broadcast");
        registry.addViewController("/group").setViewName("/group");
        registry.addViewController("/chat").setViewName("/chat");
    }
}
```

### 浏览器测试

运行程序，浏览器同时打开数个窗口，连接。

广播

![browser](/img/tech/websocket-browser-broadcast.gif)

动态群组

![browser](/img/tech/websocket-browser-group.gif)

点对点

![browser](/img/tech/websocket-browser-chat.gif)

## 安卓客户端

`STOMP` 协议在 Android 系统中没有默认实现，不过有开源项目已经实现了，所以我们只需要添加依赖直接使用就好。

### build.gradle(project)

```yml
allprojects {
    repositories {
        google()
        jcenter()
        maven { url "https://jitpack.io" }
        // 添加 maven 仓库
    }
}
```

### build.gradle(app)

```yml
compile 'com.squareup.okhttp3:okhttp:3.9.0'
compile 'org.java-websocket:Java-WebSocket:1.3.7'
compile 'com.github.NaikSoftware:StompProtocolAndroid:1.4.3'
```

我们使用的是 `StompProtocolAndroid`，它同时依赖于 WebSocket 的标准实现 `Java-WebSocket`。

不过 `Java-WebSocket` 实现的 `WebSocket` 类在我这里不太好使，所以我换了 `okhttp` 实现的 `WebSocket` 类。

### 网络权限

在 `AndroidManifest.xml` 中添加网络权限：

```xml
<uses-permission android:name="android.permission.INTERNET"/>
```

### 布局

![安卓广播布局](/img/tech/websocket-android-layout.png)

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:gravity="center"
    android:orientation="vertical"
    tools:context="me.xlui.im.activities.BroadcastActivity">

    <!--tools:text="广播" -->

    <LinearLayout
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="horizontal">

        <Button
            android:id="@+id/broadcast"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:text="@string/broadcast" />

        <Button
            android:id="@+id/groups"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:text="@string/groups" />

        <Button
            android:id="@+id/chat"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:text="@string/chat" />
    </LinearLayout>

    <LinearLayout
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:gravity="center_vertical|center"
        android:orientation="horizontal">

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:padding="12dp"
            android:text="@string/broadcast_prompt" />

        <EditText
            android:id="@+id/name"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:inputType="text"
            android:padding="16dp" />

    </LinearLayout>

    <Button
        android:id="@+id/send"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="@string/send" />

    <TextView
        android:id="@+id/show"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />

</LinearLayout>
```

### 主程序

广播 Activity 的代码，其他代码（动态群组、点对点）见 GitHub。

```java
package me.xlui.im.activities;

import android.content.Intent;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.util.Log;
import android.widget.Button;
import android.widget.EditText;
import android.widget.TextView;
import android.widget.Toast;

import org.json.JSONException;
import org.json.JSONObject;
import org.reactivestreams.Subscriber;
import org.reactivestreams.Subscription;

import me.xlui.im.R;
import me.xlui.im.conf.Const;
import me.xlui.im.util.StompUtils;
import okhttp3.WebSocket;
import ua.naiksoftware.stomp.Stomp;
import ua.naiksoftware.stomp.client.StompClient;

public class BroadcastActivity extends AppCompatActivity {
    private Button broadcast;
    private Button groups;
    private Button chat;

    private EditText name;
    private Button send;
    private TextView result;

    private void init() {
        broadcast = findViewById(R.id.broadcast);
        broadcast.setEnabled(false);
        groups = findViewById(R.id.groups);
        chat = findViewById(R.id.chat);
        name = findViewById(R.id.name);
        send = findViewById(R.id.send);
        result = findViewById(R.id.show);
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_broadcast);

        this.init();

        StompClient stompClient = Stomp.over(WebSocket.class, Const.address);
        // 连接服务器
        stompClient.connect();
        Toast.makeText(this, "开始连接", Toast.LENGTH_SHORT).show();
        StompUtils.connect(stompClient);

        // 订阅消息
        stompClient.topic(Const.broadcastResponse).subscribe(stompMessage -> {
            JSONObject jsonObject = new JSONObject(stompMessage.getPayload());
            Log.i(Const.TAG, "Receive: " + stompMessage.getPayload());
            runOnUiThread(() -> {
                try {
                    result.append(jsonObject.getString("response") + "\n");
                } catch (JSONException e) {
                    e.printStackTrace();
                }
            });
        });

        send.setOnClickListener(v -> {
            JSONObject jsonObject = new JSONObject();
            try {
                jsonObject.put("name", name.getText());
            } catch (JSONException e) {
                e.printStackTrace();
            }
            stompClient.send(Const.broadcast, jsonObject.toString()).subscribe(new Subscriber<Void>() {
                @Override
                public void onSubscribe(Subscription s) {
                    Log.i(Const.TAG, "onSubscribe: 订阅成功！");
                }

                @Override
                public void onNext(Void aVoid) {

                }

                @Override
                public void onError(Throwable t) {
                    t.printStackTrace();
                    Log.e(Const.TAG, "发生错误：", t);
                }

                @Override
                public void onComplete() {
                    Log.i(Const.TAG, "onComplete: Send Complete!");
                }
            });
        });

        groups.setOnClickListener(v -> {
            Intent intent = new Intent();
            intent.setClass(BroadcastActivity.this, GroupActivity.class);
            startActivity(intent);
            this.finish();
        });
        chat.setOnClickListener(v -> {
            Intent intent = new Intent();
            intent.setClass(BroadcastActivity.this, ChatActivity.class);
            startActivity(intent);
            this.finish();
        });
    }
}
```

### 测试

广播

![broadcast](/img/tech/websocket-android-broadcast.gif)

动态群组

![group](/img/tech/websocket-android-group.gif)

点对点

![chat](/img/tech/websocket-android-chat.gif)

## 源码

源代码已经上传到 GitHub，[https://github.com/xlui/WebSocketExample](https://github.com/xlui/WebSocketExample)，欢迎 star。
