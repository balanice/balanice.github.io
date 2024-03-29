---
title: "adb tips"
date: 2021-01-10T08:12:37+08:00
draft: true
tags: Android
---

参考资料: [命令行工具](https://developer.android.google.cn/studio/command-line/)

## adb shell

### am

* 发送广播 `adb shell am boradcast -a [ACTION]` 例如：`adb shell am broadcast -a com.force.test.hello`
* 获取Activity启动时间 `adb shell am start -W [PACKAGE/.ACTIVITY]` 例如:`adb shell am start -W com.example/.MainActivity`
* 发送 json 格式的参数, 使用引号包裹 adb shell 后面的语句, 将其当为参数传递 `adb shell "am start service '"'{json}'"' -n xxx/.service"`
* 打开设置主页面 `adb shell am start com.android.settings/com.android.settings.Settings`

### svc

```shell
# 开启 WiFi
adb shell svc wifi enable
# 关闭 WiFi
adb shell svn wifi disable
# 开启手机数据
adb shell svc data enable
# 关闭手机数据
adb shell svc data disable
```

### dumpsys

参考地址:[dumpsys](https://developer.android.google.cn/studio/command-line/dumpsys)

`adb shell dumpsys meminfo <package_name>` 命令查看APP内存使用情况

`dumpsys` 是一个运行在 Android 设备上的可以提供系统服务信息的工具. 可以在命令行通过 adb 工具调用 `dumpsys` 来获取已连接的设备上面的服务的诊断信息. 这个输出通常比你想象中要更冗长, 可以使用如下描述的命令行选项来筛选你想要的系统服务输出.

**语法:**

`adb shell dumpsys [-t timeout] [--help | -l | --skip services | service [arguments] | -c | -h]`

运行 `adb shell dumpsys` 命令会输出所有的服务信息. 

dump activity stack 信息:

`adb shell dumpsys activity` 

查询 app 版本:

`adb shell dumpsys package [packagename] | grep versionName`

查看当前 Activity:

`adb shell dumpsys activity | grep mResumed`

查看移动网络信号强度:

`adb shell dumpsys telephony.registry | grep mSignalStrength`

### echo

* 输出手机当前时间, 包含毫秒值: `adb shell echo \$EPOCHREALTIME`

### input

```shell
adb shell input text xxx # 输入文字
adb shell input tap 540 900 # 点击坐标
adb shell input swipe 540 900 540 900 # swipe 输入两个相同的坐标, 也是点击 
adb shell input swipe 540 900 540 900 1000 # 长按坐标 1s
adb shell input keyevent 82 # 解锁屏幕
```

### keyevent

* Power   ->  26
* 解锁    ->  82
* 语音助手 -> 261

### cmd

```shell
# 展开通知栏
adb shell cmd statusbar expand-notification
# 收起通知栏
adb shell cmd statusbar collapse-notification
```

## fastboot 刷机

* root `adb root`
* 以 fastboot 模式启动 `adb reboot bootloader`
* 重新挂载 `adb remount`
* fastboot 刷机 `fastboot flash [para] [para file]`
* fastboot OEM 解锁 `fastboot oem unlock`
* `adb disable-verity` 在userdebug 版本中关闭

## 文件操作 [pull|push|sync]

* 推送本地文件到手机: `adb push [locale file] [remote]`
* 拉取手机文件到本地: `adb pull [remote] [locale path]`
* 复制所有有变化的文件到设备中: `adb sync [DIR]`

## logcat

* 抓取指定 app 的 log

    `adb logcat --pid=$(adb shell pidof -s [packagename])` or 
    
    `adb logcat --pid='adb shell pidof -s [packagename]'`

*  清除 logcat 缓存

    `adb logcat -c`

* read: unexpected EOF

    解决方法: `adb logcat -G 2m`

## 启动 adb server

* `adb -a start-server`       启动 adb 并监听所有端口, 某些版本 adb 无法监听本机 IP;
* `adb -a nodaemon server`    将 adb 启动为前台进程, 同时监听本机 IP 的 adb 端口;

## Other

* `adb shell wm size` 获取手机屏幕分辨率
* `adb shell monkey -v 3` 执行 3 次 monkey 操作