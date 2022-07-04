---
title: "Android 证书固定(certificate pinning)及如何绕过它"
date: 2022-02-18
date: 2022-02-18T08:12:37+08:00
draft: true
tags: Android
categories:
    - Android
---

## 如何配置固定证书

从 Android7.0 开始, Android 默认只信任 system 证书, 不再信任 user 及管理员安装的证书. 固定证书则更进一步, 将服务器证书信息存入到 app 中, 在访问后台接口时候, 如果发现后台证书与预置的证书不一致, 就会拒绝访问. 实现证书固定的方式有两种, Android 官方给定的和在代码中设置:

### OkHttp设置 certificate pinning

如果不知道网站证书的 sha256 hash 值的话, 可以先填写一个其他网站的, 比如我这里用的就是 google 的, 这个值可以填一个错的, 但千万不能写一个空的, 如 "sha/AAA", 否则会触发不了我们想要的效果

```kotlin
class PinningRequest {

    private var client: OkHttpClient? = null
    private val url = "https://api.github.com/api/users/balanice"
    private val tag = "PinningRequest"

    init {
        val pinning = CertificatePinner.Builder()
            .add("api.github.com", "sha256/7HIpactkIAq2Y49orFOOQKurWxmmSFZhBCoQYcRhJ3Y=")
            .build()

        client = OkHttpClient.Builder().certificatePinner(pinning).build()
    }

    fun getDetails() {
        val request = Request.Builder()
            .url(url)
            .get()
            .build()

        client?.newCall(request)?.enqueue(object: Callback {
            override fun onFailure(call: Call, e: IOException) {
                Log.e(tag, e.message.toString())
                e.printStackTrace()
            }

            override fun onResponse(call: Call, response: Response) {
                Log.i(tag, response.toString())
            }
        })
    }
}
```

因为上面我们使用的证书信息与目标网站不一样, 在执行的时候会爆出如下错误, 这里的三个证书就是目标网站的真实证书链, 用前面两个来替换之前的假证书就好了

```
2022-02-19 17:39:15.282 29052-29118/com.force.demo E/PinningRequest: Certificate pinning failure!
      Peer certificate chain:
        sha256/azE5Ew0LGsMgkYqiDpYay0olLAS8cxxNGUZ8OJU756k=: CN=*.github.com,O=GitHub\, Inc.,L=San Francisco,ST=California,C=US
        sha256/vnCogm4QYze/Bc9r88xdA6NTQY74p4BAz2w5gxkLG2M=: CN=DigiCert High Assurance TLS Hybrid ECC SHA256 2020 CA1,O=DigiCert\, Inc.,C=US
        sha256/WoiWRyIOVNa9ihaBciRSC7XHjliYS9VwUGOIud4PB18=: CN=DigiCert High Assurance EV Root CA,OU=www.digicert.com,O=DigiCert Inc,C=US
      Pinned certificates for api.github.com:
        sha256/7HIpactkIAq2Y49orFOOQKurWxmmSFZhBCoQYcRhJ3Y=
```

将正确的证书加入代码中

```kotlin
 val pinning = CertificatePinner.Builder()
                .add("api.github.com", "sha256/azE5Ew0LGsMgkYqiDpYay0olLAS8cxxNGUZ8OJU756k=")
                .add("api.github.com", "sha256/vnCogm4QYze/Bc9r88xdA6NTQY74p4BAz2w5gxkLG2M=")
                .build()
```

### Android 官方配置方法

此方式只适用 Android 7.0+ 的手机, 设置固定证书时候, 应该始终包含一个备用密钥, 这样当更换证书时候不会影响应用的连接性. 此外, 可以设置证书固定的到期时间, 在该时间之后不再固定证书. 这有助于防止尚未更新的应用出现连接性问题. 创建文件 `res/xml/network_security_config.xml`, 证书可以使用我们上个步骤获取到的

```xml
    <?xml version="1.0" encoding="utf-8"?>
    <network-security-config>
        <base-config>
            <trust-anchors>
                <certificates src="system"/>
            </trust-anchors>
        </base-config>
        <domain-config>
            <domain includeSubdomains="true">api.github.com</domain>
            <pin-set expiration="2022-12-31">
                <pin digest="SHA-256">azE5Ew0LGsMgkYqiDpYay0olLAS8cxxNGUZ8OJU756k=</pin>
                <!-- backup pin -->
                <pin digest="SHA-256">vnCogm4QYze/Bc9r88xdA6NTQY74p4BAz2w5gxkLG2M=</pin>
            </pin-set>
        </domain-config>
    </network-security-config>
```

在 manifest.xml 中引用:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest ... >
    <application android:networkSecurityConfig="@xml/network_security_config"
                    ... >
        ...
    </application>
</manifest>
```

## 绕过 certificate pinning 对 HTTPS 进行抓包

通过前面的了解, 不难发现, 在较新版本的 Android 手机上, 只要手机使用了 certificate pinning, 那么使用代理软件直接对第三方应用 HTTPS 抓包基本是不可能的了, 抓包基本只剩以下方法:
* root 手机, 将抓包软件的证书安装到 system 区;
* 对应用进行修改后重新打包, 然后再抓包;
* VirtualXposed 或者在手机上安装虚拟机的方式在华为 Mate30, P30 等 Android10+ 手机上实测用不了;

### 使用 apktool 对 apk 进行修改后重新打包, 再进行抓包

比如我们现有一个 app-debug.apk, 先用 apktool 进行解包

```shell
apktool d app-debug.apk
```

在解压后的 `app-debug` 文件夹中找到对应的 `res/xml/network_security_config.xml`, 在 base-config 中将 `user` 证书加入信任

```xml
    <base-config>
        <trust-anchors>
            <certificates src="user">
            <certificates src="system"/>
        </trust-anchors>
    </base-config>
```

另外代码中也可能设置了证书, 这就需要找到对应的 smali 代码进行修改了, 情况比较复杂此处不再展开