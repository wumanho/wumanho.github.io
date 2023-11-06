---
weight: 2
title: "理解 Redis 简洁高效的系统设计"
summary: "对计算机基础进行一些复习和回顾"
date: 2021-01-21T08:49:20+08:00
draft: false
categories: ["技术","推荐阅读"]
tags: ["Redis","上下文切换","多路复用"]
---

![Redis logo](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/whyredisisfast/Redis_logo.png)

Redis 是一个基于内存的键值数据库，由于使用简单、优秀的设计以及提供了持久化和分片等数据可靠性机制，被广泛应用在各种项目上，相信做过后端开发的朋友就算没用过也一定听说过它的大名。

Redis 被使用最多的应该是缓存和集群这两种场景，由于它是基于内存的，而且有丰富的数据结构支持，所以在几年前第一次接触到这个被称为「内存数据库」的东西的时候，我第一感觉就是*这不就是一段可以跨设备共享的内存吗*😅。

确实，随着系统架构由「单机」演变成「集群」或者「分布式」，依靠多台机器协同工作的时候，一些需要共享的数据不再适合保存在内存中。

例如用户的会话信息，而仅仅依靠传统关系型数据库又无法保证访问速度，所以 Redis 便担起了这一重任。

不过经过一些时间的学习和使用后，发现在 Redis 为什么快这个问题上，如果仅仅解释为「因为它是基于内存的」，其实并不够，所以这篇博客就主要就来讨论一下这个问题。

不过，虽然这篇博客是以 Redis 为标题和出发点展开讨论，但涉及到 Redis 本身的篇幅却不多，主要会聚焦在操作系统以及 I/O 的角度，探讨 Redis 的架构设计为它带来的优势，算是一次对于计算机基础的复习和回顾。



## 单线程的 Redis :thinking:

刚刚接触 Redis 的时候通常会被告知 Redis 是单线程的，虽然这么理解也没错，只是并不严谨，因为它并不是严格的单线程结构。

说 Redis 单线程实际上是指**Redis 的网络 IO和键值对读写是由一个线程来完成的，这也是 Redis 对外提供键值存储服务的主要流程**。

而 Redis 的持久化、数据同步等功能其实还是由 fork 出来的子进程去完成。

在大多数时候我们可能总是会认为多线程就等于高并发，可以增加系统吞吐率，意味着高性能，那为什么 Redis 要如此反直觉地使用单线程模型呢，要更好地理解这一点，就需要先了解多线程的开销。

### 并发问题

通常情况下，多线程系统在有合理的资源分配的情况下，确实可以增加系统中处理请求操作的资源实体，进而提升系统能够同时处理的请求数，即吞吐率。

不过，一个多线程系统无可避免的需要面对**并发问题**，原因是系统中通常会存在被多线程同时访问的共享资源，比如一个共享的数据结构。

当有多个线程要修改这个共享资源时，为了保证共享资源的正确性，就需要有额外的机制进行保证，例如常用的锁机制，而这个额外的机制，就会带来额外的开销。

### 上下文切换

我们知道 Linux 是一个多任务系统，操作系统通过将 CPU 执行权在极短的时间内轮流分配给这些进程，去造成多任务同时运行的错觉。

而「上下文切换」，就是指 CPU 从一个进程或线程切换到另一个进程或线程这个动作。

操作系统中的「进程」是指一个程序运行的实例，在 Linux 系统中，「线程」就是能并行运行并且与他们的父进程（创建他们的进程）共享同一地址空间（一段内存区域）和其他资源的轻量级的进程，**但实际上这两者在 Linux 系统上并无区别，线程本质上就是一个进程（有时称作微进程），都属于操作系统调度的基本单位，只不过线程间的内存是共享的**。

要理解进程的「上下文」，需要先知道什么是 CPU 「寄存器」和「程序计数器」，因为「上下文」的主要作用便是被用于保存他们的内容。

寄存器是 CPU 内部的高速内存，通过对常用值（通常是运算的中间值）的快速访问来提高计算机程序运行的速度，而程序计数器则用于表明指令序列中 CPU 正在执行的位置，计数器的值为正在执行的指令的位置或者下一个将要被执行的指令的位置。

所以简单来说，可以将这些「上下文」理解为该当前进程在某一个时间点的状态，在发生上下文切换时，系统需要将这些状态保留，当该进程再次被调度时，用于恢复进程的状态。

当发生硬件中断、程序主动挂起、或进程所分配的 CPU 时间片耗尽时，就会发生上下文切换，过程大概如下：

1. 挂起一个进程（或线程），将这个进程在 CPU 中的上下文存储于内存中的某处
2. 在内存中检索下一个进程（或线程）的上下文并将其在 CPU 的寄存器中恢复
3. 跳转到程序计数器所指向的位置（即跳转到进程被中断时的代码行），以恢复该进程

从以上几个步骤可以发现，操作系统发生线程切换的时候，需要保留线程的上下文，然后执行系统调用。

此时如果线程数过高，可能执行线程切换的时间甚至会大于线程执行的时间，带来的表现往往是系统 load 偏高、CPU sys 使用率特别高（超过 20% 以上)，导致系统几乎陷入不可用的状态。

而且，**唯一可以保存上下文的地方**就只有 CPU 外部相对较慢的 RAM 内存，而 RAM 内存对于 CPU 来说是非常慢的，在 RAM 中对上下文的存取和检索操作约等于白白浪费了 1000 个 CPU 时钟周期。

所以可以说，上下文切换是计算密集型的，需要占用相当可观的处理器时间，对于一个多线程系统来说，将会是一笔不可忽略的开销。

所以综上所述，如果理解了 Redis 为什么设计为单线程，接下来的问题便是，Redis 如何通过单线程架构实现超高性能。

&nbsp;

## 多路复用 I/O 机制 :tada:

Redis 在只使用单线程响应客户端的前提下，仍然可以提供每秒数十万级别的处理能力，需要归功于另一个设计，I/O 多路复用机制。

与多进程和多线程技术相比，I/O 多路复用技术的最大优势是系统开销小，系统不必创建进程/线程，也不必维护这些进程/线程，从而大大减小了系统的开销。

### Blocking I/O 模型

先回忆一下传统的服务器端同步阻塞 I/O 处理的经典编程模型：

```java
/*以下是伪代码*/
public class MyServer {
    public static void main(String[] args) throws IOException {
        //创建一个 socket 绑定到本地端口
        ServerSocket serverSocket = new ServerSocket();
        serverSocket.bind("127.0.0.1","8080");
		
        //死循环等待客户端请求，并处理读写事件
        while (true){
            //这时候会阻塞，如果没有客户端连接，将会永远阻塞
            final Socket socket = serverSocket.accept();
            //当有客户端访问，则可以利用socket对象读取以及写入返回数据
            String data = socket.getInputStream().read();
            if(null != data){
                socket.write("...") //写数据
            }
        }
    }
}
```

请注意，在上面的代码中，socket.accept()、socket.read()、socket.write() 三个主要函数都是同步阻塞的，意味着当一个连接在处理 I/O 的时候，系统如果是单线程的，就必然会挂死在那里。

不过线程阻塞的时候，CPU 是被释放出来的，所以实际情况下，当有请求到达时，通常会利用一个新的线程去处理读写事件：

```java
/*以下是伪代码*/
public class MyServer {
    public static void main(String[] args) throws IOException {
        //创建一个 socket 绑定到本地端口
        ServerSocket serverSocket = new ServerSocket();
        serverSocket.bind("127.0.0.1","8080");
        while (true){
            final Socket socket = serverSocket.accept();
            //start 一个新的线程处理读写事件
            new MyThread(socket).start();
        }
    }
}

//执行读写逻辑
class MyThread extends Thread{
    private final Socket socket;
    public MyThread(Socket socket) {
        this.socket = socket;
    }
    @Override
    public void run() {
       //获取客户端输入
       socket.getInputStream().read();
    }
}
```

这就是经典的 Blocking I/O 模型，它所导致的问题也很明显，就是严重依赖线程。

而根据上文的分析，线程属于非常昂贵的资源，每个线程都需要一定的内存资源保存线程栈，当线程数量过多的时候，还需要考虑上下文切换的开销问题。

在面对少量客户端的时候，这种便于理解和维护的经典 BIO 模型其实是一个不错的选择，不过当面对大量并发连接的时候，为每一个客户端都创建一个线程不是一个好的主意。

### 多路复用的关键 select, poll 和 epoll

多路复用解决方案的核心是利用内核机制对一组文件描述符（file descriptors 简称FD）进行轮询操作，以达到以单个线程处理多个 I/O 事件的效果。

在 Linux（Unix）系统中有一个重要的基本概念，就是「一切皆文件」，每个 Linux 进程都有一个文件描述符表（File descriptor table）指向文件、设备、socket 连接等等操作系统对象，也就是说，每一个客户端 socket 连接，在 Linux 看来就是一个 FD。	

`select()`、`poll()`和`epoll()`都是由 Linux 内核提供的系统调用，这几个系统调用函数提供了一种相同的思路：**创建并监听一组文件描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），通知程序进行相应的读写操作**。

目前主流的高级语言都有对于上述系统调用的封装，例如 `Java` 的 `NIO`包，`golang` 和 `Node.js` 的网络库等等，本文的主角 Redis 也不例外，它的多路复用模块同样封装了`select()`，`epoll()`，`kqueue()`这些系统调用，为上层提供了相同的接口。

### select

```c
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

`select()`会返回就绪描述符的数目，超时返回0，出错返回-1，对 `select()` 的调用将会阻塞，直到给定的文件描述符可以执行 I/O ，或者直到指定的超时时间结束。

`select()`监听的文件描述符会被分成三类：

* readfds 集合中的文件描述符会被监听以检查数据是否可以读取；
* writefds 集合中的文件描述符会被监听以检查写操作是否可以完成；
* exceptfds 集合中的文件描述符会被监听以检查是否发生了异常；

如果某个集合为 Null ，那么 select 则不会监听它的事件。

不过`select()`也存在一些缺陷，先看看以下示例代码：

```C
//子进程
void child_process(void)
{
  sleep(2);
  char msg[MAXBUF];
  struct sockaddr_in addr = {0};
  int n, sockfd,num=1;
  srandom(getpid());
  /* 创建 socket 并连接服务器 */
  sockfd = socket(AF_INET, SOCK_STREAM, 0);
  addr.sin_family = AF_INET;
  addr.sin_port = htons(2000);
  addr.sin_addr.s_addr = inet_addr("127.0.0.1");
 
  connect(sockfd, (struct sockaddr*)&addr, sizeof(addr));
 
  printf("child {%d} connected \n", getpid());
  while(1){
        int sl = (random() % 10 ) +  1;
        num++;
     	sleep(sl);
  	sprintf (msg, "Test message %d from client %d", num, getpid());
  	n = write(sockfd, msg, strlen(msg));	/* 发送消息 */
  }
 
}
//服务器进程
int main()
{
  //初始化设置，启动子进程
  char buffer[MAXBUF];
  int fds[5];
  struct sockaddr_in addr;
  struct sockaddr_in client;
  int addrlen, n,i,max=0;;
  int sockfd, commfd;
  fd_set rset;
  for(i=0;i<5;i++)
  {
  	if(fork() == 0)
  	{
  		child_process();
  		exit(0);
  	}
  }
 //socket建立，创建文件描述符
  sockfd = socket(AF_INET, SOCK_STREAM, 0);
  memset(&addr, 0, sizeof (addr));
  addr.sin_family = AF_INET;
  addr.sin_port = htons(2000);
  addr.sin_addr.s_addr = INADDR_ANY;
  bind(sockfd,(struct sockaddr*)&addr ,sizeof(addr));
  listen (sockfd, 5); 
 
  for (i=0;i<5;i++) 
  {
    memset(&client, 0, sizeof (client));
    addrlen = sizeof(client);
    fds[i] = accept(sockfd,(struct sockaddr*)&client, &addrlen);
    if(fds[i] > max)
    	max = fds[i];
  }
  //创建死循环，调用select
  while(1){
	FD_ZERO(&rset);
  	for (i = 0; i< 5; i++ ) {
  		FD_SET(fds[i],&rset);
  	}
 
   	puts("round again");
	select(max+1, &rset, NULL, NULL, NULL);
 
	for(i=0;i<5;i++) {
		if (FD_ISSET(fds[i], &rset)){
			memset(buffer,0,MAXBUF);
			read(fds[i], buffer, MAXBUF);
			puts(buffer);
		}
	}	
  }
  return 0;
}
```

这里启动了五个子进程模拟客户端连接到服务器发送消息，服务器进程在一个死循环中将创建一个包含所有文件描述符的集合，调用`select()`并遍历检查哪个文件描述符有可读事件发生。

`select()`返回的时候会将集合替换为只包含那些已经准备好的文件描述符，所以每一次循环都要重新构建文件描述符集合。

所以我们来总结一下`select()`的优缺点：

* 优点：select 的主要优点是可移植性非常强，所有 unix like 操作系统都支持；
* 缺点：每次调用前都需要重新构建集合；
* 缺点：对socket进行扫描时是线性扫描，即采用轮询的方法，效率较低，时间复杂度O(n)；
* 缺点：单个进程能够监视的文件描述符的数量存在最大限制，一般来说这个数目和系统内存关系很大，具体数目可以`cat /proc/sys/fs/file-max`察看；
* 缺点：`select()`基于位图，这也是它数量存在最大限制的原因，而且即使只监听 5 个文件描述符，也要遍历所有位图。

### poll

```C
int poll (struct pollfd *fds, unsigned int nfds, int timeout);
```

`poll()`最大的改进是取消了像`select()`那样的最大链接数限制，并且将`select()`的三类文件描述符集合改成了使用单一 nfds pollfd结构数组，原型更简单。

`poll()`本质上和`select()`没有区别，它将用户传入的数组拷贝到内核空间，然后查询每个 fd 对应的设备状态，如果设备就绪则在设备等待队列中加入一项并继续遍历，如果遍历完所有 fd 后没有发现就绪设备，则挂起当前进程，直到设备就绪或者主动超时，被唤醒后它又要再次遍历 fd。这个过程经历了多次无谓的遍历。

总结一下`poll()`的优缺点：

* 优点：没有最大链接数限制；
* 改进：不需要像`select()`一样遍历位图，但不幸的是仍然要遍历整个文件描述符集合；
* 改进：**如果给予的文件描述符集合没有被更改**，则不需要再像`select()`一样每次都重新构建一个新的集合返回，允许重用；
* 缺点：不是所有 Unix 系统都支持；

### epoll

`epoll`在Linux 2.6 内核中被提出，是对`select()`和`poll()`的改进版本。

回想一下刚才的`select()`和`pool()`，我们所有操作都发生在用户空间，然后每一次调用都需要将集合在用户空间和内核空间中拷贝一次，而且每当有一个新的 socket 连接时，我们需要将它添加到集合中，然后再重新调用`select()`/`poll()`。

而`epoll`系统调用帮我们在内核中创建和管理一个简易的文件系统。

在调用时只需要以下三个操作：

1. 使用`epoll_create(int size)`创建一个 `epoll` 上下文，size 参数用于告诉内核这个监听的数目一共有多大；
2. 使用`epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)`向`epoll`添加、删除或修改文件描述符，告诉内核需要监听什么事件；
3. 使用` epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout)`在给定的timeout时间内，当在监控的所有句柄中有事件发生时，就返回用户态的进程；

将上面的示例改成使用`epoll`：

```c
 struct epoll_event events[5];
  int epfd = epoll_create(10);
  ...
  ...
  for (i=0;i<5;i++) 
  {
    static struct epoll_event ev;
    memset(&client, 0, sizeof (client));
    addrlen = sizeof(client);
    ev.data.fd = accept(sockfd,(struct sockaddr*)&client, &addrlen);
    ev.events = EPOLLIN;
    epoll_ctl(epfd, EPOLL_CTL_ADD, ev.data.fd, &ev); 
  }
  
  while(1){
  	puts("round again");
  	nfds = epoll_wait(epfd, events, 5, 10000);
	
	for(i=0;i<nfds;i++) {
			memset(buffer,0,MAXBUF);
			read(events[i].data.fd, buffer, MAXBUF);
			puts(buffer);
	}
  }
```

首先需要创建一个`epoll`上下文，当客户端连接时，创建一个`epoll`事件对象并注册到`epoll`上下文中，在死循环里面我们只需要等待事件通知。

总结一下`epoll`的优缺点：

* 优点：文件描述符只在调用`epoll_ctl()`时拷贝，每次调用`epoll_wait()`不拷贝；
* 优点：没有最大连接数限制；
* 优点：事件驱动，不需要遍历集合，复杂度O(1)；
* 优点：利用`mmap()`文件映射内存加速与内核空间的消息传递，减少拷贝开销。
* 缺点：只有 Linux 支持；

&nbsp;

## 总结

这篇博客主要在操作系统以及 I/O 这两个方面分析了 Redis 在简洁的架构下却能表现出优异性能的根本原因。

Redis 通过单线程规避了并发问题以及多线程上下文切换带来的开销，然后利用设计良好的 I/O 多路复用模块保证了大量并发下的稳定性能，并且由于数据读写都发生在内存中，可以保证超低的读写延迟，利用子进程处理数据的持久化和同步问题，这样简洁又优雅的设计确实值得我们学习。



### 参考

[极客时间《Redis核心技术与实战》](https://time.geekbang.org/column/intro/100056701)

[Redis 和 I/O 多路复用](https://draveness.me/redis-io-multiplexing/)

[彻底理解 I/O 多路复用](https://juejin.cn/post/6844904200141438984#heading-14)

[Remzi H. Arpaci-Dusseau and Andrea C. Arpaci-Dusseau (University of Wisconsin-Madison). 2018.Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/)

上面是官方免费英文版，推荐一下中文版：[《操作系统导论》](https://book.douban.com/subject/33463930/)

