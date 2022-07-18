---
title: "Notification 的使用"
date: 2018-01-26T08:55:17+08:00
draft: true
---

## Android 8.0 新特性
Android 8.0重新设计了通知，以便为管理通知行为和设置提供更轻松和更统一的方式。这些变更包括：

### 通知渠道
Android 8.0 引入了通知渠道特性，将应用的通知进行分门别类，用户可以针对不同的通知类别单独设置通知优先级别和提醒方式。用户界面将通知渠道称之为通知类别。

#### 1.创建通知渠道

```java
NotificationChannel channel = new NotificationChannel(channelId， name，
                importance);
channel.setShowBadge(canShowBadge);
channel.enableVibration(vibration);
channel.setSound(sound， null);
channel.enableLights(lights);
channel.setBypassDnd(dnd);
channel.setLockscreenVisibility(lockScreen);

getNotificationManager(context).createNotificationChannel(channel);
```
从方法上可以看出，渠道将之前由`Notification.Builder`设置的部分方法转移到`NotificationChannel`中了.

#### 2.读取通知渠道
用户可以设置通知渠道的行为，包括震动和声音。可以通过如下方法来读取用户对通知渠道的设置:
1. 通过`getNotificationChannel()`获取单个通知渠道;
2. 通过`getNotificationChannels()`方法获取应用所有的通知渠道。
通过上面的方法获取到`NotificationChannel`之后，可以通过`getVibrationPattern()`，`getSound()`等方法获取到用户当前的设置。要查看当前通知渠道是否被禁用，可以通过`getImportance()`方法，如果被禁用，这个方法会返回`IMPORTANCE_NONE`。

#### 3. 更新通知渠道
通知渠道创建之后，用户将负责其设置和行为。此时，可以通过调用`createNotificationChannel()`方法，然后再次将通知渠道提交到通知管理器来对已存在的通知渠道更名，或者更新它的内容。
通过下面的代码，你可以通过以创建新的Activity的方式跳转到通知渠道的设置界面。该方法需要这个渠道的ID和应用的包名
```java
Intent intent = new Intent(Settings.ACTION_CHANNEL_NOTIFICATION_SETTINGS);
intent.putExtra(Settings.EXTRA_CHANNEL_ID， mChannel.getId());
intent.putExtra(Settings.EXTRA_APP_PACKAGE， getPackageName());
startActivity(intent);
```
#### 4. 删除通知渠道
可以通过调用`deleteNotificationChannel()`方法来删除通知渠道:
```java
NotificationManager mNotificationManager =
        (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);
// The id of the channel.
String id = "my_channel_01";
mNotificationManager.deleteNotificationChannel(id);
```
>注意: 作为一种垃圾邮件预防机制，通知设置界面会显示被删除的渠道数量。

## 通知标志
Android 8.0 开始，应用可以在启动器图标上显示通知圆点来提示用户，但这个圆点角标和 iOS 上那个有所不同——它仅提示用户该应用有通知，不会显示具体的通知数量。
通知标志显示用户未操作或者关闭的,与一个或者多个通知渠道管理的通知.
用户可以在支持该特性的桌面通过长按应用图标查看与标志相关联的通知.

### 适配通知标志
默认情况下, 每个通知渠道都会在应用程序的启动徽章中反映其活动通知.可以使用setShowBadge()方法来阻止由标志反射的通道中的通知存在. 在通知渠道被创建并提交给通知管理器之后, 无法通过编程的方法修改通知渠道的设置.
>注意: 用户可以随时通过系统设置关闭通知渠道或者应用的标志.

下面的代码展示了如何从通知渠道隐藏标志与通知的关联:
```java
NotificationManager mNotificationManager =
        (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);
// The ID of the channel.
String id = "my_channel_01";
// The user visible name of the channel.
CharSequence name = getString(R.string.channel_name);
// The user visible description of the channel.
String description = getString(R.string.channel_description);
int importance = NotificationManager.IMPORTANCE_LOW;
NotificationChannel mChannel = new NotificationChannel(id, name, importance);
// Configure the notification channel.
mChannel.setDescription(description);
mChannel.setShowBadge(false);
mNotificationManager.createNotificationChannel(mChannel);
```
可以通过`setNumber()`自定义展示在长按菜单上面的通知数量, 下面的代码展示了一个消息应用的做法:
```java
mNotificationManager = (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);
int notificationID = 1;
String CHANNEL_ID = "my_channel_01";
// Set a message count to associate with this notification in the long-press menu.
int messageCount = 3;
// Create a notification and set a number to associate with it.
NotificationCompat notification =
        new NotificationCompat.Builder(MainActivity.this, CHANNEL_ID)
            .setContentTitle("New Messages")
            .setContentText("You've received 3 new messages.")
            .setSmallIcon(R.drawable.ic_notify_status)
            .setNumber(messageCount)
            .build();
// Issue the notification.
mNotificationManager.notify(notificationID, notification);
```
在可用的情况下,长按菜单会为关联通知显示或大或小的图标. 默认情况下, 系统会显示大的, 但是可以通过调用` NotificationCompat.Builder.setBadgeIconType()`方法, 传参数 `BADGE_ICON_SMALL` 来显示小图标
如果应用创建了一个有复制快捷键的通知, 可以在通知活动的时候暂时隐藏该快捷方式. 如果想这么做, 请调用`setShortcutId()`方法并传递快捷方式的ID.有关快捷键的更多消息,详见[APP shortcut](https://developer.android.google.cn/guide/topics/ui/shortcuts.html).

### 休眠
用户可以将通知置于休眠状态，以便稍后重新显示它。重新显示时通知的重要程度与首次显示时相同。应用可以移除或更新已休眠的通知，但更新休眠的通知并不会使其重新显示。

### 通知延后
通常，当一条通知出现在通知栏，除了点击查看、划掉不理以外，我们就只剩下「放任不管」这种处理方式了。这显然不太优雅，太多的通知驻留不仅会让通知栏拥挤不堪，回过头进行处理的时候也很不方便。所以，Android 8.0 引入了另一种通知处理操作——通知延后。当我们暂时不便处理某条应用通知时，只需要在该条通知上清扫，点击出现的时钟图标，即可让这条通知暂时从通知栏消失，在设定好的时间后再回来。

### 通知超时
现在，使用 setTimeoutAfter() 创建通知时您可以设置超时。您可以使用此函数指定一个持续时间，超过该持续时间后，通知应取消。如果需要，您可以在指定的超时持续时间之前取消通知。
###通知设置
当您使用 `Notification.INTENT_CATEGORY_NOTIFICATION_PREFERENCES` Intent 从通知创建指向应用通知设置的链接时，您可以调用 `setSettingsText()` 来设置要显示的文本。此系统可以提供以下 Extra 数据和 Intent，用于过滤应用必须向用户显示的设置：**EXTRA_CHANNEL_ID**、**NOTIFICATION_TAG** 和 **NOTIFICATION_ID**。

### 通知清除
系统现在可区分通知是由用户清除，还是由应用移除。要查看清除通知的方式，您应实现`NotificationListenerService`类的新`onNotificationRemoved()`函数。

### 背景颜色
您现在可以设置和启用通知的背景颜色。只能在用户必须一眼就能看到的持续任务的通知中使用此功能。例如，您可以为与驾车路线或正在进行的通话有关的通知设置背景颜色。您还可以使用`Notification.Builder.setColor()`设置所需的背景颜色。这样做将允许您使用`Notification.Builder.setColorized()`启用通知的背景颜色设置。

### 消息样式
现在，使用`MessagingStyle`类的通知可在其折叠形式中显示更多内容。对于与消息有关的通知，您应使用`MessagingStyle`类。您还可以使用新的`addHistoricMessage()`函数，通过向与消息相关的通知添加历史消息为会话提供上下文。

---

## 管理通知
当为同一类型的事件多次发出同一通知时候，应避免创建全新的通知，而是应该考虑通过更改之前通知的某些值和/或为其添加某些值来更新通知

### 更新通知
要将通知设置为能够更新，请通过调用 NotificationManager.notify() 发出带有通知 ID 的通知。 要在发出之后更新此通知，请更新或创建NotificationCompat.Builder 对象，从该对象构建 Notification 对象，并发出与之前所用 ID 相同的 Notification。如果之前的通知仍然可见，则系统会根据 Notification 对象的内容更新该通知。相反，如果之前的通知已被清除，系统则会创建一个新通知。

### 删除通知
除非发生以下情况之一，否则通知仍然可见：
1. 用户单独或通过使用“全部清除”清除了该通知（如果通知可以清除）。
2. 用户点击通知，且您在创建通知时调用了 setAutoCancel()。
3. 您针对特定的通知 ID 调用了 cancel()。此方法还会删除当前通知。
4. 您调用了 cancelAll() 方法，该方法将删除之前发出的所有通知。

### 浮动通知
对于 Android 5.0（API 级别 21），当设备处于活动状态时（即，设备未锁定且其屏幕已打开），通知可以显示在小型浮动窗口中（也称为“浮动通知”）。 这些通知看上去类似于精简版的通知​​，只是浮动通知还显示操作按钮。 用户可以在不离开当前应用的情况下处理或清除浮动通知。
可能触发浮动通知的条件示例包括：
1. 用户的 Activity 处于全屏模式中（应用使用 fullScreenIntent），或者
即使用 setFullScreenIntent(PendingIntent inteng, boolean highPriority. 当通知发出时候，在锁屏状态下，会加载一个页面来展示通知；当用户正在使用设备时候，SystemUI可能会选择展示浮动通知代替加载通知的界面，实测这种情况下展示的浮动通知如果不做操作，会一直停留在页面上，不会自动消失
2. 通知具有较高的优先级并使用铃声或振动
通过设置优先级和使用震动铃声的浮动通知需要用到两个方法：
setPriority(int priority) : priority 至少需要设置为 high. 
>注：该方法在 API26 中已废弃，新方法为 setImportance(int importance).

setDefaults(int defaults): 相应的参数为 DEFAULT_SOUND, DEFAULT_VIBRATE, DEFAULT_LIGHTS 和 DEFAULT_ALL.分别是为通知设置铃声，震动，指示灯或者三者全有。
>注：该方法已在API26中废弃，可以使用 enableVibration(boolean vibrate), enableLights(boolean lights), setSound(Uri uri, AudioAttributes attr) 代替

## 锁定屏幕通知
随着 Android 5.0（API 级别 21）的发布，通知现在还可显示在锁定屏幕上。您的应用可以使用此功能提供媒体播放控件以及其他常用操作。 用户可以通过“设置”选择是否将通知显示在锁定屏幕上，并且您可以指定您应用中的通知在锁定屏幕上是否可见。
### 设置可见性
您的应用可以控制在安全锁定屏幕上显示的通知中可见的详细级别。 调用 setVisibility() 并指定以下值之一：
>* VISIBILITY_PUBLIC 显示通知的完整内容。
>* VISIBILITY_SECRET 不会在锁定屏幕上显示此通知的任何部分。
>* VISIBILITY_PRIVATE 显示通知图标和内容标题等基本信息，但是隐藏通知的完整内容。

设置 VISIBILITY_PRIVATE 后，您还可以提供其中隐藏了某些详细信息的替换版本通知内容。例如，短信 应用可能会显示一条通知，指出“您有 3 条新短信”，但是隐藏了短信内容和发件人。要提供此替换版本的通知，请先使用 NotificationCompat.Builder 创建替换通知。创建专用通知对象时，请通过setPublicVersion() 方法为其附加替换通知。

## 自定义通知布局
通知框架允许自定义通知布局, 即通过`RemoteViews`对象来定义通知的外观.自定义布局的通知与普通通知类似,但是其依赖于XML布局文件定义的RemoteViews.
自定义通知布局的可用高度依赖于通知视图.普通视图布局的限制为64dp, 展开的视图布局限制为256dp.
###自定义视图
自Android 7.0 开始, 自定义通知视图可以适用系统装饰器,比如头部,动作和可展开的布局.
为支持这个功能,Android提供了下列API 来设计自定义视图:

1. `DecoratedCustomViewStyle()` 装饰媒体通知之外的通知
2. `DecoratedMediaCustomViewStyle()` 装饰媒体通知

使用方法如下:
```java
Notification notification = new Notification.Builder(context, CHANNEL_ID)
           .setSmallIcon(R.drawable.ic_stat_player)
           .setLargeIcon(albumArtBitmap))
           .setCustomContentView(contentView);
           .setStyle(new Notification.DecoratedCustomViewStyle())
           .build();
```

## 其它
### setDeleteIntent(PendingIntent intent)
该方法可在用户显式地清除通知时候发送一个PendingIntent，即可以在用户划掉通知时候做一些处理

### Chronometer
Chronometer是一个可以自动更新时间的控件。通过给它一个 elepsedRealtime() 基准的时间，它就可以以给定的时间计数，如果没有给出开始时间，它会以调用 start() 的时间为基准。
Chronometer 在API24 新增了倒数的功能，可以通过设置 `setCountDown(boolean)` 为 true来使用，相应的也可以在 `Notification.Builder` 中调用 `setChronometerCountDown(boolean countDown)` 来使用。