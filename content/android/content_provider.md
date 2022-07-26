---
title: "Content_provider"
date: 2018-04-19T08:18:27+08:00
draft: true
---

`ContentProvider` 是构建 Android 应用的重要组件之一, 它封装数据并通过简单的 `ContentResolver` 接口向应用提供数据. 仅在需要在多个应用间共享数据时候才需要实现 `ContentProvider`, 例如联系人应用.
![内容提供程序如何管理存储空间访问的概览图](https://developer.android.google.cn/static/guide/topics/providers/images/content-provider-overview.png)

`ContentProvider` 是一种标准接口, 能提供很好的抽象, 可以在修改修改应用数据存储实现的同时不影响访问数据的其它应用. 例如, 可以将 SQLite 数据库换成其它存储空间
![迁移内容提供程序存储空间的图示意图](https://developer.android.google.cn/static/guide/topics/providers/images/content-provider-migration.png)

`ContentProvider` 是应用程序之间共享数据的接口. 使用的时候首先自定义一个继承 `ContentProvider` 的子类, 然后覆写 `query`, `insert`, `update`, `delete` 等方法. 因为是四大组件之一, 因此必须在 `AndroidManifest` 文件中进行注册, 将自己的数据以 `URI` 的方式共享出去,  第三方应用可以通过`ContentResolver` 来访问.

底层也是由 binder 实现

## URI介绍
每一个 `ContentProvider` 都有一个公共的 `URI`, 只能通过访问该 `URI` 来访问数据
Android提供的 `ContentProvider` 都存放在 `android.provider` 包中, 将其分为A, B, C, D四个部分:
1. A: 标准前缀, 用来说明一个 ContentProvider 控制这些数据, 无法改变: "content://"
2. B: URI 的标识, 用于唯一标示这个 ContentProvider, 外部调用者可以通过这个标识来找到它. 它定义了是哪个 ContentProvider 提供这些数据.对于第三方应用, 为了保证 URI 的唯一性, 它必须是一个完整的, 小写的类名, 这个标识在元素的 `authorities` 属性中说明: 一般是定义该 ContentProvider 的包.类名称
3. C: 路径(path), 通俗的讲就是你要操作的数据库中表的名字, 或者你也可以自定义, 记得在使用的时候保持一致就可以了, 例如: "content://com.bing.provider.myprovider/tablename"
4. D: 如果URI包含表示需要获取的记录的 id, 则就返回该 id 对应的数据, 如果没有 id, 就表示返回全部; 例如: "content://com.bing.provider.myprovider/tablename/#", # 表示 id;

---
参考: [ContentProvider](https://developer.android.google.cn/guide/topics/providers/content-providers)