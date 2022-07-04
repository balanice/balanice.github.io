---
title: "minicap 使用"
date: 2021-02-21T08:12:37+08:00
draft: true
tags: [Android, Java]
categories:
    - Android
    - Java
---

## minicap 是什么

minicap 是为 Android 系统开发的一款快速输出手机界面的库, 不需要 root 权限. 通过它可以实现超快的截图, 实时显示手机界面. 通常用来开发投屏软件或者远程调试平台.

## minicap 如何启动

参考[官方文档](https://github.com/DeviceFarmer/minicap)

### 编译

#### 下载 minicap 工程到本地

```shell
git submodule init
git submodule update
```

#### 使用 NDK 编译

minicap 编译需要使用到 NDK, 请先下载好新版 NDK, 并配置好环境变量, 然后使用下面命令编译:

```shell
ndk-build
```

编译成功后, 可执行文件生成在 `./libs` 文件夹

### 启动

#### 查询手机 cpu 架构:

```shell
ABI=$(adb shell getprop ro.product.cpu.abi | tr -d '\r')
```

#### 将对应架构的二进制文件推入手机:

```shell
adb push libs/$ABI/minicap /data/local/tmp/
```

#### 添加可执行权限 (部分手机需要):

```shell
adb shell chmod 777 /data/local/tmp/minicap
```

#### 推送 `minicap.so` 库. `minicap.so` 需要使用 `AOSP` 源码进行编译, minicap 已经预先编译好了, 我们可以直接拿来用

```shell
SDK=$(adb shell getprop ro.build.version.sdk | tr -d '\r')
adb push jni/minicap-shared/aosp/libs/android-$SDK/$ABI/minicap.so /data/local/tmp/
```

#### 查看帮助信息

```shell
adb shell LD_LIBRARY_PATH=/data/local/tmp /data/local/tmp/minicap -h
```

#### 查询手机分辨率

```shell
adb shell wm size
```

#### 启动 minicap, 通过上一步拿到手机分辨率信息, 如 `Physical size: 1080x1920`, 那么我们就可以通过如下命令启动 minicap:

```shell
adb shell LD_LIBRARY_PATH=/data/local/tmp /data/local/tmp/minicap -P 1080x1920@1080x1920/0
```

#### 创建端口转发, 将手机端口映射到本地

```shell
adb forward tcp:1313 localabstract:minicap
```

#### 验证, 通过如下命令如果获取到数据, 那么就证明 minicap 成功启动

```shell
nc localhost 1313
```

## minicap 数据解析

### minicap 数据协议

启动 minicap 后, minicap 会发送一个 24 byte 的头文件, 该头文件只会在启动后发送一次, 之后就是帧数据, 每一帧的前 4 byte 是图像数据按 byte 算的长度, 后面跟着的就是图像数据

#### Header 结构

|Bytes  |长度    |格式                   |说明
|-------|---    |---                    |---
|0      |1      |unsigned char          |版本号,目前为1
|1      |1      |unsigned char          |header的长度
|2-5    |4      |unint32(low endian)    |minicap 的 pid
|6-9    |4      |unint32(low endian)    |设备真实宽度
|10-13  |4      |unint32(low endian)    |设备真实高度
|14-17  |4      |unint32(low endian)    |设备虚拟宽度
|18-21  |4      |unint32(low endian)    |设备虚拟高度
|22     |1      |unsigned char          |Display orientation
|23     |1      |unsigned char          |Quirk bitflags (see below)

#### 帧数据结构

|Bytes  |Length |Type               |Explantion
|---    |---    |---                |---
|0-3    |4      |uint32(low endian) |Frame size in bytes(=n)
|4-(n+4)|n      |unsigned char[]    |Frame in JPG format

### 解析 minicap 数据

通过上面的列表, 我们知道了 minicap 中数据的涵义, 把其中的 byte 数据转换为对应格式的数据即可, 这里我们对数据进行解析并保存为本地图片, 代码如下:

```java
private static final int CACHE_SIZE = 1024 * 1024;

public void run() {
    Socket socket = new Socket();
    InetSocketAddress address = new InetSocketAddress("127.0.0.1", 1313);
    byte[] cache = new byte[CACHE_SIZE];
    boolean readHead = false;
    int count = 0;
    int index = 0;
    int imageSize = 0;
    try {
        socket.connect(address);
        InputStream inputStream = socket.getInputStream();
        int read;
        while ((read = inputStream.read(cache, index, CACHE_SIZE - index)) != -1) {
            // 读取头文件
            if (!readHead) {
                int version = cache[0] & 0xFF;
                int size = cache[1] & 0xFF;
                int pid = Utils.byteArrayToInt(Arrays.copyOfRange(cache, 2, 6));
                int readWidth = Utils.byteArrayToInt(
                        Arrays.copyOfRange(cache, 6, 10));
                int realHeight = Utils.byteArrayToInt(
                        Arrays.copyOfRange(cache, 10, 14));
                logger.info("version: {}, size: {}, pid: {}, realWidth: {}, 
                        realHeight:{}", version, size, pid, readWidth, realHeight);
                // 依次类推 ...
                readHead = true;
            } else {
                index += read;

                int offset = 0;
                boolean flag = false;
                // 一次读取的数据量可能超过一张图片
                while (index - offset >= imageSize + 4) {
                    // 获取图片数据大小
                    if (imageSize == 0) {
                        imageSize = Utils.byteArrayToInt(
                                Arrays.copyOfRange(cache, offset, offset + 4));
                        if (imageSize < 0) {
                            logger.info("imageSize: {}", imageSize);
                            break;
                        }
                        continue;
                    }
                    offset += 4;
                    logger.info("imageSize: {}, index: {}, offset: {}", 
                            imageSize, index, offset);
                    
                    // 将图片保存至本地
                    byte[] bytes = Arrays.copyOfRange(cache, offset, 
                            offset + imageSize);
                    ByteArrayInputStream stream = new ByteArrayInputStream(bytes);
                    BufferedImage i = ImageIO.read(stream);
                    String filename = String.format("%s%s%d.jpg", 
                            "/home/force/Projects/temp", File.separator, count++);
                    ImageIO.write(i, "JPG", new File(filename));

                    offset += imageSize;
                    imageSize = 0;
                    flag = true;
                }

                // 将剩下的数据移动到缓存数组的头部
                if (flag) {
                    for (int i = 0, j = offset; i < index - offset; i++, j++) {
                        cache[i] = cache[j];
                    }
                    index -= offset;
                }
                if (imageSize < 0) {
                    break;
                }
            }
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
    logger.info("run end");
}

// byte[] 与 int 互相转换方法, 
// 参考: https://stackoverflow.com/questions/5399798/byte-array-and-int-conversion-in-java
public class Utils {

    public static int byteArrayToInt(byte[] b) {
        return (b[0] & 0xFF) |
                (b[1] & 0xFF) << 8 |
                (b[2] & 0xFF) << 16 |
                (b[3] & 0xFF) << 24;
    }

    public static byte[] intToByteArray(int a) {
        return new byte[]{
                (byte) (a & 0xFF),
                (byte) ((a >> 8) & 0xFF),
                (byte) ((a >> 16) & 0xFF),
                (byte) ((a >> 24) & 0xFF)
        };
    }
}
```