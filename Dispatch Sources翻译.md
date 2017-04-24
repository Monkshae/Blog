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

dispatch sources代替通常用于处理系统相关事件的异步回调函数,当你配置调度源时，可以指定要监视的事件以及用于处理这些事件的调度队列和代码。您可以使用block或函数指定代码，当你感兴趣的事件到达的时候，调度源将您的block或者函数提交到指定的的调度队列执行。

与您手动提交到队列的任务不同，调度源为应用程序提供了连续的事件源。dispatch source将一直绑定在dispatch quene上，直到您明确的取消它。一旦绑定成功，只要相应的事件发生，就会将相关的的任务代码提交到对应的dispatch queue。某些事件比如timer事件定时发生，视具体情况而定，大多数情况下偶尔发生。因此，调度源保留其相关的调度队列，以防止在事件可能仍处于待处理状态时过早地释放它们。

为了防止时间在调度队列中积压,调度源实现了事件合并方案。在旧事件的事件处理句柄已经出队列并执行之前，新事件已经到达了,dispatch source 会将新的事件数据和旧的事件数据进行合并。根据事件的类型，合并可能会代替旧事件或者更新了旧事件持有的信息。例如，基于信号的调度源提供仅关于最近信号的信息，但是还报告自上次调用事件处理器以来已经传送了多少总信号。

#Creating Dispatch Sources
创建一个Dispatch source的同时，会涉及到事件的来源和dispatch source本身的创建。事件的来源是处理事件所需要的任何本地的数据结构。例如，对于基于描述符的调度源，您需要打开描述符。对于基于进程的源，您需要获取目标程序的进程ID。当您有事件来源时，您可以按如下所示创建相应的调度源：

1.使用dispatch_source_create函数创建一个dispatch source
2.配置调度来源:

* 分配一个事件处理器给事件来源;见[ Writing and Installing an Event Handler.](https://goo.gl/uYpDRJ)
* 对于时间源，使用 [dispatch_source_set_timer](https://goo.gl/vyC6y5)设置timer信息,见[Creating a Timer](https://goo.gl/41oSxO)

3.可以可选的分配一个取消事件处理器给dispatch source,见[Installing a Cancellation Handler](https://goo.gl/NBsPXX)
4.调用dispatch_resume函数开始处理时间;见[Suspending and Resuming Dispatch Sources](https://goo.gl/rMAzqS)

因为dispatch source需要一些额外的配置才能使用，dispatch_source_create函数返回的调度源，将会处于挂起状态。挂起状态时，调度源接受事件但是并不处理它们。这给了你时间去安装事件处理器并处理实际事件所需的额外配置。

以下部分将介绍如何配置调度源的各个方面，有关如何配置特定类型的调度源的详细实例，见[Dispatch Source Examples.](https://goo.gl/FYa9gc)
有关用于创建和配置调度源的功能的其他信息,见`Grand Central Dispatch (GCD) Reference`

#编写和安装一个事件处理器






http://geek.csdn.net/news/detail/69122
http://www.tuicool.com/articles/JN36BnY
https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/GCDWorkQueues/GCDWorkQueues.html#//apple_ref/doc/uid/TP40008091-CH103-SW1
http://www.jianshu.com/p/880c2f9301b6


