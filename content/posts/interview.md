---
title: "面试"
date: 2022-07-14T22:11:12+08:00
draft: true
_build:
    list: false
    render: false
---

## Java 基础

### == 和 equals 的区别, Object还有什么方法?
1. == 是操作符, equals() 是方法; equals() 方法可以被重写, == 不可以;
2. == 比较的是值对象(int, char, double等), equals 是 Object 的方法, 默认情况下使用的是 ==
3. equals() 方法是提供给开发者来重写的, 由开发者定义两个对象在什么情况下相等

### 谈谈对多态的理解
1. 定义: 是指不同类的对象对同一方法做出不同的响应
2. 实现方式:1. 继承父类, 方法重写; 同一个类中进行方法重载
3. 存在的三个必要条件: 
    * 要有继承; 
    * 要有重写;
    * 父类引用指向子类对象
4. 多态的好处：
    * 可替换性（substitutability）。多态对已存在代码具有可替换性。例如，多态对圆Circle类工作，对其他任何圆形几何体，如圆环，也同样工作。
    * 可扩充性（extensibility）。多态对代码具有可扩充性。增加新的子类不影响已存在类的多态性、继承性，以及其他特性的运行和操作。实际上新加子类更容易获得多态功能。例如，在实现了圆锥、半圆锥以及半球体的多态基础上，很容易增添球体类的多态性。
    * 接口性（interface-ability）。多态是超类通过方法签名，向子类提供了一个共同接口，由子类来完善或者覆盖它而实现的。如图8.3 所示。图中超类Shape规定了两个实现多态的接口方法，computeArea()以及computeVolume()。子类，如Circle和Sphere为了实现多态，完善或者覆盖这两个接口方法。
    * 灵活性（flexibility）。它在应用中体现了灵活多样的操作，提高了使用效率。
    * 简化性（simplicity）。多态简化对应用软件的代码编写和修改过程，尤其在处理大量对象的运算和操作时，这个特点尤为突出和重要。

Java中多态的实现方式：接口实现，继承父类进行方法重写，同一个类中进行方法重载

### 使用 Searilizable 有什么需要注意的地方, 混淆是否会影响序列化?
1. 实现 serializable 接口而付出的最大的代价就是, 一旦一个类被发布, 就大大降低了 改变这个类的实现的灵活性.
2. 第二个代价是, 它增加了出现 Bug 和安全性漏洞的可能性
3. 第三个代价是, 随着类发行新的版本, 相关测试的负担更重

### 内存泄漏
* 已申请的内存在不使用以后仍然无法释放, 导致程序占用内存一直升高
* 解决方法: 1. 内部类尽量不要持有外部类的对象, 2. 内部类尽量使用静态的, 3. 使用软引用

### sleep和wait有什么区别
#### 第一种解释：
功能差不多,都用来进行线程控制,他们最大本质的区别是: sleep() 不释放同步锁, wait() 释放同步缩.   
    
  还有用法的上的不同是: sleep(milliseconds) 可以用时间指定来使他自动醒过来,如果时间不到你只能调用 interreput() 来强行打断; wait() 可以用 notify() 直接唤起.

#### 第二种解释：
sleep是Thread类的静态方法。sleep的作用是让线程休眠制定的时间，在时间到达时恢复，也就是说sleep将在接到时间到达事件事恢复线程执行，例如：
```java
try {
    System.out.println("I'm going to bed");
    Thread.sleep(1000);
    System.out.println("I wake up");
} catch(IntrruptedException e) {
}
```
wait是Object的方法，也就是说可以对任意一个对象调用wait方法，调用wait方法将会将调用者的线程挂起，直到其他线程调用同一个对象的notify方法才会重新激活调用者，例如：
```java
// Thread 1
try {
    obj.wait();     // suspend thread until obj.notify() is called
} catch(InterrputedException e) {
}
```

#### 第三种解释：
这两者的施加者是有本质区别的. 
sleep()是让某个线程暂停运行一段时间,其控制范围是由当前线程决定,也就是说,在线程里面决定.好比如说,我要做的事情是 "点火->烧水->煮面",而当我点完火之后我不立即烧水,我要休息一段时间再烧.对于运行的主动权是由我的流程来控制.

而wait(),首先,这是由某个确定的对象来调用的,将这个对象理解成一个传话的人,当这个人在某个线程里面说"暂停!",也是 `thisOBJ.wait()`,这里的暂停是阻塞,还是"点火->烧水->煮饭",`thisOBJ`就好比一个监督我的人站在我旁边,本来该线程应该执行1后执行2,再执行3,而在2处被那个对象喊暂停,那么我就会一直等在这里而不执行3,但正个流程并没有结束,我一直想去煮饭,但还没被允许,直到那个对象在某个地方说"通知暂停的线程启动!",也就是 `thisOBJ.notify()` 的时候,那么我就可以煮饭了,这个被暂停的线程就会从暂停处 继续执行.

其实两者都可以让线程暂停一段时间,但是本质的区别是一个线程的运行状态控制,一个是线程之间的通讯的问题
在 `java.lang.Thread` 类中，提供了 `sleep()`，
而 `java.lang.Object` 类中提供了 `wait()`， `notify()` 和 `notifyAll()` 方法来操作线程
sleep()可以将一个线程睡眠，参数可以指定一个时间。
而wait()可以将一个线程挂起，直到超时或者该线程被唤醒。
    wait有两种形式wait()和wait(milliseconds).
#### sleep 和 wait 的区别有：
1. 这两个方法来自不同的类分别是`Thread`和`Object`
2. 最主要是`sleep`方法没有释放锁，而`wait`方法释放了锁，使得其他线程可以使用同步控制块或者方法。
3. `wait`，`notify` 和 `notifyAll` 只能在同步控制方法或者同步控制块里面使用，而 `sleep` 可以在
    任何地方使用
```java
   synchronized(x) {
      x.notify()
     //或者wait()
   }
```
4. `sleep`必须捕获异常, 而`wait`，`notify`和`notifyAll`不需要捕获异常

### 什么是哈希表？
散列表（Hash table, 也叫哈希表), 是根据关键码值(Key value)而直接进行访问的数据结构.

### TODO
1. Java虚拟机的内存划分
2. Java 内存模型
5. 面向对象的理解, 多态, 继承,封装
1. Java多线程

## 数据结构

### 什么是链表, 链表的查询复杂度是多少, 如何优化
* 链表是一种递归的数据结构, 它或者为空, 或者是指向一个节点的引用, 该节点包含一个泛型的元素和一个指向另一个链表的引用
* 查询复杂度为 O(N)
* 优化: 考虑使用`跳表`

### SparseArray 和 ArrayMap 的数据结构是怎样的?
都是用两个数组分别存放其键值对, 查找使用二分查找, 所以在数据量较大的时候速度会比较慢
1. 如果key的类型已经确定为int类型，那么使用SparseArray，因为它避免了自动装箱的过程，如果key为long类型，它还提供了一个LongSparseArray来确保key为long类型时的使用
2. 如果key类型为其它的类型，则使用ArrayMap

## Android

### Activity 启动模式
* Standard 标准模式, 每次启动都会创建一个 Activity 对象;
* SingleTop 如果 Activity 对象位于回退栈顶, 那么再次启动不会创建新对象, 如果不位于栈顶, 则会创建新的对象
* SingleTask 如果 Activity 位于回退栈中, 那么重新启动 Activity 不会创建对象, 如果不位于栈顶, 则会把它上面的 Activity 顶掉
* SingleInstance Activity 位于独立的栈中

### Android 内存分配机制
Android 内存分为内核内存和应用内存, 应用内存是弹性分配的, 在超过分给它的初始容量以后, 会再次增大分配

### Android给应用的运行内存由多大
不同的手机数据可能不一样，自测小米的是192m

### Acticity和Service是否在同一个线程工作
这个问题有坑，没有绝对条件可以判定是不是在一个线程工作 
同一个包内的activity和service
1. 如果service没有设定属性android:process=”:remote”的话，service会和activity跑在同一个进程中，由于当前这个进程只有一个UI线程，所以，service和acitivity就是在同一个线程里面的； 
2. 如果设定了属性android:process=”:remote”的话，那么就是跨进程访问，但是跨进程访问有可能出现各种各样的问题

### Application 和 Activity 的 Context 对象的区别?
生命周期的差别, Application 的会存在整个应用的生命周期, Activity 只会存在这个 Activity 

### AlertDialog 和 PopupWindow 的区别
AlertDialog 是非阻塞式对话框, 弹出后, 后台还可以做事情;
PopupWindow 是阻塞式对话框, 弹出时, 程序会等待, 在 PopupWindow 退出前, 程序一直等待, 只有调用了 dismiss 方法后, PopupWindow 退出, 程序才会向下执行

### Android 中 main 方法入口在哪?
ActivityThread 的 main 方法.
参考地址:https://www.zhihu.com/question/30648827

### RecyclerView 有什么地方可以进行优化?
可以使用局部刷新

### onPause, onStop 写入数据会有什么问题?
1. 可以在onPause 时候可以写入一些持久化数据, 或者停止动画之类, 但是这些动作必须尽可能快的进行, 因为如果它没有返回的话, 另一个 Activity 就不会进入 onResume 状态. 比如一个 Activity 启动 另一个 Activity 时候, 第一个 Activity 的 onPause 执行之后才会开始启动另一个 Activity.
2. onStop 在低内存时候可能并不会被执行.

### Android主线程是安全的吗?
答: 不安全. 是因为性能考虑, 线程安全性能不好, 线程不安全性能不好.
Android为了性能选择了性能不安全, 但是为了保证安全, 设计主线程为单线程, 这样就不会由安全问题了.

UI线程的作用是什么呢?就是刷新界面对吧?对,它是通过Android的invalidate()方法去刷新的,,但是invalidate不能再非UI线程去调用,(原理就是你通过其他线程去调用这个方法,而UI线程也在调用这个方法,所以就会导致线程不安全了,而且在Android里面,这样做是不被允许的)所以上结论:Android UI是通过invalidate()方法去刷新界面的,而invalidate()方法是线程不安全的,所以UI线程就就是不安全的.

作者：知乎用户
链接：https://www.zhihu.com/question/36283467/answer/75692448
来源：知乎

### Dalvik虚拟机与java虚拟机的区别
1. java虚拟机运行的是Java字节码，Dalvik虚拟机运行的是Dalvik字节码；传统的Java程序经过编译，生成Java字节码保存在class文件中，java虚拟机通过解码class文件中的内容来运行程序。而Dalvik虚拟机运行的是Dalvik字节码，所有的Dalvik字节码由Java字节码转换而来，并被打包到一个DEX(Dalvik Executable)可执行文件中Dalvik虚拟机通过解释Dex文件来执行这些字节码。
2. Dalvik可执行文件体积更小。SDK中有一个叫dx的工具负责将java字节码转换为Dalvik字节码。
3. java虚拟机与Dalvik虚拟机架构不同。java虚拟机基于栈架构。程序在运行时虚拟机需要频繁的从栈上读取或写入数据。这过程需要更多的指令分派与内存访问次数，会耗费不少CPU时间，对于像手机设备资源有限的设备来说，这是相当大的一笔开销。Dalvik虚拟机基于寄存器架构，数据的访问通过寄存器间直接传递，这样的访问方式比基于栈方式快的多.
>* 注: Android 自 5.0 开始后, ART(Android Runtime) 虚拟机正式替代 Dalvik,  其特点是在安装时候就将应用程序转换为机器语言, 速度比较快;  比较明显的缺点就是，apk经过dex2oat预编译之后，占用的空间增加，因此Android ROM占用的空间更大。手机在安装下载的apk时，安装时间也明显变长。

### View
1. `getWidth()`和`getMeasuredWidth()`的区别
`getMeasuredWidth`：只要一执行完`setMeasuredDimension()`方法，就有值了，并且不再改变
`getWidth()`:必须执行完`onMeasure()`才有值，可能发生改变。
如果onLayout没有对子View的实际宽高进行修改，那么getWidth()的值 == getMeasuredWidth()的值

### 事件分发中的 onTouch 和 onTouchEvent 有什么区别
1. onTouchListener 的 onTouch 方法优先级比 onTouchEvent 高, 会先触发;
2. 假如 onTouch 方法返回 false 会接着触发 onTouchEvent, 反之onTouchEvent 不会被触发;
3. 内置诸如 onClick 事件等的实现都基于 onTouchEvent, 假如 onTouch 返回true, 这些事件将不会被触发;

### TODO
0. Looper是死循环, 为什么没有导致主线程ANR
1. Handler会持有activity的引用吗, 延迟发送的消息,如果此时Activity被销毁, 是否会产生内存泄漏
1. Android 四大组件的理解, 如何跨进程通信. 四大组件都可以进行跨进程通信, 此外还有 AIDL, Socket, 共享文件等
2. 请给出跨进程复制另一个进程中数据的方法(只知道表的访问地址, 不知道表的结构)
3. 请给出跨进程复制另一个进程中文件的方法
4. 如果解决线程池满载时候, 丢失线程的问题
5. 请给出有时效性网络缓存的方法
7. 请给出 Android 异步消息处理机制
8. 一个后台服务会接收多个前台UI调用, 请设计一个异步处理方案
9. 常用图片加载框架的区别及原理
4. 图片加载框架原理, 常用第三方框架原理
6. 应用框架理解
8. View 的绘制流程, Activity 的启动流程
9. APP 的构建流程, 是否了解过 gradle
10. 使用 Bundle 传数据有什么限制. 有最大 1MB 数据的限制
11. ContentProvider 的原理
12. 事件分发的 dispatch 方法返回值有什么区别?
17. Android 如何捕获未捕获的异常
18. Framework 工作方式及原理
19. Activity 如何生成 view

## 设计模式
2. 设计模式理解
(工厂模式, 适配器模式, 建造者模式)


## 网络

### 简单介绍一下 HTTPS

[参考文章](https://blog.csdn.net/liupeifeng3514/article/details/79840274)

### http 和 https 的区别
https 是 http 的安全版，即 http 下加入 SSL（Secure Socket Layer), TLS 对传输内容进行加密

### TODO
1. TCP 流的使用, 如何判断传输的完整性?
2. 是否有使用过单元测试
3. 谈谈对应用框架的理解
4. HTTP理解, TCP/UDP 的区别
6. 请给出 socket 的使用和理解