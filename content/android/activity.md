---
title: "Android 四大组件之 Activity"
date: 2018-05-05T10:22:32+08:00
draft: true
---
Activity 是一个应用组件, 用户可与其提供的屏幕进行交互, 以执行拨打电话, 拍摄照片, 发送电子邮件或查看地图等操作. 每个 Activity 都会获得一个用于绘制其用户界面的窗口. 通常会充满屏幕, 但也可以 **小于屏幕并浮动在其他窗口之上**.

## Activity 生命周期
![Activity 生命周期][1]

### onCreate()
表示 Activity 正在被创建, 这是生命周期的第一个方法. 在这个方法中, 我们可以做一些初始化工作, 比如调用 setContentView() 去加载界面资源, 初始化 Activity 所需数据等.

### onRestart()
表示 Activity 正在重新启动. 一般情况下, 当当前 Activity 从不可见重新变为可见状态时, onRestart() 会被调用. 这种情况一般是用户行为所导致的, 比如用户按 Home 键切换到桌面状态或者用户打开了一个新的 Activity, 这时当前的 Activity 就会暂停, 也就是 onPause() 和 onStop() 执行了, 接着用户又回到了这个 Activity, 就会出现这种情况.

### onStart()
表示 Activity 正在被启动, 即将开始, 这时 Activity 已经可见了, 但是还没有出现在前台, 还无法和用户交互. 这时其实可以理解为 Activity 已经显示出来了, 但是我们还看不到.

### onResume()
表示 Activity 已经可见了, 并且出现在活动前台开始活动. 要注意这个和 onStart() 的区别, 它们都表示 Activity 可见, 但是 onStart() 时候 Activity 还在后台, onResume() 时候 Activity 已经在前台.

### onPause()
表示 Activity 正在停止, 正常情况下, 接着 onStop() 就会被调用. 在特殊情况下, 如果这个时候快速地再回到当前 Activity, 那么 onResume() 就会被调用. 这种情况属于极端情况, 用户操作很难复现这一场景. 此时可以做一些存储数据, 停止动画等工作, 但是注意不能太耗时, 因为这会影响到新 Activity 的显示, onPause() 必须先执行完, 下一个 Activity 才能继续执行.

### onStop()
表示 Activity 即将停止, 可以做一些稍微重量级的回收工作, 同样不能太耗时. 如果 Activity 恢复与用户的交互, 则后接 onRestart(), 如果 Activity 被销毁, 则后接 onDestroy().

### onDestroy()
表示 Activity 即将被销毁, 这是 Activity 生命周期中的最后一个回调, 在这里, 我们可以做一些回收工作和最终的资源释放.

### onSaveInstanceState()
当系统配置发生改变时候, Activity 会被销毁, 其 onPause(), onStop(), onDestroy() 均会被调用, 同时由于 Activity 是在异常状态下被终止的, 系统会调用 onSaveInstanceState() 来保持当前 Activity 的状态. 这个方法的调用是在 onStop 之前, 它和 onPause() 没有既定的时序关系, 它即可能是在 onPause() 之前调用, 也有可能是在 onPause() 之后调用. 这种方法只会在 Activity 被异常终止的情况下调用, 正常情况下系统不会调用这个方法. 当 Activity 被重新创建后, 系统会调用 onRestoreInstanceState(), 并把 onSaveInstanceState() 方法所保存的 Bundle 对象作为参数同时传递给 onRestoreInstanceState() 和 onCreate().

即使什么都不做, 也不实现 onSaveInstanceState(), Activity 类的 onSaveInstanceState() 默认实现也会恢复部分 Activity 状态. 具体地讲, 默认实现会为布局中每个 View 调用相应的 onSaveInstanceState() 方法, 让每个视图都能提供有关自身的应保存信息.

Android 框架中几乎每个小部件都会根据需要实现此方法, 以便在重建 Activity 时自动保持和恢复对 UI 所做的任何可见更改. 例如, EditText 小部件保存用户输入的任何文本, CheckBox 保存复选框的选中或未选中状态. 只需要为想要保存状态的每个小部件提供一个唯一的 ID. 如果小部件没有 ID, 则系统无缝保存其状态.

还可以通过将 android:saveEnabled 属性设置为 'false' 或者调用 setSaveEnabled() 方法显式阻止布局内的视图保存其状态
>* 注意: 当用户显式关闭 Activity 时，或者在其他情况下调用 `finish()` 时，系统不会调用 `onSaveInstanceState()`。

### 使用保存的实例状态恢复 Activity 界面状态
重建先前被销毁的 Activity 后，可以从系统传递给 Activity 的 Bundle 中恢复保存的实例状态。`onCreate()` 和 `onRestoreInstanceState()` 回调方法均会收到包含实例状态信息的相同 Bundle。

因为无论系统是新建 Activity 实例还是重新创建之前的实例，都会调用 `onCreate()` 方法，所以在尝试读取之前，必须检查状态 Bundle 是否为 null。如果为 null，系统将新建 Activity 实例，而不会恢复之前销毁的实例。

例如，以下代码段显示如何在 `onCreate()` 中恢复某些状态数据：
```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState); // Always call the superclass first

    // Check whether we're recreating a previously destroyed instance
    if (savedInstanceState != null) {
        // Restore value of members from saved state
        currentScore = savedInstanceState.getInt(STATE_SCORE);
        currentLevel = savedInstanceState.getInt(STATE_LEVEL);
    } else {
        // Probably initialize members with default values for a new instance
    }
    // ...
}
```
可以选择实现系统在 onStart() 方法之后调用的 onRestoreInstanceState()，而不是在 onCreate() 期间恢复状态。仅当存在要恢复的已保存状态时，系统才会调用 onRestoreInstanceState()，因此您无需检查 Bundle 是否为 null：
```java
public void onRestoreInstanceState(Bundle savedInstanceState) {
    // Always call the superclass so it can restore the view hierarchy
    super.onRestoreInstanceState(savedInstanceState);

    // Restore state members from saved instance
    currentScore = savedInstanceState.getInt(STATE_SCORE);
    currentLevel = savedInstanceState.getInt(STATE_LEVEL);
}
```
>* 注意：应始终调用 `onRestoreInstanceState()` 的父类实现，以便默认实现可以恢复视图层次结构的状态。 

### 协调 Activity
当一个 Activity 启动另一个 Activity 时，它们都会经历生命周期转换。第一个 Activity 停止运行并进入“已暂停”或“已停止”状态，同时创建另一个 Activity。如果这些 Activity 共享保存到磁盘或其他位置的数据，必须要明确第一个 Activity 在创建第二个 Activity 之前并未完全停止。相反，启动第二个 Activity 的过程与停止第一个 Activity 的过程重叠。

生命周期回调的顺序已有明确定义，特别是当两个 Activity 在同一个进程（应用）中，并且其中一个要启动另一个时。以下是 Activity A 启动 Activity B 时的操作发生顺序：
1. Activity A 的 onPause() 方法执行。
2. Activity B 的 onCreate()、onStart() 和 onResume() 方法依次执行（Activity B 现在具有用户焦点）。
3. 然后，如果 Activity A 在屏幕上不再显示，其 onStop() 方法执行。

## Activity 的启动模式
目前有四种启动模式: standard, singleTop, singleTask, singleInstance

### standard
标准模式, 也是系统的默认模式. 每次启动一个 Activity 都会重新创建一个新的实例, 不管这个实例是否已经存在. 在这种模式下, 谁启动了这个 Activity, 那么这个 Activity 就会运行在启动它的那个 Activity 所在的栈中.

当使用 ApplicationContext 启动 Activity 时候会报错. 这是因为 standard 模式的 Activity 默认会进入启动它的 Activity 所属的栈中, 但是由于非 Activity 类型的 Context 没有所谓的任务栈, 所以这就有问题了. 解决这个问题的方法就是为待启动的 Activity 指定 FLAG_ACTIVITY_NEW_TASK 标记位, 这样启动时候就会为它创建一个新的任务栈, 这个时候待启动的 Activity 实际上是以 singleTask 模式启动的.

### singleTop
栈顶复用模式. 在这种模式下, 如果新 Activity 已经位于任务栈的栈顶, 那么此 Activity 不会被重新创建, 同时它的 onNewIntent() 方法会被调用, 通过此方法的参数我们可以获得当前请求的信息. 需要注意的是这个 Activity 的 onCreate(), onStart() 不会被调用, 因为并没有重新创建它.

### singleTask
站内复用模式. 这是一种单实例模式, 这种情况下, 只要 Activity 在一个栈中存在, 那么多次启动此 Activity 都不会重新创建实例, 和 singleTop 一样, 系统也会回调其 onNewIntent().

### singleInstance
单实例模式. 这是一种加强的 singleTask 模式, 它具有 singleTask 模式的所有特性外, 还加强了一点, 那就是具有此种模式的 Activity 只能单独地位于一个任务栈中. 通常用于需要与应用分离的 Activity, 比如闹钟的响铃界面.

### TaskAffinity
TaskAffinity 主要和 singleTask 启动模式或者 allowTaskReparenting 属性配对使用, 在其它情况下没有意义.

当 TaskAffinity 和 singleTask 启动模式配对使用的时候, 它是具有该模式的 Activity 当前任务栈的名字, 待启动的 Activity 会运行在名字和 TaskAffinity 相同的任务栈中.

当 TaskAffinity 和 allowTaskReparenting 属性结合的时候, 这种情况比较特殊, 会产生特殊的效果. 当一个应用 A 启动了应用 B 的某个 Activity 后, 如果这个 Activity 的 allowTaskReparenting 属性为 true 的话, 那么当 B 应用被启动后, 此 Activity 会直接从应用 A 的任务栈转移到应用 B 的任务栈中

### 如何给 Activity 设置启动模式
第一种, 通过 AndroidManifest 为 Activity 指定启动模式:
```xml
<activity
    android:name=".main.MainActivity"
    android:windowSoftInputMode="adjustResize"
    android:theme="@style/AppTheme.Splash"
    android:launchMode="singleTask"/>
```

另一种情况是通过在 Intent 中设置标志位为 Activity 指定启动模式:
```Java
Intent intent = new Intent();
intent.setClass(this, SecondActivity.class);
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
startActivity(intent);
```

这两种方式都可以为 Activity 指定启动模式, 但是两者之间还是有区别的. 首先, 优先级上, 第二种方式的优先级要高于第一种, 当两者同时存在时候, 以第二种方式为准; 其次, 上述两种方式在限定范围上有所不同, 第一种方式无法直接为 Activity 设定 FLAG_ACTIVITY_CLEAR_TOP 标识, 而第二种方式无法为 Activity 指定 singleInstance 模式.


[1]: https://developer.android.google.cn/images/activity_lifecycle.png "Activity 生命周期"

参考资料: [Activity](https://developer.android.google.cn/guide/components/activities), Android 开发艺术探索