---
title: "使用 Java 解析 MJPEG 为 JPG 数组"
date: 2021-01-19
layout: post
tags: [Java, MJPEG]
---

## MJPEG 是什么

MJPEG 全名为 "Motion Joint Photographic Experts Group"，是一种视频编码格式，中文名称翻译为“技术即运动静止图像（或逐帧）压缩技术”。MJPEG 广泛应用于非线性编辑领域可精确到帧编辑和多层图像处理，把运动的视频序列作为连续的静止图像来处理，这种压缩方式单独完整地压缩每一帧，在编辑过程中可随机存储每一帧，可进行精确到帧的编辑

如果只是想在前端展示, 直接用 img 标签即可, 例如你有一个地址为 `http://localhost:8964/apple.mjpeg` 的视频流:

```html
<img src="http://localhost:8964/apple.mjpeg" width="1080" height="1920">
```

## 如何解析

通过上面介绍我们知道了 MJPEG 是由完整的图片组成的视频流, 我所使用的 MJPEG 是由 jpg 图片组成的, 那么想解析 MJPEG 就要先了解一下 jpg 格式. 找一张 jpg 图片, 使用 `Bless Hex Editor` 或者 `UltraEdit` 打开

![hex-start]({{ site.baseurl }}/assets/images/2021-01-19-jpg-hex-start.png)

从上图可以看到 jpg 图片的开头是 `FF D8`

![hex-end]({{ site.baseurl }}/assets/images/2021-01-19-jpg-hex-end.png)

下拉到结尾, 可以看到 jpg 图片以 `FF D9` 结尾.

### 方法一

通过万能的 Google 找到了 [github 上的实现](https://github.com/andrealaforgia/mjpeg-client/blob/master/src/main/java/com/andrealaforgia/mjpegclient/MjpegRunner.java), 具体代码如下:

```java
private static final String CONTENT_LENGTH = "Content-Length: ";
private static final String CONTENT_TYPE = "Content-type: image/jpeg";
private InputStream urlStream;

/**
* Using the urlStream get the next JPEG image as a byte[]
*
* @return byte[] of the JPEG
* @throws IOException
*/
private byte[] retrieveNextImage() throws IOException {
    int currByte = -1;

    String header = null;
    // build headers
    // the DCS-930L stops it's headers

    boolean captureContentLength = false;
    StringWriter contentLengthStringWriter = new StringWriter(128);
    StringWriter headerWriter = new StringWriter(128);

    int contentLength = 0;

    while ((currByte = urlStream.read()) > -1) {
        if (captureContentLength) {
            if (currByte == 10 || currByte == 13) {
                contentLength = Integer.parseInt(contentLengthStringWriter.toString());
                break;
            }
            contentLengthStringWriter.write(currByte);

        } else {
            headerWriter.write(currByte);
            String tempString = headerWriter.toString();
            int indexOf = tempString.indexOf(CONTENT_LENGTH);
            if (indexOf > 0) {
                captureContentLength = true;
            }
        }
    }

    // 255 indicates the start of the jpeg image
    while ((urlStream.read()) != 255) {
        // just skip extras
    }

    // rest is the buffer
    byte[] imageBytes = new byte[contentLength + 1];
    // since we ate the original 255 , shove it back in
    imageBytes[0] = (byte) 255;
    int offset = 1;
    int numRead = 0;
    while (offset < imageBytes.length
            && (numRead = urlStream.read(imageBytes, offset, imageBytes.length - offset)) >= 0) {
        offset += numRead;
    }

    return imageBytes;
}
```

上述代码是通过先解析 MJPEG 数据流的头部, 获取到一张图片的 `Content-Length`, 然后开始查找 jpg 图片的开头, 接下来持续读取直到图片长度, 就可以获取到完整的一张 jpg 图片的 byte 数组.

通过 `Ctrl C+V` 现在可以解析 MJPEG 为单张图片的 byte[] 了, 但是测试后发现解析一张图片居然要 100ms 左右, 这怎么行, 连 30 帧都达不到, 八成是这个连 star 都没有的项目代码写得不行, 我得给它优化一下, 于是就有了方法二

### 方法二

从上文我们知道 jpg 图片以 `FF D8` 开头, `FF D9` 结尾, 我的初步思路就是定义一个足够大的 byte[], 将数据读取到 byte[] 中, 然后遍历一遍查找 jpg 图片的开头和结尾, 然后取出开头和结尾间的数据就是一帧完整的图片, 代码如下:

```java
public byte[] retrieve() {
    byte ff = (byte)0xFF;
    byte s2 = (byte)0xD8;
    byte e2 = (byte)0xD9;

    int startIndex = -1;
    int endIndex = -1;
    // 1MB 的缓存, 根据项目实际情况调整
    final int length = 1024 * 1024;
    byte[] cache = new byte[length];
    int offset = 0;
    try {
        int read;
        while ((read = inputStream.read(cache, offset, length - offset)) != -1) {
            for (int i = offset; i < offset + read - 1; i++) {
                if (startIndex == -1) {
                    if (cache[i] == ff && cache[i + 1] == s2) {
                        startIndex = i;
                        logger.info("startIndex: {}", startIndex);
                    }
                    continue;
                }

                if (cache[i] == ff && cache[i + 1] == e2) {
                    endIndex = i + 2;
                    logger.info("endIndex: {}", endIndex);
                    break;
                }
            }
            offset += read;
            if (startIndex != -1 && endIndex != -1) {
                return Arrays.copyOfRange(cache, startIndex, endIndex);
            }
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
    return null;
}
```

使用上述代码测试, 发现图片的开始位置是固定的 68, 将前面 68 个 byte 转换成 String 输出:

```yaml
--boundryString
Content-type: image/jpeg
Content-Length: 67832
```

原来每一帧图像数据都有这么一个头文件, 通过这个头文件我们可以拿到图片的格式和大小, 那么只要解析这个头文件, 拿到图片的大小, 再定位到 jpg 的开头, 就能得到完整的图像.

## 结语

经过测试, 新的方法解析速度还是 100ms, 时间大部分消耗在 `read = inputStream.read(cache, offset, length - offset)` 这里了 ...
