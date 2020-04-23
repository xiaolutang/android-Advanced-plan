小王今天再次看了《Android进阶解密》的根Activity这一章节。发现上次看过的内容又已经忘记的差不多了。总结了看书存在的问题，有些知识点不方便使用代码进行训练。如果自己不进行总结十分容易再次忘记。就像没看过一样。于是小王总结了下面的学习笔记。

具体的启动流程可以自行在网上搜索。

在书中根Activity启动这一小节中作者将启动过程分为三个部分

1. Luncher请求ActivityManagerService过程
2. ActivityManagerService到ApplicationThread的调用过程
3. ActivityThread启动Activity的过程

在这几个过程中我们需要关注的点在于：

1. Luncher启动应用程序的的流程除了启动参数不同，其它的位置和我们正常启动一个Activity的流程相同，最终都会通过Activity#startActivityForResult()调用Instrumentation#execStartActivity方法借助binder跨进程请求ActivityManagerServer启动Activity。
2. 代码在进入到ActivityManagerServeice后,由于根Activity的创建时，应用的进程没有创建，在ActivityStackSupervisor#startSpecificActivityLocked方法中不会执行到启动activity的逻辑。而是调用ActivityManagerService#startProcessLocked先创建相对应的进程。
3. 进程创建好之后会进入到ActivityThread#main方法。调用ActivityThread#attach最后再次回到ActivityManagerService。在ActivityManagerService#attachApplication中经过层层调用最后借助ActivityThread#ApplicationThread来启动activity
4. 整个过程有4个进程参与：分别是 Launcher,SystemServer,Zygote,应用程序。其中ActivityManagerService存在于SystemServer,Zygote用于创建应用程序进程。
5. Instrumentation 对象需要我们稍微注意一下，它的只要职责是用来监控应用程序和系统的交互。



笔记参考：《Android进阶解密 》 根activity的启动过程

博客：[https://blog.csdn.net/Luoshengyang/article/details/6689748?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522158765918419726867820989%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=158765918419726867820989&biz_id=0&utm_source=distribute.pc_search_result.none-task-blog-2~blog~first_rank_v2~rank_v25-1](https://blog.csdn.net/Luoshengyang/article/details/6689748?ops_request_misc=%7B%22request%5Fid%22%3A%22158765918419726867820989%22%2C%22scm%22%3A%2220140713.130102334.pc%5Fblog.%22%7D&request_id=158765918419726867820989&biz_id=0&utm_source=distribute.pc_search_result.none-task-blog-2~blog~first_rank_v2~rank_v25-1)