---
title: "Scrcpy Server 源码阅读"
date: 2022-07-11T13:02:46+08:00
draft: true
---

之前的文章查看了 scrcpy client 端代码, 这里查看 server 端代码, 同样基于 `scrcpy-1.17`.

## 程序入口

server 位于 `sercver/` 目录, 程序入口为 `Server.java` 的 `main()` 方法, 在这里调用了 `scrcpy()` 方法, 看看 `scrcpy()` 方法做了什么:

```Java
private static void scrcpy(Options options) throws IOException {

    // ...
    // ...

    // 是否为 adb forward
    boolean tunnelForward = options.isTunnelForward();

    // 创建与主机通信的 server 或 socket, 拿到 videoSocket 和 controlSocket
    try (DesktopConnection connection = DesktopConnection.open(device, tunnelForward)) {
        // 编码器, 内部由 MediaCodec 实现
        ScreenEncoder screenEncoder = new ScreenEncoder(options.getSendFrameMeta(), options.getBitRate(), options.getMaxFps(), codecOptions,
                options.getEncoderName());

        Thread controllerThread = null;
        Thread deviceMessageSenderThread = null;
        if (options.getControl()) {
            final Controller controller = new Controller(device, connection);

            // asynchronous
            // 创建一个线程, 调用 controller.control()
            controllerThread = startController(controller);
            // 创建线程, 调用 sender
            deviceMessageSenderThread = startDeviceMessageSender(controller.getSender());

            device.setClipboardListener(new Device.ClipboardListener() {
                @Override
                public void onClipboardTextChanged(String text) {
                    controller.getSender().pushClipboardText(text);
                }
            });
        }

        try {
            // synchronous
            // 向主机发送视频流
            screenEncoder.streamScreen(device, connection.getVideoFd());
        } catch (IOException e) {
            // this is expected on close
            Ln.d("Screen streaming stopped");
        } finally {
            if (controllerThread != null) {
                controllerThread.interrupt();
            }
            if (deviceMessageSenderThread != null) {
                deviceMessageSenderThread.interrupt();
            }
        }
    }
}
```

### DesktopConnection.open(device, tunnelForward) 创建连接

```Java
public static DesktopConnection open(Device device, boolean tunnelForward) throws IOException {
    LocalSocket videoSocket;
    LocalSocket controlSocket;
    if (tunnelForward) {    // adb forward
        LocalServerSocket localServerSocket = new LocalServerSocket(SOCKET_NAME);
        try {
            videoSocket = localServerSocket.accept();
            // send one byte so the client may read() to detect a connection error
            videoSocket.getOutputStream().write(0); // 发送 1 byte 用来验证连接有效性的数据
            try {
                controlSocket = localServerSocket.accept();
            } catch (IOException | RuntimeException e) {
                videoSocket.close();
                throw e;
            }
        } finally {
            localServerSocket.close();
        }
    } else {    // adb reverse
        videoSocket = connect(SOCKET_NAME);
        try {
            controlSocket = connect(SOCKET_NAME);
        } catch (IOException | RuntimeException e) {
            videoSocket.close();
            throw e;
        }
    }

    // 获取视频帧尺寸并发送
    DesktopConnection connection = new DesktopConnection(videoSocket, controlSocket);
    Size videoSize = device.getScreenInfo().getVideoSize();
    connection.send(Device.getDeviceName(), videoSize.getWidth(), videoSize.getHeight());
    return connection;
}
```

这个方法根据启动时候是 `adb forward` 或 `adb reverse` 创建 ServerSocket 或者 Socket 来获取与主机通信的 `videoSocket` 和 `controlSocket`, 并发送视频帧的宽高数据

### startController() 启动 Controller

startController() 方法启动一个线程, 调用了 `controller.control()`, 最终调用了 `controller.handleEvent()`:

```Java
private void handleEvent() throws IOException {
    ControlMessage msg = connection.receiveControlMessage();
    switch (msg.getType()) {
        case ControlMessage.TYPE_INJECT_KEYCODE:
            if (device.supportsInputEvents()) {
                injectKeycode(msg.getAction(), msg.getKeycode(), msg.getRepeat(), msg.getMetaState());
            }
            break;
        case ControlMessage.TYPE_INJECT_TEXT:
            if (device.supportsInputEvents()) {
                injectText(msg.getText());
            }
            break;
        case ControlMessage.TYPE_INJECT_TOUCH_EVENT:
            if (device.supportsInputEvents()) {
                injectTouch(msg.getAction(), msg.getPointerId(), msg.getPosition(), msg.getPressure(), msg.getButtons());
            }
            break;
        
        // ...
        // ...

        default:
            // do nothing
    }
}
```

这个方法通过 `controlSocket` 接收主机发送过来的指令, 执行相应的操作控制 Android 设备

* 触摸与滚动操作是如何实现的?

查看代码, `controller.injectTouch() -> device.injectEvent() -> SERVICE_MANAGER.getInputManager().injectInputEvent(inputEvent, mode)`, 从如下代码可以看出, 最终是通过反射调用 `InputManager` 的 `injectInputEvent` 实现手机的触摸和滚动等操作

```Java
public boolean injectInputEvent(InputEvent inputEvent, int mode) {
    try {
        Method method = getInjectInputEventMethod();
        return (boolean) method.invoke(manager, inputEvent, mode);
    } catch (InvocationTargetException | IllegalAccessException | NoSuchMethodException e) {
        Ln.e("Could not invoke method", e);
        return false;
    }
}
```

### startDeviceMessageSender(controller.getSender())

这个方法调用了 `DeviceMessageSender.loop()` -> `DesktopConnection.sendDeviceMessage()`

```Java
public void sendDeviceMessage(DeviceMessage msg) throws IOException {
    writer.writeTo(msg, controlOutputStream);
}
```

通过 `controlSocket` 将 Android 设备剪切板内容发送给主机

### screenEncoder.streamScreen(device, connection.getVideoFd())

streamScreen 调用了 internalStreamScreen():

```Java
private void internalStreamScreen(Device device, FileDescriptor fd) throws IOException {
    MediaFormat format = createFormat(bitRate, maxFps, codecOptions);
    device.setRotationListener(this);
    boolean alive;
    try {
        do {
            MediaCodec codec = createCodec(encoderName);
            IBinder display = createDisplay();
            ScreenInfo screenInfo = device.getScreenInfo();
            Rect contentRect = screenInfo.getContentRect();
            // include the locked video orientation
            Rect videoRect = screenInfo.getVideoSize().toRect();
            // does not include the locked video orientation
            Rect unlockedVideoRect = screenInfo.getUnlockedVideoSize().toRect();
            int videoRotation = screenInfo.getVideoRotation();
            int layerStack = device.getLayerStack();

            setSize(format, videoRect.width(), videoRect.height());
            configure(codec, format);
            Surface surface = codec.createInputSurface();
            setDisplaySurface(display, surface, videoRotation, contentRect, unlockedVideoRect, layerStack);
            codec.start();
            try {
                alive = encode(codec, fd);
                // do not call stop() on exception, it would trigger an IllegalStateException
                codec.stop();
            } finally {
                destroyDisplay(display);
                codec.release();
                surface.release();
            }
        } while (alive);
    } finally {
        device.setRotationListener(null);
    }
}
```

这个方法将屏幕数据编码, 然后通过 encode() 方法写入 `videoSocket`, 发送到主机:

```Java
private boolean encode(MediaCodec codec, FileDescriptor fd) throws IOException {
    boolean eof = false;
    MediaCodec.BufferInfo bufferInfo = new MediaCodec.BufferInfo();

    while (!consumeRotationChange() && !eof) {
        int outputBufferId = codec.dequeueOutputBuffer(bufferInfo, -1);
        eof = (bufferInfo.flags & MediaCodec.BUFFER_FLAG_END_OF_STREAM) != 0;
        try {
            if (consumeRotationChange()) {
                // must restart encoding with new size
                break;
            }
            if (outputBufferId >= 0) {
                ByteBuffer codecBuffer = codec.getOutputBuffer(outputBufferId);

                // 发送帧数据的头文件
                if (sendFrameMeta) {
                    writeFrameMeta(fd, bufferInfo, codecBuffer.remaining());
                }

                // 发送视频帧
                IO.writeFully(fd, codecBuffer);
            }
        } finally {
            if (outputBufferId >= 0) {
                codec.releaseOutputBuffer(outputBufferId, false);
            }
        }
    }

    return !eof;
}
```

## 总结

server 的流程是先创建通信的 socket, 创建子线程处理主机发来的指令, 获取屏幕数据, 编码后发送给主机

## 问题

* MediaCodec 是什么?
* 如何通过 DisplayManager 得到屏幕内容?