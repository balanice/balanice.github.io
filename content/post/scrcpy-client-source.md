---
title: "scrcpy 客户端源码阅读"
date: 2022-07-06T08:09:37+08:00
draft: true
tags: [Android, C]
---

[scrcpy](https://github.com/Genymobile/scrcpy) 是一款提供投屏和操作 Android 设备的开源软件, 无需 root. 其源码分为 client 和 server 两部分, client 位于 app/ 目录中, 由 C 实现, server 位于 server/ 目录, 由 Java 实现. 本文基于 `scrcpy-1.17`.

client 程序入口位于 `main.c` 的 `main()` 函数, 调用了 `scrcpy()` 函数:

```C
int
main(int argc, char *argv[]) {
    //...
    //...
    int res = scrcpy(&args.opts) ? 0 : 1;

    avformat_network_deinit(); // ignore failure

    return res;
}
```

### scrcpy()

`scrcpy()` 函数位于 `scrcpy.c`, 我们定位到这里看看这个函数做了什么:

```C
bool
scrcpy(const struct scrcpy_options *options) {
    if (!server_init(&server)) {    // 初始化 server
        return false;
    }

    // ...
    // ...

    // 启动 server
    if (!server_start(&server, options->serial, &params)) {
        goto end;
    }

    server_started = true;

    // 初始化 SDL
    if (!sdl_init_and_configure(options->display, options->render_driver,
                                options->disable_screensaver)) {
        goto end;
    }

    // 通过 socket 连接上面启动的 server
    if (!server_connect_to(&server)) {
        goto end;
    }

    char device_name[DEVICE_NAME_FIELD_LENGTH];
    struct size frame_size;

    // screenrecord does not send frames when the screen content does not
    // change therefore, we transmit the screen size before the video stream,
    // to be able to init the window immediately
    // 当屏幕内容没有变化的时候, screenrecord 不会发送视频帧, 
    // 所以发送视频流之前会发送一个屏幕尺寸数据, 用来初始化窗口
    if (!device_read_info(server.video_socket, device_name, &frame_size)) {
        goto end;
    }

    struct decoder *dec = NULL;
    if (options->display) {     // 是否显示屏幕
        if (!fps_counter_init(&fps_counter)) {
            goto end;
        }
        fps_counter_initialized = true;

        if (!video_buffer_init(&video_buffer, &fps_counter,
                               options->render_expired_frames)) {
            goto end;
        }
        video_buffer_initialized = true;

        if (options->control) {
            if (!file_handler_init(&file_handler, server.serial,
                                   options->push_target)) {
                goto end;
            }
            file_handler_initialized = true;
        }

        // 初始化解码器, 底层由 ffmpeg 实现
        decoder_init(&decoder, &video_buffer);
        dec = &decoder;
    }

    struct recorder *rec = NULL;
    if (record) {   // 是否记录为文件
        if (!recorder_init(&recorder,
                           options->record_filename,
                           options->record_format,
                           frame_size)) {
            goto end;
        }
        rec = &recorder;
        recorder_initialized = true;
    }

    av_log_set_callback(av_log_callback);

    stream_init(&stream, server.video_socket, dec, rec);

    // now we consumed the header values, the socket receives the video stream
    // start the stream
    // 消费 header, socket 接收到了视频流
    if (!stream_start(&stream)) {
        goto end;
    }
    stream_started = true;

    // ...
    // ...

    input_manager_init(&input_manager, options);

    bool ret = event_loop(options);
    LOGD("quit...");

    screen_destroy(&screen);

end:
    // stop stream and controller so that they don't continue once their socket
    // is shutdown
    if (stream_started) {
        stream_stop(&stream);
    }
    if (controller_started) {
        controller_stop(&controller);
    }
    // ...
    // ...

    server_destroy(&server);    // 关闭 server

    return ret;
}
```

## server_start(&server, options->serial, &params) 启动 server

```C
bool
server_start(struct server *server, const char *serial,
             const struct server_params *params) {
    // ...
    // ...
    if (!push_server(serial)) { // 将 server push 到手机上
        goto error1;
    }

    // 开启端口转发
    if (!enable_tunnel_any_port(server, params->port_range,
                                params->force_adb_forward)) {
        goto error1;
    }

    // server will connect to our server socket
    // 启动 server
    server->process = execute_server(server, params);
    if (server->process == PROCESS_NONE) {
        goto error2;
    }
    
    // ...
    // ...
}
```

### push_server(serial) 将 scrcpy-server push 到 Android 设备上

```C
static bool
push_server(const char *serial) {
    char *server_path = get_server_path();
    if (!server_path) {
        return false;
    }
    if (!is_regular_file(server_path)) {
        LOGE("'%s' does not exist or is not a regular file\n", server_path);
        SDL_free(server_path);
        return false;
    }
    process_t process = adb_push(serial, server_path, DEVICE_SERVER_PATH);
    SDL_free(server_path);
    return process_check_success(process, "adb push");
}
```

这一步是通过 adb 将 scrcpy-server push 到 `/data/local/tmp/scrcpy-server.jar`

### enable_tunnel_any_port() 开启端口转发

```C
static bool
enable_tunnel_any_port(struct server *server, struct sc_port_range port_range,
                       bool force_adb_forward) {
    if (!force_adb_forward) {
        // Attempt to use "adb reverse"
        if (enable_tunnel_reverse_any_port(server, port_range)) {
            return true;
        }

        // if "adb reverse" does not work (e.g. over "adb connect"), it
        // fallbacks to "adb forward", so the app socket is the client

        LOGW("'adb reverse' failed, fallback to 'adb forward'");
    }

    return enable_tunnel_forward_any_port(server, port_range);
}
```

这一步是通过 `adb reverse` 或者 `adb forward` 创建端口转发, 后续通过 socket 与 server 端建立连接.

### execute_server(server, params) 启动 server

``` C
static process_t
execute_server(struct server *server, const struct server_params *params) {
    
    // ...
    // ...
    
    const char *const cmd[] = {
        "shell",
        "CLASSPATH=" DEVICE_SERVER_PATH,
        "app_process",
#ifdef SERVER_DEBUGGER
# define SERVER_DEBUGGER_PORT "5005"
# ifdef SERVER_DEBUGGER_METHOD_NEW
        /* Android 9 and above */
        "-XjdwpProvider:internal -XjdwpOptions:transport=dt_socket,suspend=y,server=y,address="
# else
        /* Android 8 and below */
        "-agentlib:jdwp=transport=dt_socket,suspend=y,server=y,address="
# endif
            SERVER_DEBUGGER_PORT,
#endif
        "/", // unused
        "com.genymobile.scrcpy.Server",
        SCRCPY_VERSION,
        log_level_to_server_string(params->log_level),
        max_size_string,
        bit_rate_string,
        max_fps_string,
        
        // ...
        // ...

    }
    return adb_execute(server->serial, cmd, sizeof(cmd) / sizeof(cmd[0]));
}
```

这个函数是通过 `adb shell CLASSPATH=/data/local/tmp/scrcpy-server.jar app_process / ...` 命令启动 `scrcpy-server`

## server_connect_to(&server) 创建连接 server 的 socket

```C
bool
server_connect_to(struct server *server) {
    // 未使用 adb forward
    if (!server->tunnel_forward) {
        server->video_socket = net_accept(server->server_socket);
        if (server->video_socket == INVALID_SOCKET) {
            return false;
        }

        server->control_socket = net_accept(server->server_socket);
        if (server->control_socket == INVALID_SOCKET) {
            // the video_socket will be cleaned up on destroy
            return false;
        }

        // we don't need the server socket anymore
        if (!atomic_flag_test_and_set(&server->server_socket_closed)) {
            // close it from here
            close_socket(server->server_socket);
            // otherwise, it is closed by run_wait_server()
        }
    } else {    // 使用 adb forward
        uint32_t attempts = 100;
        uint32_t delay = 100; // ms
        // 与 server 连接的第一个 socket, 传输视频流
        server->video_socket =
            connect_to_server(server->local_port, attempts, delay);
        if (server->video_socket == INVALID_SOCKET) {
            return false;
        }

        // we know that the device is listening, we don't need several attempts
        // 与 server 连接的第二个 socket, 传输控制命令
        server->control_socket =
            net_connect(IPV4_LOCALHOST, server->local_port);
        if (server->control_socket == INVALID_SOCKET) {
            return false;
        }
    }

    // ...
    // ...
}
```

这个函数根据使用的 `adb reverse` 或者 `adb forward` 与 `scrcpy-server` 建立两个 socket 连接, 第一个传输视频流, 第二个则是传输控制命令. 其中 `connect_to_server()` 函数使用了一个 `connect_and_read_byte()` 的函数:

```C
static socket_t
connect_and_read_byte(uint16_t port) {
    socket_t socket = net_connect(IPV4_LOCALHOST, port);
    if (socket == INVALID_SOCKET) {
        return INVALID_SOCKET;
    }

    char byte;
    // the connection may succeed even if the server behind the "adb tunnel"
    // is not listening, so read one byte to detect a working connection
    if (net_recv(socket, &byte, 1) != 1) {
        // the server is not listening yet behind the adb tunnel
        net_close(socket);
        return INVALID_SOCKET;
    }
    return socket;
}
```

在执行了 `adb tunnel` 之后的 `scrcpy-server` 可能还未完全启动并监听, 但此时通过 socket 连接已转发的端口仍然是成功的, 所以此处需要读取 1 byte 的数据来确认 `scrcpy-server` 是否成功启动, 如果未启动成功, 则需要重新连接.

## device_read_info() 读取在视频流数据前发出的帧分辨率数据

```C
bool
device_read_info(socket_t device_socket, char *device_name, struct size *size) {
    unsigned char buf[DEVICE_NAME_FIELD_LENGTH + 4];
    int r = net_recv_all(device_socket, buf, sizeof(buf));
    if (r < DEVICE_NAME_FIELD_LENGTH + 4) {
        LOGE("Could not retrieve device information");
        return false;
    }
    // in case the client sends garbage
    buf[DEVICE_NAME_FIELD_LENGTH - 1] = '\0';
    // strcpy is safe here, since name contains at least
    // DEVICE_NAME_FIELD_LENGTH bytes and strlen(buf) < DEVICE_NAME_FIELD_LENGTH
    strcpy(device_name, (char *) buf);
    size->width = (buf[DEVICE_NAME_FIELD_LENGTH] << 8)
            | buf[DEVICE_NAME_FIELD_LENGTH + 1];
    size->height = (buf[DEVICE_NAME_FIELD_LENGTH + 2] << 8)
            | buf[DEVICE_NAME_FIELD_LENGTH + 3];
    return true;
}
```

这里读取 64 + 4 个 byte 数据, 最后 4 个 byte 数据记录了视频帧的宽高.

## stream_start() 接收并解析视频流

此处通过 SDL 创建了一个线程, 用来接收和解析 h264 视频流

```C
bool
stream_start(struct stream *stream) {
    LOGD("Starting stream thread");

    stream->thread = SDL_CreateThread(run_stream, "stream", stream);
    if (!stream->thread) {
        LOGC("Could not start stream thread");
        return false;
    }
    return true;
}
```

`run_stream()` 函数中调用了一个 `stream_recv_packat()` 的函数:

```C
static bool
stream_recv_packet(struct stream *stream, AVPacket *packet) {
    // The video stream contains raw packets, without time information. When we
    // record, we retrieve the timestamps separately, from a "meta" header
    // added by the server before each raw packet.
    //
    // The "meta" header length is 12 bytes:
    // [. . . . . . . .|. . . .]. . . . . . . . . . . . . . . ...
    //  <-------------> <-----> <-----------------------------...
    //        PTS        packet        raw packet
    //                    size
    //
    // It is followed by <packet_size> bytes containing the packet/frame.

    uint8_t header[HEADER_SIZE];
    ssize_t r = net_recv_all(stream->socket, header, HEADER_SIZE);
    if (r < HEADER_SIZE) {
        return false;
    }

    uint64_t pts = buffer_read64be(header);
    uint32_t len = buffer_read32be(&header[8]);
    assert(pts == NO_PTS || (pts & 0x8000000000000000) == 0);
    assert(len);

    // ...
    // ...

}
```

从注释可以看出, 每一个 h264 raw 数据帧前面都有一个 12 byte 的 meta 数据, 前面 8 byte 记录 PTS, 后面 4 byte 记录 raw 数据的大小.

## controller_start(&controller) 创建并启动接收控制命令的线程

这里最终调用了 `controller.c` 中的 `process_msg()` 函数:

```C
static bool
process_msg(struct controller *controller,
              const struct control_msg *msg) {
    static unsigned char serialized_msg[CONTROL_MSG_MAX_SIZE];
    int length = control_msg_serialize(msg, serialized_msg);
    if (!length) {
        return false;
    }
    int w = net_send_all(controller->control_socket, serialized_msg, length);
    return w == length;
}
```

将用户的操作命令序列化以后通过 `control_socket` 发送给 `scrcpy-server`

## 总结

从源码中可以看出, client 端主要做的事情如下:

* 将 `scrcpy-server` push 到 Android 设备上;
* 通过 `adb reverse` 或 `adb forward` 创建端口转发;
* 通过 `adb shell CLASSPATH=... app_process ...` 命令启动 `scrcpy-server`;
* 建立 `video_socket` 和 `control_socket` 两个 socket 与 `scrcpy-server` 通信, 前者接收视频流, 后者发送操作命令;
* 通过 `video_socket` 读取视频帧的宽高初始化界面, 解析视频流, 将 raw 视频帧发送给前台展示; 接收用户操作指令, 序列化后通过 `control_socket` 发送给 `scrcpy-server` 处理;
* 将视频帧保存为文件;