---
title: "Framework 代码结构"
date: 2022-07-14T12:21:50+08:00
draft: true
_build:
    list: false
    render: false
---

以 InputManager 为例:

* `/frameworks/base/core/java/android/hardware/input/` 下包含 `IInputManager.aidl` 和 `InputManager.java`
* `/frameworks/base/services/core/java/com/android/server/input/` 下的 `InputManagerService.java` 继承了 `IInputManager.Stub`, Wraps the C++ InputManager and provides its callbacks.
* `/frameworks/base/services/core/jni/` 下的 `com_android_server_input_InputManagerService.cpp` 被 `InputManagerService` 调用