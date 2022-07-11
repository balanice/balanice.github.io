---
title: "使用 minicap 搭建远程操作网站"
date: 2021-03-27T08:12:37+08:00
draft: true
tags: [Android, SpringBoot]
---

## 技术选型

我们本次使用 minicap 获取手机屏幕内容, 使用 SpringBoot 实现网站服务, ddmlib 来对手机进行操作, WebSocket 传输图片数据, JavaScript + Html 实现前端界面.

## 实现

### 添加依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>

<dependency>
    <groupId>com.android.tools.ddms</groupId>
    <artifactId>ddmlib</artifactId>
    <version>27.1.3</version>
</dependency>
```

### 创建 WebSocket 服务端

1. 使用 WebSocket 传输数据, 在连接上时候启动 minicap, 接受并解析 minicap 数据, 在检测到 WebSocket 关闭或发生错误时候关闭 minicap, 代码如下:

```java
@Component
@ServerEndpoint("/screen")
public class MiniWebSocket {

    private static final Logger logger = LoggerFactory.getLogger(MiniWebSocket.class);

    @OnOpen
    public void onOpen(Session session) {
        logger.info("onOpen");
        SpringContextUtil.getBean(DeviceService.class).startScreen(session);
    }

    @OnMessage
    public void onMessage(Session session, String message) {
        logger.info("message: {}", message);
    }

    @OnClose
    public void onClose(Session session) {
        logger.info("onClose");
        SpringContextUtil.getBean(DeviceService.class).stopScreen();
    }

    @OnError
    public void onError(Session session, Throwable t) {
        logger.error(t.getMessage(), t);
        SpringContextUtil.getBean(DeviceService.class).stopScreen();
    }
}
```

2. WebSocket 配置

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig {
    @Bean
    public ServerEndpointExporter serverEndpointExporter() {
        return new ServerEndpointExporter();
    }
}
```

### 使用 java 代码实现对 minicap 的操作, 对数据流的解析等, 详见我之前的文章及[项目源码](https://github.com/balanice/miniserver)

### 前端实现

这里主要建立 WebSocket 的连接, 接收数据及将页面显示出来

```html
<!DOCTYPE html>
<html>

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">

    <title>miniserver</title>
    <script>
        var ws;

        function connect() {
            var url = "ws://127.0.0.1:8080/screen";
            ws = new WebSocket(url);
            ws.onopen = function () {
                console.debug('Info: WebSocket connection opened.');
            };
            ws.onmessage = function (event) {
                console.debug('Received: ' + event.data);
                // 将图片数据转换为 url 方便显示
                var url = URL.createObjectURL(event.data);
                document.getElementById("img_show").src = url;
            };
            ws.onclose = function () {
                console.debug('Info: WebSocket connection closed.');
            };
        }

        function disconnect() {
            if (ws != null) {
                ws.close();
            }
            document.getElementById("img_show").src = null;
            console.debug("disconnect");
        }
    </script>
</head>

<body>
<h1>MiniServer</h1>
<button onclick="connect()">Connect</button>
<button onclick="disconnect()">Disconnect</button>
<br />
<img id="img_show" width="540" height="960" />
</body>

</html>
```

## 实现结果

![hex-end](/images/2021-04-05_result.jpg)