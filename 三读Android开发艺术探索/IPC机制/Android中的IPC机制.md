# Android中的IPC机制

IPC是Inter-Process Communication的缩写，含义是进程间的通信或者跨进程通信。进程对应移动设备来说可以简单的理解成为一个应用程序。

# Android中实现多进程模式

1. 通过android:process属性指定，可以以 ：开头；也可以全名称例如 com.txl.dem ；两者之间的区别是：开头的进程属于当前应用的私有进程，其他应用的组件不可以和他泡在一个进程中，而全名称进程属于全局进程，其他应用可以通过SharedUID的方式跑在同一个进程
2. 直接运行多个进程

使用process开启多进程的问题：

- 静态成员和单例模式失效
- 线程同步机制失效
- SharedPreferences的可靠性下降
- Application会多次创建

原因：每个进程有自己独立的虚拟机，不同的虚拟机在内存分配上有不同的地址空间

# 跨进程通信的方式

1. Bundle
2. 文件共享
3. Messenger
4. aidl
5. Socket
6. ContentProvider

