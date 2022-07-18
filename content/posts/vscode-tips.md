---
title: "Vscode Tips"
date: 2022-07-13T10:24:51+08:00
draft: true
tags: [vscode, SpringBoot]
_build:
    list: false
    render: false
---

记录 vscode 使用过程中遇到的一些问题及解决方案

## 设置 maven 的镜像地址

* 找到 maven 安装路径下的 `settings.xml`, 比如我的环境是: `/opt/maven/conf/settings.xml`;
* 找一个镜像源贴到 `mirrors` 节点里面, 比如我使用的是华为镜像:

```xml
<mirrors>
    ...
    <mirror>
        <id>huaweicloud</id>
        <mirrorOf>*</mirrorOf>
        <url>https://repo.huaweicloud.com/repository/maven/</url>
    </mirror>
    ...
</mirrors>
```

* 将路径 `/opt/maven/conf/settings.xml` 填入 `Settings -> Extensions -> Java -> Configuration -> Maven:User Settings` 中;

## The Language Support for Java server crashed ... 问题

* 环境: ArchLinux
* 版本: 1.69.0
* Java: OpenJDK-18.0.1.1
* Maven: 3.8.6

解决方法: 取消勾选 `Settings -> Java -> Jdt -> Ls -> Lombok Support: Enabled`

[参考方案](https://blog.csdn.net/hzlxb123/article/details/92801867)