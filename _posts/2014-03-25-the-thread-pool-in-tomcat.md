---
layout: post
title: "The Thread Pool in Tomcat"
category: 
tags: [tomcat, thread pool, concurrect]
---

*摘自<http://blog.csdn.net/hulefei29/article/details/3849150>*

目前市场上常用的开源Java Web容器有[Tomcat](http://tomcat.apache.org/)、[Resin](http://www.caucho.com/)和[Jetty](http://jetty.mortbay.org/)。其中Resin从V3.0后需要购买才能用于商业目的，而其他两种则是纯开源的。可以分别从他们的网站上下载最新的二进制包和源代码。

作为Web容器，需要承受较高的访问量，能够同时响应不同用户的请求，能够在恶劣环境下保持较高的稳定性和健壮性。在HTTP服务器领域，[Apache HTTPD](http://httpd.apache.org/)的效率是最高的，也是最为稳定的，但它只能处理静态页面的请求，如果需要支持动态页面请求，则必须安装相应的插件，比如mod_perl可以处理Perl脚本，mod_python可以处理Python脚本。

上面介绍的三中Web容器，都是使用Java编写的HTTP服务器，当然他们都可以嵌到Apache中使用，也可以独立使用。分析它们处理客户请求的方法有助于了解Java多线程和线程池的实现方法，为设计强大的多线程服务器打好基础。

## Tomcat

Tomcat是使用最广的Java Web容器，功能强大，可扩展性强。最新版本的Tomcat（5.5.17）为了提高响应速度和效率，使用了[Apache Portable Runtime](http://apr.apache.org/)（APR）作为最底层，使用了APR中包含Socket、缓冲池等多种技术，性能也提高了。APR也是Apache HTTPD的最底层。可想而知，同属于[ASF](http://www.apache.org/)（Apache Software Foundation）中的成员，互补互用的情况还是很多的，虽然使用了不同的开发语言。

Tomcat 的线程池位于tomcat-util.jar文件中，包含了两种线程池方案。方案一：使用APR的Pool技术，使用了JNI；方案二：使用Java实现的ThreadPool。这里介绍的是第二种。如果想了解APR的Pool技术，可以查看APR的源代码。

ThreadPool默认创建了5个线程，保存在一个200维的线程数组中，创建时就启动了这些线程，当然在没有请求时，它们都处理“等待”状态（其实就是一个while循环，不停的等待notify）。如果有请求时，空闲线程会被唤醒执行用户的请求。

具体的请求过程是： 服务启动时，创建一个一维线程数组（maxThread＝200个），并创建空闲线程(minSpareThreads＝5个)随时等待用户请求。 当有用户请求时，调用 threadpool.runIt(ThreadPoolRunnable)方法，将一个需要执行的实例传给ThreadPool中。其中用户需要执行的实例必须实现ThreadPoolRunnable接口。ThreadPool 首先查找空闲的线程，如果有则用它运行要执行ThreadPoolRunnable；如果没有空闲线程并且没有超过maxThreads，就一次性创建 minSpareThreads个空闲线程；如果已经超过了maxThreads了，就等待空闲线程了。总之，要找到空闲的线程，以便用它执行实例。找到 后，将该线程从线程数组中移走。 接着唤醒已经找到的空闲线程，用它运行执行实例（ThreadPoolRunnable）。 运行完ThreadPoolRunnable后，就将该线程重新放到线程数组中，作为空闲线程供后续使用。

由此可以看出，Tomcat的线程池实现是比较简单的，ThreadPool.java也只有840行代码。用一个一维数组保存空闲的线程，每次以一个较小步伐（5个）创建空闲线程并放到线程池中。使用时从数组中移走空闲的线程，用完后，再“归还”给线程池。

ThreadPool提供的仅仅是线程池的实现，而如何使用线程池也是有很大学问的。让我们看看Tomcat是如何使用ThreadPool的吧。

Tomcat有两种EndPoint，分别是AprEndpoint和PoolTcpEndpoint。前者自己实现了一套线程池（其实这和Tomcat 老版本的方案是相同的，至今Tomcat中还保留着老版本的线程池，PoolTcpEndpoint也有类似的代码，通过“策略”可以选择不同的线程池方 案）。我们只关注PoolTcpEndpoint如何使用ThreadPool的。

首先，PoolTcpEndpoint创建了一个ThreadPoolRunnable实例——LeaderFollowerWorkerThread，实际上该实例就是接收（Accept）并处理（Process）用户socket请求。接着将该实例放进ThreadPool中并运行，此时就可以接收用户的请求了。

当有Socket请求时，LeaderFollowerWorkerThread首先获得了Socket实例，注意此时LeaderFollowerWorkerThread并没有急着处理该Socket，而是在响应Socket消息前，再次将LeaderFollowerWorkerThread放进ThreadPool中，从而它（当然是另外一个线程了）可以继续处理其他用户的Socket请求；接着，拥有Socket的LeaderFollowerWorkerThread再来处理该用户的Socket请求。

整个过程与传统的处理用户Socket请求是不同的，也和Tomcat老版本不同。传统的处理方法是：有一个后台运行的监听线程负责统一处理接收（注意只 是“接收”）Socket请求，当有新的Socket请求时，将它赋值给一个Worker线程（通常是唤醒该线程），并有后者处理Socket请求，监听 线程继续等待其他Socket请求。所以整个过程中有一个从Listener到Worker切换的过程。

而新版本Tomcat很有创造性的使用了另外一种方法，正如前文所描述的，接收和处理某个用户Socket请求的始终是由一个线程全程负责，没有切换到其 他线程处理，少了这种线程间的切换是否更有效率呢？我还不能确认。不过这种使用方式确实有别于传统模式，有种耳目一新的感觉。

## Jetty

除了Tomcat外，Jetty是另外一个重要的Java Web容器，号称“最小的”Web容器，从Jetty的源代码规模可以看出它确实比较小。而且它的ThreadPool的实现也非常简单，整个代码ThreadPool代码只有450行左右，可见小巧之极。

ThreadPool代码位于com.mortbty.thread包中，其中最重要的方法是dispatch（）和内部类PoolThread。顾名思 义，dispatch方法主要是将Runnable实例派给线程池中的空闲PoolThread，由后者运行Runnable。

还是看看整个过程吧。首先，ThreadPool创建_minThreads个空闲PoolThread，并把它们添加到空闲线程队列中。当需要运行 Runnable时，首先查找是否有空闲的PoolThread，如果有空闲的，这由它处理；如果没有并且PoolThread并没有超过 _maxThreads个时，则创建一个新的PoolThread，并由这个新创建的PoolThread运行Runnable；如果 PoolThread超过了_maxThreads，则一直等待有空闲的PoolThread出现。在PoolThread运行之前，必须把该 PoolThread从空闲线程队列中移走。

再来看看PoolThread的实现吧。和所有的Worker线程一样，用一个while（flag）{wait();}循环等待Runnable的到 来，当有Runnable被ThreadPool.dispatch（）时，该PoolThread就运行Runnable；当运行完成后，再“归还”给 空闲线程队列。

Jetty如何使用ThreadPool？整个Jetty只使用了一个ThreadPool实例，具体入口在 org.mortbay.jetty.Server中被实例化的，Connector中也使用Server的ThreadPool处理用户的Socket 请求。Connector是处理用户Socket请求的入口，一个Connector创建_acceptors个Acceptor，由Acceptor处 理用户Socket请求时，当有Socket请求时，就创建一个Connection放到线程池中处理，而Acceptor继续处理其他的Socket请 求。这是个传统的Listener和Worker处理方式。

## Resin

在这些Java Web容器中，Resin算得上很特别的，小巧稳定，而且效率很高。在这些Java Web容器中，算它的效率最高了。很多大型的网站中都能找到它的身影。Resin从3.0版本后开始走“特色”的开源路，与MySql很相似——如果用于 商业目的，则需要买它的License。但对于个人研究而言，这已经不错了，在网站上可以下载除了涉及License的源代码外其他所有代码。

说Resin特别，还主要是由于它的性能出众，即使在很多企业级应用中也能派上用场。Resin的数据库连接池做的很不错，效率非常高。不过这里我们讨论它的线程池，看看有何特别之处。

Resin的ThreadPool位于com.caucho.util.ThreadPool中，不过这个类的命名有点蹊跷，更恰当的命名是 ThreadPoolItem，因为它确实只是一个普通的Thread。那线程调度何管理在哪里呢？也在这个类中，不过都是以静态函数方式提供的，所以这 个类起到了两重作用：线程池调度和Worker线程。也由于这种原因，Resin实例中只有一个线程池，不像Tomcat和Jetty可以同时运行多个线 程池，不过对于一个系统而言，一个线程池足够了。

和其他线程池实现方式不同的是，Resin采用链表保存线程。如果有请求时，就将Head移走并唤醒该线程；待运行完成后，该线程就变成空闲状态并且被添 加到链表的Head部分。另外，每一个线程运行时都要判断当前空闲线程数是否超过_minSpareThreads，如果超过了，该线程就会退出（状态变 成Dead），也从链表中删除。

Resin如何使用该ThreadPool？所有需要用线程池的地方，只需调用ThreadPool. Schedule(Runnable)即可。该方法就是一个静态函数，顾名思义，就是将Runnable加到ThreadPool中待运行。

Resin使用的还是传统方法：监听线程（com.caucho.server.port.Port）,系统中可以有多个Port实例，前提端口号不同， 比如有80和8080端口；另外就是Worker线程，其实就是ThreadPool中的空闲线程。Port本身是一个Thread，在启动时，会在 ThreadPool中运行5个线程——TcpConnection同时等待用户请求，当有用户请求时，其中的一个会处理。其他继续等待。当处理用户请求 完成后，还可以重用这些TcpConnection，这与Jetty的有所不同，Jetty是当有用户请求时，才创建连接，处理完成后也不会重用这些连 接，效率会稍差一些。

另外Resin有两个后台运行线程：ThreadLauncher和ScheduleThread，前者负责当空闲线程小于最小空闲线程时创建新的线程； 而后者则负责运行实际的Runnable。我觉得有的负责，没有必要用一个线程来创建新线程，多此一举。不过ScheduleThread是必须的，因为 它就是Worker线程。

## 总结 {#summary}

介绍了tomcat、jetty和resin三种Java Web容器的线程池后，按照惯例应该比较它们的优缺点。不过先总结线程池的特点。

线程池作为提高程序处理数据能力的一种方案，应用非常广泛。大量的服务器都或多或少的使用到了线程池技术，不管是用Java还是C++实现，线程池都有如下的特点：

线程池一般有三个重要参数：

1. 最大线程数。在程序运行的任何时候，线程数总数都不会超过这个数。如果请求数量超过最大数时，则会等待其他线程结束后再处理。
2. 最大共享线程数，即最大空闲线程数。如果当前的空闲线程数超过该值，则多余的线程会被杀掉。
3. 最小共享线程数，即最小空闲线程数。如果当前的空闲数小于该值，则一次性创建这个数量的空闲线程，所以它本身也是一个创建线程的步长。

线程池有两个概念：

1. Worker线程。工作线程主要是运行执行代码，有两种状态：空闲状态和运行状态。在空闲状态时，类似“休眠”，等待任务；处理运行状态时，表示正在运行任务（Runnable）。
2. 辅助线程。主要负责监控线程池的状态：空闲线程是否超过最大空闲线程数或者小于最小空闲线程数等。如果不满足要求，就调整之。

如果按照上述标准去考察这三个容器就会发现：Tomcat实现的线程池是最完备的，Resin次之，而Jetty最为简单。Jetty没有控制空闲线程的数量，可能最后空闲线程数会达到最大线程数，影像性能，毕竟即使是休眠线程也会耗费CPU时钟的。

谈谈Resin的线程池。Resin的实现比Tomcat复杂些。也有上述三个参数，也有两个概念，这与Tomcat相当。但考虑到如何使用ThreadPool时，Resin也要复杂些。

或许由于Resin的ThreadPool是单间模式的，所有使用ThreadPool的线程都是相同设置，比如相同的最大线程数，最大空闲线程数等，在 使用它时会多些考虑。比如在控制最大Socket连接数时，com.caucho.server.port.Port还要有自己的一套控制“数量”的机 制，而无法使用ThreadPool所特有的控制机制。所以使用起来比Tomcat复杂。

Tomcat使用ThreadPool却很简单。由于Tomcat的ThreadPool可以有不同的实例存在，很方便的定制属于自己的“数量”控制，直接用ThreadPool控制Socket连接数量。所以代码也比较清爽。

如果要使用线程池，那就用Tomcat的ThreadPool吧。