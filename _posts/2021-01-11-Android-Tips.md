---
title: "Android Tips"
date: 2021-01-11
tags: Android
layout: post
---

## Android 部分小技巧

### Android 自动弹出软键盘：

```java
Window window = getWindow();
if (window != null) {
     window.setSoftInputMode(WindowManager.LayoutParams.SOFT_INPUT_STATE_ALWAYS_VISIBLE);
}
```

### 分享一个修复固定屏幕的 Bug 的一点经验：

`设置 -> 锁屏、指纹、安全 -> 更多安全设置 -> 屏幕固定`

现在可以通过最近任务固定屏幕，固定屏幕的意图是只允许用户在一个 Task 内使用手机，这个时候如果有点击事件触发了后台服务的逻辑，可能会出现 Bug

```java
ActivityManager am = (ActivityManager) getSystemService(Context.ACTIVITY_SERVICE);
if (am.getLockTaskModeState() != ActivityManager.LOCK_TASK_MODE_NONE) {
    // 当前处于固定屏幕模式下
}
```

### Android lazy load 问题
 
```java
getWindow.getDecorView.post(new Runnable{
    
    @override
    public void run() {
        new Handler().post(new Runnable{});
    });
```

使用上面代码可以滞后加载数据，但是并不能在这里放过多的操作，因为它也是运行在 UI 线程，如加载多个 Fragment, 易造成 ANR，加载一个 Fragment 时候未测出问题

### 关于号码格式化及其他号码知识

[参照](https://github.com/googlei18n/libphonenumber)

### 修改 SearchView 的光标

```java
try {
    Field searchTextField = mSearchView.getClass().getDeclaredField("mSearchSrcTextView");
    boolean searchTextAccessible = searchTextField.isAccessible();
    searchTextField.setAccessible(true);

    TextView searchText = (TextView) searchTextField.get(mSearchView);
    Field cursorDrawableField = TextView.class.getDeclaredField("mCursorDrawableRes");
    boolean cursorDrawableAccessible = cursorDrawableField.isAccessible();
    cursorDrawableField.setAccessible(true);
    cursorDrawableField.set(searchText, R.drawable.ic_search_edit_indicator);
    cursorDrawableField.setAccessible(cursorDrawableAccessible);
    searchTextField.setAccessible(searchTextAccessible);
} catch (NoSuchFieldException | IllegalAccessException | ClassCastException e) {
     e.printStackTrace();
}
```

在修改 MK 时经常可以看到使用了 `\` 这个换行连接符，所以实际上，通过换行连接符连接的多行语句其实是一句代码，因此，注释无关内容时要将注释移到该语句的最后，不能直接在某几行中通过 `#` 号注释。

### 锁屏 Dialog 弹出输入法

在锁屏时默认情况下打开时钟后弹出的 Dialog 无法显示输入法，经搜索后发现 Dialog 有自己的 window 参数，按如下方法修改即可实现锁屏下的 Dialog 弹出输入法：

```java
mEditDialog.getWindow().addFlags(WindowManager.LayoutParams.FLAG_DISMISS_KEYGUARD);
mEditDialog.getWindow().addFlags(WindowManager.LayoutParams.FLAG_SHOW_WHEN_LOCKED);
mEditDialog.getWindow().addFlags(WindowManager.LayoutParams.FLAG_TURN_SCREEN_ON);
mEditDialog.getWindow().addFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN);
```

### Recyclerview

Recyclerview 中使用属性动画时候，在动画结束时记得将使用的属性还原
RecyclerView 缓存的 view 个数默认为2，但是会增长，最大个数为4

```java
// 局部刷新
mStopWatchAdapter.notifyItemChanged(p, PAY_LOAD);
// 去除notifyItemChanged动画
((DefaultItemAnimator) mRecyclerView.getItemAnimator()).setSupportsChangeAnimations(false);  
```
### 源码编译 OutOfMemory

```shell
export JACK_SERVER_VM_ARGUMENTS="-Dfile.encoding=UTF-8 -XX:+TieredCompilation -Xmx4096m"

out/host/linux-x86/bin/jack-admin kill-server
out/host/linux-x86/bin/jack-admin start-server
```

### 设置圆角矩形的背景

在 drawable 文件夹中加入 drawable 的 xml

```xml
<shape xmlns:android="http://schemas.android.com/apk/res/android">;;
    <solid android:color="@color/tp_header_background_color" />
    <corners android:radius="20dp" />
</shape>
```

在代码中如下设置：

```java
Paint paint = new Paint();
paint.setAntiAlias(true);
Shader shader = new LinearGradient(bounds.centerX(), 
        bounds.centerY() - minDimension / 2, 
        bounds.centerX(), 
        bounds.centerY() + minDimension / 2, 
        mTopColor, 
        mBottomColor, 
        Shader.TileMode.MIRROR);
paint.setShader(shader);
canvas.drawCircle(bounds.centerX(), bounds.centerY(), minDimension / 2, paint);
```

### 阿拉伯语下字体对齐问题, 可以研究一下前端 css 样式

`android:fontFeatureSettings="tnum"`

### 保持唤醒的 Receiver

WakefulBroadcastReceiver

### 无背景水波纹效果

`android:background="?android:attr/selectableItemBackgroundBorderless"`

### HandlerThread

Handy class for starting a new thread that has a looper. The looper can then be 
used to create handler classes. Note that `start()` must still be called.

1. HandlerThread 将 loop 转到子线程中处理，说白了就是将分担 MainLooper 的工作量，降低了主线程的压力，使主界面更流畅。
2. 开启一个线程起到多个线程的作用。处理任务是串行执行，按消息发送顺序进行处理。HandlerThread 本质是一个线程，在线程内部，代码是串行处理的。
3. 但是由于每一个任务都将以队列的方式逐个被执行到，一旦队列中有某个任务执行时间过长，那么就会导致后续的任务都会被延迟处理。
4. HandlerThread 拥有自己的消息队列，它不会干扰或阻塞UI线程。
5. 对于网络 IO 操作，HandlerThread 并不适合，因为它只有一个线程，还得排队一个一个等着。

### 拍照

简单实现代码：

```java
Intent intent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
startActivityForResult(intent, CAPTURE_TWO);
```

在 `onActivityResult` 获取 Bitmap：

```java
    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {

        if (resultCode == RESULT_OK) {
            switch (requestCode) {
                case CAPTURE_ONE: {
                    if (data != null && data.getExtras() != null) {
                        Bundle extras = data.getExtras();
                        Bitmap imageBitmap = (Bitmap) extras.get("data");
                        btnFirst.setBmp(imageBitmap);
                        changeBindBtnStatus();
                        mBitmaps[0] = imageBitmap;
                    }
                }
                break;
            }
        }
        ...
        ...
    }
```

[参考资料](https://developer.android.google.cn/training/camera/photobasics.html)