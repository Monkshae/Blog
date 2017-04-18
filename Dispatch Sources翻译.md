#Dispatch Sources

每当你与底层系统进行交互的时候，你必须花费大量的时间为此任务做好准备。调用内核或者其他系统层设计上下文的更改，与您自己的进程相比，代价相当昂贵。因此，很多系统库提供了接口，允许你想系统提交请求，并在处理该请求时继续执行其他工作。GCD正是在这个基础上构建的，允许您提交请求并通过block或者dispatch queue回调执行结果。

#关于Dispatch Sources
调度源是协调特定的低级系统处理的基本数据类型。GCD支持下面几种类型:

* 定时器调度源产生定期通知。
* 信号调度源在UNIX信号到达时通知您。
* 描述符来源通知您各种基于文件和套接字的操作，如：
	* 当数据可用于阅读时
	* 当可以写入数据时
	* 在文件系统中删除，移动或重命名文件时
	* 当文件元信息更改时
* 流程调度源通知您进程相关事件，如：
	* 进程退出时
	* 当进程发出fork或exec类型的调用
	* 当一个信号传送到该过程
* Mach port调度源通知您Mach相关事件
*自定义dispatch sources是您定义并触发自己

http://geek.csdn.net/news/detail/69122
http://www.tuicool.com/articles/JN36BnY
https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/GCDWorkQueues/GCDWorkQueues.html#//apple_ref/doc/uid/TP40008091-CH103-SW1
http://www.jianshu.com/p/880c2f9301b6


