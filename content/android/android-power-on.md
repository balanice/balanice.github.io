---
title: "Android 开机启动流程"
date: 2022-07-22T08:39:54+08:00
draft: true
---
>* 本文代码基于 Android 12

Android 系统底层基于 Linux 内核, 其启动过程与 Linux 系统类似, 在按下电源键以后, 首先启动的是引导程序 `BootLoader`, Linux 上一般是 `GRUB`. 我们平时刷机一般都是需要先解锁 `BootLoader`, 安装第三方 Recovery, 如 `TWRP`, 然后才能刷机, 当然厂商有更便捷的刷机方式, 不过普通人一般接触不到, 此处不做讨论.

>* `BootLoader` 为什么要加锁? 当然是出于安全原因, 为了保护手机中的数据. 比如有人捡到一台手机, 这个手机有锁屏密码, 但是 `BootLoader` 未加锁, 那么别人就可以轻易通过刷机的方式绕过锁屏密码, 类似于给电脑重装系统.

### BootLoader 做了什么?
`BootLoader` 是在操作系统运行之前执行的一段程序, 通过这个程序来初始化硬件设备, 建立内存空间的映射表, 从而建立适当的系统软硬件环境, 为最终调用系统内核做好准备.

### init
Linux 内核启动 Android 的 `init` 进程, 入口代码位于 `/system/core/init/main.cpp` 的 main() 函数:
```C
int main(int argc, char** argv) {
  #if __has_feature(address_sanitizer)
      __asan_set_error_report_callback(AsanReportCallback);
  #endif
      // Boost prio which will be restored later
      setpriority(PRIO_PROCESS, 0, -20);
      if (!strcmp(basename(argv[0]), "ueventd")) {
          return ueventd_main(argc, argv);
      }
  
     if (argc > 1) {
          if (!strcmp(argv[1], "subcontext")) {
              android::base::InitLogging(argv, &android::base::KernelLogger);
              const BuiltinFunctionMap& function_map = GetBuiltinFunctionMap();
  
              return SubcontextMain(argc, argv, &function_map);
          }
  
          if (!strcmp(argv[1], "selinux_setup")) {
              return SetupSelinux(argv);
          }
  
          if (!strcmp(argv[1], "second_stage")) {
              return SecondStageMain(argc, argv);
          }
      }
  
      return FirstStageMain(argc, argv);
}
```

`init` 进程被分为三个阶段, `first stage init`, `SELinux setup`, `second stage init`
* `first stage init` 代码位于 `/system/core/init/first_stage_init.cpp` 的 `FirstStageMain()`. 这个阶段主要是最小化设置设备, 为加载系统剩余功能做准备. 尤其是挂载 /dev, /proc 和包含系统代码的所有分区, 并将有 ramdisk 的设备的 `system.img` 挂载到 `/`. 这个阶段执行结束后会以 `selinux_setup` 的参数执行 `/system/bin/init`;
* `SELinux setup`  代码位于 `/system/core/init/selinux.cpp` 的 `SetupSelinux()`. 这个阶段 SELinux 会被编译加载到系统中. 这个阶段执行结束后, 会再次以 `second_stage` 参数执行 `/system/bin/init`;
* `second stage init` 代码位于 `/system/core/init/init.cpp` 的 `SecondStageMain()`. 这个阶段会通过 `init.rc` 脚本执行主要的初始化和启动服务进程工作.

### second stage init
```C
int SecondStageMain(int argc, char** argv) {
    // ...
    ActionManager& am = ActionManager::GetInstance();
    ServiceList& sm = ServiceList::GetInstance();

    LoadBootScripts(am, sm);
    // ...
}

static void LoadBootScripts(ActionManager& action_manager, ServiceList& service_list) {
    Parser parser = CreateParser(action_manager, service_list);

    std::string bootscript = GetProperty("ro.boot.init_rc", "");
    if (bootscript.empty()) {
        parser.ParseConfig("/system/etc/init/hw/init.rc");
        if (!parser.ParseConfig("/system/etc/init")) {
            late_import_paths.emplace_back("/system/etc/init");
        }
        // late_import is available only in Q and earlier release. As we don't
        // have system_ext in those versions, skip late_import for system_ext.
        parser.ParseConfig("/system_ext/etc/init");
        if (!parser.ParseConfig("/vendor/etc/init")) {
            late_import_paths.emplace_back("/vendor/etc/init");
        }
        if (!parser.ParseConfig("/odm/etc/init")) {
            late_import_paths.emplace_back("/odm/etc/init");
        }
        if (!parser.ParseConfig("/product/etc/init")) {
            late_import_paths.emplace_back("/product/etc/init");
        }
    } else {
        parser.ParseConfig(bootscript);
    }
}
```

从代码中看出, 有多个 `init.rc` 文件:
1. `/system/etc/init/hw/init.rc` 是主要的 `init.rc` 文件, 在 init 进程初次执行时候就会被加载, 它负责系统的初始化设置;
2. `/{system,system_ext,vendor,odm,product}/etc/init/` 这些文件会在 `/system/etc/init/hw/init.rc` 加载之后被立即加载. 这些文件的作用如下:
* `/system/etc/init/` 用于系统核心项目, 例如: SurfaceFlinger, MediaService, logd;
* `/vendor/etc/init/` 由 SoC 厂商使用, 里面是一些 SoC 功能所需的操作或守护进程;
* `/odm/etc/init/` 由设备制造商使用, 例如提供运动传感器或者其他周边功能需要的操作或守护进程.

>* 以前没有 `first stage` 挂载机制的旧设备能够在 `mount_all` 期间导入初始化脚本，但是不推荐使用, 并且这种用法在 Android Q 之后的设备上已被禁止。

### init.rc
在 `/system/core/rootdir/` 下找到 `init.rc`:
```
on late-init
    # ...
    # Now we can start zygote for devices with file based encryption
    trigger zygote-start

    # Remove a file to wake up anything waiting for firmware.
    trigger firmware_mounts_complete

    trigger early-boot
    trigger boot


# It is recommended to put unnecessary data/ initialization from post-fs-data
# to start-zygote in device's init.rc to unblock zygote start.
on zygote-start && property:ro.crypto.state=unencrypted
    wait_for_prop odsign.verification.done 1
    # A/B update verifier that marks a successful boot.
    exec_start update_verifier_nonencrypted
    start statsd
    start netd
    start zygote
    start zygote_secondary
```
在 `init.rc` 中, 看到了眼熟的 `zygote`, 在这里触发了 `zygote-start`, `zygote`, `netd`, `statsd` 等进程.

## zygote
在 `/system/core/rootdir/init.zygote64_32.rc` 脚本中:
```
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
    class main
    priority -20
    user root
    group root readproc reserved_disk
    socket zygote stream 660 root system
    socket usap_pool_primary stream 660 root system
    onrestart exec_background - system system -- /system/bin/vdc volume abort_fuse
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    onrestart restart wificond
    task_profiles ProcessCapacityHigh MaxPerformance
    critical window=${zygote.critical_window.minute:-off} target=zygote-fatal

service zygote_secondary /system/bin/app_process32 -Xzygote /system/bin --zygote --socket-name=zygote_secondary --enable-lazy-preload
    class main
    priority -20
    user root
    group root readproc reserved_disk
    socket zygote_secondary stream 660 root system
    socket usap_pool_secondary stream 660 root system
    onrestart restart zygote
    task_profiles ProcessCapacityHigh MaxPerformance
```

脚本中通过 `--zygote` 参数调用 `/system/bin/app_processxx` 进程, 这个 `app_process` 进程代码位于 ` /frameworks/base/cmds/app_process/app_main.cpp`:
```C++
int main(int argc, char* const argv[])
{
    // ..
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        }
        // ...
    }

    if (!niceName.isEmpty()) {
        runtime.setArgv0(niceName.string(), true /* setProcName */);
    }

    // ...
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    }
}
```

如果参数中带了 `--zygote`, niceName 就被设置为 `ZYGOTE_NICE_NAME`, 并启动 `com.android.internal.os.ZygoteInit`, 源码位于 `/frameworks/base/core/java/com/android/internal/os/ZygoteInit.java`:
```Java
public static void main(String[] argv) {
    // ...
    Runnable caller;
    try {
        zygoteServer = new ZygoteServer(isPrimaryZygote);

        if (startSystemServer) {
            Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);

            // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
            // child (system_server) process.
            if (r != null) {
                r.run();
                return;
            }
        }

        Log.i(TAG, "Accepting command socket connections");

        // The select loop returns early in the child process after a fork and
        // loops forever in the zygote.
        caller = zygoteServer.runSelectLoop(abiList);
    } catch (Throwable ex) {
        Log.e(TAG, "System zygote died with fatal exception", ex);
        throw ex;
    } finally {
        if (zygoteServer != null) {
            zygoteServer.closeServerSocket();
        }
    }

    // We're in the child process and have exited the select loop. Proceed to execute the
    // command.
    if (caller != null) {
        caller.run();
    }
}
```

创建 ZygoteServer, 监听请求, 源码位于 `/frameworks/base/core/java/com/android/internal/os/ZygoteServer.java`:
```Java
/**
 * Initialize the Zygote server with the Zygote server socket, USAP pool server socket, and USAP
 * pool event FD.
 *
 * @param isPrimaryZygote  If this is the primary Zygote or not.
 */
ZygoteServer(boolean isPrimaryZygote) {
    mUsapPoolEventFD = Zygote.getUsapPoolEventFD();

    if (isPrimaryZygote) {
        mZygoteSocket = Zygote.createManagedSocketFromInitSocket(Zygote.PRIMARY_SOCKET_NAME);
        mUsapPoolSocket =
                Zygote.createManagedSocketFromInitSocket(
                        Zygote.USAP_POOL_PRIMARY_SOCKET_NAME);
    } else {
        mZygoteSocket = Zygote.createManagedSocketFromInitSocket(Zygote.SECONDARY_SOCKET_NAME);
        mUsapPoolSocket =
                Zygote.createManagedSocketFromInitSocket(
                        Zygote.USAP_POOL_SECONDARY_SOCKET_NAME);
    }

    mUsapPoolSupported = true;
    fetchUsapPoolPolicyProps();
}
```

ZygoteServer 中的循环 `runSelectLoop()`, 接收新连接的请求, 并读取连接发来的命令, 这部分的具体功能我们后续再分析:
```Java
/**
 * Runs the zygote process's select loop. Accepts new connections as
 * they happen, and reads commands from connections one spawn-request's
 * worth at a time.
 * @param abiList list of ABIs supported by this zygote.
 */
Runnable runSelectLoop(String abiList) {
    ArrayList<FileDescriptor> socketFDs = new ArrayList<>();
    ArrayList<ZygoteConnection> peers = new ArrayList<>();

    socketFDs.add(mZygoteSocket.getFileDescriptor());
    peers.add(null);

    mUsapPoolRefillTriggerTimestamp = INVALID_TIMESTAMP;

    while (true) {
        // ...
        int pollReturnValue;
        try {
            pollReturnValue = Os.poll(pollFDs, pollTimeoutMs);
        } catch (ErrnoException ex) {
            throw new RuntimeException("poll failed", ex);
        }

        if (pollReturnValue == 0) {
            // The poll returned zero results either when the timeout value has been exceeded
            // or when a non-blocking poll is issued and no FDs are ready.  In either case it
            // is time to refill the pool.  This will result in a duplicate assignment when
            // the non-blocking poll returns zero results, but it avoids an additional
            // conditional in the else branch.
            mUsapPoolRefillTriggerTimestamp = INVALID_TIMESTAMP;
            mUsapPoolRefillAction = UsapPoolRefillAction.DELAYED;

        } else {
            boolean usapPoolFDRead = false;

            while (--pollIndex >= 0) {
                if ((pollFDs[pollIndex].revents & POLLIN) == 0) {
                    continue;
                }

                if (pollIndex == 0) {
                    // Zygote server socket
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    peers.add(newPeer);
                    socketFDs.add(newPeer.getFileDescriptor());
                } else if (pollIndex < usapPoolEventFDIndex) {
                    // Session socket accepted from the Zygote server socket

                    try {
                        ZygoteConnection connection = peers.get(pollIndex);
                        boolean multipleForksOK = !isUsapPoolEnabled()
                                && ZygoteHooks.isIndefiniteThreadSuspensionSafe();
                        final Runnable command =
                                connection.processCommand(this, multipleForksOK);

                        // TODO (chriswailes): Is this extra check necessary?
                        if (mIsForkChild) {
                            // We're in the child. We should always have a command to run at
                            // this stage if processCommand hasn't called "exec".
                            if (command == null) {
                                throw new IllegalStateException("command == null");
                            }

                            return command;
                        } else {
                            // We're in the server - we should never have any commands to run.
                            if (command != null) {
                                throw new IllegalStateException("command != null");
                            }

                            // We don't know whether the remote side of the socket was closed or
                            // not until we attempt to read from it from processCommand. This
                            // shows up as a regular POLLIN event in our regular processing
                            // loop.
                            if (connection.isClosedByPeer()) {
                                connection.closeSocket();
                                peers.remove(pollIndex);
                                socketFDs.remove(pollIndex);
                            }
                        }
                    } catch (Exception e) {
                        if (!mIsForkChild) {
                            // We're in the server so any exception here is one that has taken
                            // place pre-fork while processing commands or reading / writing
                            // from the control socket. Make a loud noise about any such
                            // exceptions so that we know exactly what failed and why.

                            Slog.e(TAG, "Exception executing zygote command: ", e);

                            // Make sure the socket is closed so that the other end knows
                            // immediately that something has gone wrong and doesn't time out
                            // waiting for a response.
                            ZygoteConnection conn = peers.remove(pollIndex);
                            conn.closeSocket();

                            socketFDs.remove(pollIndex);
                        } else {
                            // We're in the child so any exception caught here has happened post
                            // fork and before we execute ActivityThread.main (or any other
                            // main() method). Log the details of the exception and bring down
                            // the process.
                            Log.e(TAG, "Caught post-fork exception in child process.", e);
                            throw e;
                        }
                    } finally {
                        // Reset the child flag, in the event that the child process is a child-
                        // zygote. The flag will not be consulted this loop pass after the
                        // Runnable is returned.
                        mIsForkChild = false;
                    }

                } else {
                    // ...
                }
            }

            // ...
        }

        // ...
    }
}
```
### Zygote 总结
1. 通过 init.rc 脚本启动 Zygote;
2. Zygote 创建 ServerSocket 来接收客户端请求;
3. fork 出 SystemServer 进程;

### 问题
* USAP 是什么?

## SystemServer
在前面 ZygoteInit 的 main 方法调用了 `forkSystemServer()` 方法, 返回一个 Runnable 对象:
```Java
private static Runnable forkSystemServer(String abiList, String socketName,
            ZygoteServer zygoteServer) {
    
    // ...

    /* Hardcoded command line to start the system server */
    String[] args = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1023,"
                    + "1024,1032,1065,3001,3002,3003,3006,3007,3009,3010,3011",
            "--capabilities=" + capabilities + "," + capabilities,
            "--nice-name=system_server",
            "--runtime-args",
            "--target-sdk-version=" + VMRuntime.SDK_VERSION_CUR_DEVELOPMENT,
            "com.android.server.SystemServer",  // 记住这个参数, 后续会用到
    };
    ZygoteArguments parsedArgs;

    int pid;

    try {
        ZygoteCommandBuffer commandBuffer = new ZygoteCommandBuffer(args);
        try {
            parsedArgs = ZygoteArguments.getInstance(commandBuffer);
        } catch (EOFException e) {
            throw new AssertionError("Unexpected argument error for forking system server", e);
        }
        
        // ...

        /* Request to fork the system server process */
        pid = Zygote.forkSystemServer(
                parsedArgs.mUid, parsedArgs.mGid,
                parsedArgs.mGids,
                parsedArgs.mRuntimeFlags,
                null,
                parsedArgs.mPermittedCapabilities,
                parsedArgs.mEffectiveCapabilities);
    } catch (IllegalArgumentException ex) {
        throw new RuntimeException(ex);
    }

    /* For child process */
    if (pid == 0) {
        if (hasSecondZygote(abiList)) {
            waitForSecondaryZygote(socketName);
        }

        zygoteServer.closeServerSocket();
        return handleSystemServerProcess(parsedArgs);
    }

    return null;
}
```

上面代码通过 `Zygote.forkSystemServer()` 创建了 system server 进程, 这个方法最终是调用了一个 native 方法 `nativeForkSystemServer()` 来创建进程, 这个方法位于 ` /frameworks/base/core/jni/com_android_internal_os_Zygote.cpp`:
```C++
static jint com_android_internal_os_Zygote_nativeForkSystemServer(
        JNIEnv* env, jclass, uid_t uid, gid_t gid, jintArray gids,
        jint runtime_flags, jobjectArray rlimits, jlong permitted_capabilities,
        jlong effective_capabilities) {

    // ...

    pid_t pid = zygote::ForkCommon(env, true,
                                 fds_to_close,
                                 fds_to_ignore,
                                 true);
    if (pid == 0) {
        // System server prcoess does not need data isolation so no need to
        // know pkg_data_info_list.
        SpecializeCommon(env, uid, gid, gids, runtime_flags, rlimits, permitted_capabilities,
                        effective_capabilities, MOUNT_EXTERNAL_DEFAULT, nullptr, nullptr, true,
                        false, nullptr, nullptr, /* is_top_app= */ false,
                        /* pkg_data_info_list */ nullptr,
                        /* allowlisted_data_info_list */ nullptr, false, false);
    } else if (pid > 0) {
        // The zygote process checks whether the child process has died or not.
        ALOGI("System server process %d has been created", pid);
        gSystemServerPid = pid;
        // There is a slight window that the system server process has crashed
        // but it went unnoticed because we haven't published its pid yet. So
        // we recheck here just to make sure that all is well.
        int status;
        if (waitpid(pid, &status, WNOHANG) == pid) {
            ALOGE("System server process %d has died. Restarting Zygote!", pid);
            // 如果 system server 挂掉了, 重启 Zygote.
            RuntimeAbort(env, __LINE__, "System server process has died. Restarting Zygote!");
        }

        if (UsePerAppMemcg()) {
            // Assign system_server to the correct memory cgroup.
            // Not all devices mount memcg so check if it is mounted first
            // to avoid unnecessarily printing errors and denials in the logs.
            if (!SetTaskProfiles(pid, std::vector<std::string>{"SystemMemoryProcess"})) {
                ALOGE("couldn't add process %d into system memcg group", pid);
            }
        }
    }
    return pid;
}
```
从代码注释中可以看到, 如果 system server 挂掉了, 就会重启 Zygote 进程. system server 作为 Zygote 启动的第一个进程, 其地位确实非常高, 到了与 Zygote 同生共死的地步.

### system server 的作用
system server 调用 ZygoteInit 的 `handleSystemServerProcess()` 方法来发挥自己的作用:
```Java
 private static Runnable handleSystemServerProcess(ZygoteArguments parsedArgs) {
    // ...

    if (parsedArgs.mInvokeWith != null) {
        String[] args = parsedArgs.mRemainingArgs;
        // If we have a non-null system server class path, we'll have to duplicate the
        // existing arguments and append the classpath to it. ART will handle the classpath
        // correctly when we exec a new process.
        if (systemServerClasspath != null) {
            String[] amendedArgs = new String[args.length + 2];
            amendedArgs[0] = "-cp";
            amendedArgs[1] = systemServerClasspath;
            System.arraycopy(args, 0, amendedArgs, 2, args.length);
            args = amendedArgs;
        }

        WrapperInit.execApplication(parsedArgs.mInvokeWith,
                parsedArgs.mNiceName, parsedArgs.mTargetSdkVersion,
                VMRuntime.getCurrentInstructionSet(), null, args);

        throw new IllegalStateException("Unexpected return from WrapperInit.execApplication");
    } else {
        ClassLoader cl = getOrCreateSystemServerClassLoader();
        if (cl != null) {
            Thread.currentThread().setContextClassLoader(cl);
        }

        /*
         * Pass the remaining arguments to SystemServer.
         */
        return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion,
                parsedArgs.mDisabledCompatChanges,
                parsedArgs.mRemainingArgs, cl);
    }
 }
```

查看 `zygoteInit()` 方法:
```Java
public static Runnable zygoteInit(int targetSdkVersion, long[] disabledCompatChanges,
            String[] argv, ClassLoader classLoader) {
    if (RuntimeInit.DEBUG) {
        Slog.d(RuntimeInit.TAG, "RuntimeInit: Starting application from zygote");
    }

    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ZygoteInit");
    RuntimeInit.redirectLogStreams();

    RuntimeInit.commonInit();
    ZygoteInit.nativeZygoteInit();
    return RuntimeInit.applicationInit(targetSdkVersion, disabledCompatChanges, argv,
            classLoader);
}
```

这个 `RuntimeInit` 位于 `/frameworks/base/core/java/com/android/internal/os/RuntimeInit.java`:
```Java
protected static Runnable applicationInit(int targetSdkVersion, long[] disabledCompatChanges,
        String[] argv, ClassLoader classLoader) {
    // If the application calls System.exit(), terminate the process
    // immediately without running any shutdown hooks.  It is not possible to
    // shutdown an Android application gracefully.  Among other things, the
    // Android runtime shutdown hooks close the Binder driver, which can cause
    // leftover running threads to crash before the process actually exits.
    nativeSetExitWithoutCleanup(true);

    VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);
    VMRuntime.getRuntime().setDisabledCompatChanges(disabledCompatChanges);

    final Arguments args = new Arguments(argv);

    // The end of of the RuntimeInit event (see #zygoteInit).
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

    // Remaining arguments are passed to the start class's static main
    return findStaticMain(args.startClass, args.startArgs, classLoader);
}
```

来到了 `findStaticMain()` 方法, 向这里传入了一个 args.startClass 参数, 就是之前创建 system server 进程时候使用的 `com.android.server.SystemServer`:
```Java
protected static Runnable findStaticMain(String className, String[] argv,
        ClassLoader classLoader) {
    Class<?> cl;

    try {
        cl = Class.forName(className, true, classLoader);
    } catch (ClassNotFoundException ex) {
        throw new RuntimeException(
                "Missing class when invoking static main " + className,
                ex);
    }

    Method m;
    try {
        m = cl.getMethod("main", new Class[] { String[].class });
    } catch (NoSuchMethodException ex) {
        throw new RuntimeException(
                "Missing static main on " + className, ex);
    } catch (SecurityException ex) {
        throw new RuntimeException(
                "Problem getting static main on " + className, ex);
    }

    int modifiers = m.getModifiers();
    if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
        throw new RuntimeException(
                "Main method is not public and static on " + className);
    }

    /*
     * This throw gets caught in ZygoteInit.main(), which responds
     * by invoking the exception's run() method. This arrangement
     * clears up all the stack frames that were required in setting
     * up the process.
     */
    return new MethodAndArgsCaller(m, argv);
}
```
这个方法通过反射拿到了 `com.android.server.SystemServer` 的 main 方法, 创建了一个 `MethodAndArgsCaller` 对象返回给 `ZygoteInit` 的 `forkSystemServer()`:
```Java
 if (startSystemServer) {
    Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);

    // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
    // child (system_server) process.
    if (r != null) {
        r.run();
        return;
    }
}
```
看看这个 `MethodAndArgsCaller` 对象的 `run` 方法:
```Java
/**
  * Helper class which holds a method and arguments and can call them. This is used as part of
  * a trampoline to get rid of the initial process setup stack frames.
  */
static class MethodAndArgsCaller implements Runnable {
    /** method to call */
    private final Method mMethod;

    /** argument array */
    private final String[] mArgs;

    public MethodAndArgsCaller(Method method, String[] args) {
        mMethod = method;
        mArgs = args;
    }

    public void run() {
        try {
            mMethod.invoke(null, new Object[] { mArgs });
        } catch (IllegalAccessException ex) {
            throw new RuntimeException(ex);
        } catch (InvocationTargetException ex) {
            Throwable cause = ex.getCause();
            if (cause instanceof RuntimeException) {
                throw (RuntimeException) cause;
            } else if (cause instanceof Error) {
                throw (Error) cause;
            }
            throw new RuntimeException(ex);
        }
    }
}
```
通过这个 `run()` 方法, 调用了 `com.android.server.SystemServer` 的 main 方法, 其源码位于 `/frameworks/base/services/java/com/android/server/SystemServer.java`:
```Java
public static void main(String[] args) {
    new SystemServer().run();
}
```
Hmmm... 看看这个 run 方法:
```Java
private void run() {
    // ...

    // Initialize native services.
    System.loadLibrary("android_servers");

    // ...

    // Initialize the system context.
    createSystemContext();

    // Call per-process mainline module initialization.
    ActivityThread.initializeMainlineModules();

    // Sets the dumper service
    ServiceManager.addService("system_server_dumper", mDumper);
    mDumper.addDumpable(this);

    // Create the system service manager.
    mSystemServiceManager = new SystemServiceManager(mSystemContext);
    mSystemServiceManager.setStartInfo(mRuntimeRestart,
            mRuntimeStartElapsedTime, mRuntimeStartUptime);
    mDumper.addDumpable(mSystemServiceManager);

    LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);

    // ...

    // Start services.
    try {
        t.traceBegin("StartServices");
        startBootstrapServices(t);  // 启动系统启动所需的一小部分关键服务.
        startCoreServices(t);       // 启动那些不在上面一步启动的必要服务, 如电池, SystemConfigService等.
        startOtherServices(t);      // 启动 AlarmManagerService, InputManagerService等
    } catch (Throwable ex) {
        Slog.e("System", "******************************************");
        Slog.e("System", "************ Failure starting system services", ex);
        throw ex;
    } finally {
        t.traceEnd(); // StartServices
    }

    // ...

    // Loop forever.
    // 进入死循环, 让进程保持存活
    Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```
上面代码还是比较清楚的, 这里启动了 SystemContex, SystemServiceManager, AlarmManagerService 等一系列我们熟悉的 framework 层服务:
```Java
private void startBootstrapServices(@NonNull TimingsTraceAndSlog t) {
    // ...
    // Activity manager runs the show.
    t.traceBegin("StartActivityManager");
    // TODO: Might need to move after migration to WM.
    ActivityTaskManagerService atm = mSystemServiceManager.startService(
            ActivityTaskManagerService.Lifecycle.class).getService();
    // 启动 ActivityManagerService
    mActivityManagerService = ActivityManagerService.Lifecycle.startService(
            mSystemServiceManager, atm);
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    mActivityManagerService.setInstaller(installer);
    mWindowManagerGlobalLock = atm.getGlobalLock();
    t.traceEnd();
    // ...
}


private void startOtherServices(@NonNull TimingsTraceAndSlog t) {
    // ...

    try {
        // ...

        // 启动 AlarmManagerService
        t.traceBegin("StartAlarmManagerService");
        mSystemServiceManager.startService(ALARM_MANAGER_SERVICE_CLASS);
        t.traceEnd();

        // 启动 InputManagerService
        t.traceBegin("StartInputManagerService");
        inputManager = new InputManagerService(context);
        t.traceEnd();

        // 启动 DeviceStateManagerService
        t.traceBegin("DeviceStateManagerService");
        mSystemServiceManager.startService(DeviceStateManagerService.class);
        t.traceEnd();

        if (!disableCameraService) {
            t.traceBegin("StartCameraServiceProxy");
            // 启动 CameraServiceProxy
            mSystemServiceManager.startService(CameraServiceProxy.class);
            t.traceEnd();
        }

        t.traceBegin("StartWindowManagerService");
        // WMS needs sensor service ready
        mSystemServiceManager.startBootPhase(t, SystemService.PHASE_WAIT_FOR_SENSOR_SERVICE);
        // 启动 WindowManagerService
        wm = WindowManagerService.main(context, inputManager, !mFirstBoot, mOnlyCore,
                new PhoneWindowManager(), mActivityManagerService.mActivityTaskManager);
        ServiceManager.addService(Context.WINDOW_SERVICE, wm, /* allowIsolated= */ false,
                DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PROTO);
        ServiceManager.addService(Context.INPUT_SERVICE, inputManager,
                /* allowIsolated= */ false, DUMP_FLAG_PRIORITY_CRITICAL);
        t.traceEnd();
    }

    // ...
}
```

### SystemServer 总结
SystemServer 的调用还是比较清晰的, Zygote 进程启动 SystemServer, SystemServer 启动一系列 framwork 层的服务, 如我们常常听到的 ActivityManagerService, AlarmManagerService 等, 然后进入 `Looper.loop()` 死循环, 保持进程一直存活.

---
参考:
    [深入理解zygote](https://blog.csdn.net/Innost/article/details/47207845#t7)
    [Android 源码](http://www.aospxref.com/android-12.0.0_r3/xref/)