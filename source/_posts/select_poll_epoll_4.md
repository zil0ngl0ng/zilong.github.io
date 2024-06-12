---
title: select/poll/epoll —— epoll
date: 2024-05-07 14:36:17
categories:
- C/C++
tags:
- I/O多路复用
mathjax: true
---
I/O多路复用 —— epoll()函数
<!-- more -->
# epoll

## 概述

epoll 全称 eventpoll，是 linux 内核实现IO多路转接/复用（IO multiplexing）的一个实现。IO多路转接的意思是在一个操作里同时监听多个输入输出源，在其中一个或多个输入输出源可用的时候返回，然后对其的进行读写操作。epoll是select和poll的升级版，相较于这两个前辈，epoll改进了工作方式，因此它更加高效。

* 对于待检测集合select和poll是基于线性方式处理的，epoll是基于红黑树来管理待检测集合的。
* select和poll每次都会线性扫描整个待检测集合，集合越大速度越慢，epoll使用的是回调机制，效率高，处理效率也不会随着检测集合的变大而下降
* select和poll工作过程中存在内核/用户空间数据的频繁拷贝问题，在epoll中内核和用户区使用的是共享内存（基于mmap内存映射区实现），省去了不必要的内存拷贝。
* 程序猿需要对select和poll返回的集合进行判断才能知道哪些文件描述符是就绪的，通过epoll可以直接得到已就绪的文件描述符集合，无需再次检测
* 使用epoll没有最大文件描述符的限制，仅受系统中进程能打开的最大文件数目限制

当多路复用的文件数量庞大、IO流量频繁的时候，一般不太适合使用select()和poll()，这种情况下select()和poll()表现较差，推荐使用epoll()。

## 操作函数

在epoll中一共提供是三个API函数，分别处理不同的操作，函数原型如下：

```C
#include <sys/epoll.h>
// 创建epoll实例，通过一棵红黑树管理待检测集合
int epoll_create(int size);
// 管理红黑树上的文件描述符(添加、修改、删除)
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
// 检测epoll树中是否有就绪的文件描述符
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

select/poll低效的原因之一是将“添加/维护待检测任务”和“阻塞进程/线程”两个步骤合二为一。每次调用select都需要这两步操作，然而大多数应用场景中，需要监视的socket个数相对固定，并不需要每次都修改。epoll将这两个操作分开，先用`epoll_ctl()`维护等待队列，再调用`epoll_wait()`阻塞进程（解耦）。通过下图的对比显而易见，epoll的效率得到了提升。

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/images/IO4_f1.png" width = "50%" alt=""/>
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
      <!-- figure 1: 套接字通信的文件描述符 -->
  	</div>
</center> 

`epoll_create()`函数的作用是创建一个红黑树模型的实例，用于管理待检测的文件描述符的集合。

```C
int epoll_create(int size);
```

* 函数参数 size：在Linux内核2.6.8版本以后，这个参数是被忽略的，只需要指定一个大于0的数值就可以了。
* 函数返回值：
  * 失败：返回-1
  * 成功：返回一个有效的文件描述符，通过这个文件描述符就可以访问创建的epoll实例了

`epoll_ctl()`函数的作用是管理红黑树实例上的节点，可以进行添加、删除、修改操作。

```C
// 联合体, 多个变量共用同一块内存        
typedef union epoll_data {
 	void        *ptr;
	int          fd;	// 通常情况下使用这个成员, 和epoll_ctl的第三个参数相同即可
	uint32_t     u32;
	uint64_t     u64;
} epoll_data_t;

struct epoll_event {
	uint32_t     events;      /* Epoll events */
	epoll_data_t data;        /* User data variable */
};
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

* 函数参数：
  * epfd：epoll_create() 函数的返回值，通过这个参数找到epoll实例
  * op：这是一个枚举值，控制通过该函数执行什么操作
    * EPOLL_CTL_ADD：往epoll模型中添加新的节点
    * EPOLL_CTL_MOD：修改epoll模型中已经存在的节点
    * EPOLL_CTL_DEL：删除epoll模型中的指定的节点
  * fd：文件描述符，即要添加/修改/删除的文件描述符
  * event：epoll事件，用来修饰第三个参数对应的文件描述符的，指定检测这个文件描述符的什么事件
    * events：委托epoll检测的事件
      * EPOLLIN：读事件, 接收数据, 检测读缓冲区，如果有数据该文件描述符就绪
      * EPOLLOUT：写事件, 发送数据, 检测写缓冲区，如果可写该文件描述符就绪
      * EPOLLERR：异常事件
    * data：用户数据变量，这是一个联合体类型，通常情况下使用里边的fd成员，用于存储待检测的文件描述符的值，在调用epoll_wait()函数的时候这个值会被传出。
  * 函数返回值：
    * 失败：返回-1
    * 成功：返回0

`epoll_wait()`函数的作用是检测创建的epoll实例中有没有就绪的文件描述符。

```c
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

* 函数参数：
  * epfd：epoll_create() 函数的返回值, 通过这个参数找到epoll实例
  * events：传出参数, 这是一个结构体数组的地址, 里边存储了已就绪的文件描述符的信息
  * maxevents：修饰第二个参数, 结构体数组的容量（元素个数）
  * timeout：如果检测的epoll实例中没有已就绪的文件描述符，该函数阻塞的时长, 单位ms 毫秒
    * 0：函数不阻塞，不管epoll实例中有没有就绪的文件描述符，函数被调用后都直接返回
    * 大于0：如果epoll实例中没有已就绪的文件描述符，函数阻塞对应的毫秒数再返回
    * -1：函数一直阻塞，直到epoll实例中有已就绪的文件描述符之后才解除阻塞

* 函数返回值：
  * 成功：
    * 等于0：函数是阻塞被强制解除了, 没有检测到满足条件的文件描述符
    * 大于0：检测到的已就绪的文件描述符的总个数
  * 失败：返回-1

## epoll的使用

在服务器端使用epoll进行IO多路转接的操作步骤如下：

1. 创建监听的套接字

```C
int lfd = socket(AF_INET, SOCK_STREAM, 0);
```

2. 设置端口复用（可选）

```C
int opt = 1;
setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
```

3. 使用本地的IP与端口和监听的套接字进行绑定

```C
int ret = bind(lfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
```

4. 给监听的套接字设置监听

```C
listen(lfd, 128);
```

5. 创建epoll实例对象

```C
int epfd = epoll_create(100);
```

6. 将用于监听的套接字添加到epoll实例中

```C
struct epoll_event ev;
ev.events = EPOLLIN;    // 检测lfd读读缓冲区是否有数据
ev.data.fd = lfd;
int ret = epoll_ctl(epfd, EPOLL_CTL_ADD, lfd, &ev);
```

7. 检测添加到epoll实例中的文件描述符是否已就绪，并将这些已就绪的文件描述符进行处理

```C
int num = epoll_wait(epfd, evs, size, -1);
```

* 如果是监听的文件描述符，和新客户端建立连接，将得到的文件描述符添加到epoll实例中

```C
int cfd = accept(curfd, NULL, NULL);
ev.events = EPOLLIN;
ev.data.fd = cfd;
// 新得到的文件描述符添加到epoll模型中, 下一轮循环的时候就可以被检测了
epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &ev);
```

* 如果是通信的文件描述符，和对应的客户端通信，如果连接已断开，将该文件描述符从epoll实例中删除

```C
int len = recv(curfd, buf, sizeof(buf), 0);
if(len == 0)
{
    // 将这个文件描述符从epoll模型中删除
    epoll_ctl(epfd, EPOLL_CTL_DEL, curfd, NULL);
    close(curfd);
}
else if(len > 0)
{
    send(curfd, buf, len, 0);
}
```

8. 重复第7步的操作

## epoll的工作模式

### 水平模式

水平模式可以简称为LT模式，LT（level triggered）是缺省的工作方式，并且同时支持block和no-block socket。在这种做法中，内核通知使用者哪些文件描述符已经就绪，之后就可以对这些已就绪的文件描述符进行IO操作了。如果我们不作任何操作，内核还是会继续通知使用者。

**水平模式的特点：**

* 读事件：如果文件描述符对应的读缓冲区还有数据，读事件就会被触发，epoll_wait()解除阻塞
  * 当读事件被触发，epoll_wait()解除阻塞，之后就可以接收数据了
  * 如果接收数据的buf很小，不能全部将缓冲区数据读出，那么读事件会继续被触发，直到数据被全部读出，如果接收数据的内存相对较大，读数据的效率也会相对较高（减少了读数据的次数）
  * 因为读数据是被动的，必须要通过读事件才能知道有数据到达了，因此对于读事件的检测是必须的
* 写事件：如果文件描述符对应的写缓冲区可写，写事件就会被触发，epoll_wait()解除阻塞
  * 当写事件被触发，epoll_wait()解除阻塞，之后就可以将数据写入到写缓冲区了
  * 写事件的触发发生在写数据之前而不是之后，被写入到写缓冲区中的数据是由内核自动发送出去的
  * 如果写缓冲区没有被写满，写事件会一直被触发
  * 因为写数据是主动的，并且写缓冲区一般情况下都是可写的（缓冲区不满），因此对于写事件的检测不是必须的

### 边沿模式

边沿模式可以简称为ET模式，ET（edge-triggered）是高速工作方式，只支持no-block socket。在这种模式下，当文件描述符从未就绪变为就绪时，内核会通过epoll通知使用者。然后它会假设使用者知道文件描述符已经就绪，并且不会再为那个文件描述符发送更多的就绪通知（only once）。如果我们对这个文件描述符做IO操作，从而导致它再次变成未就绪，当这个未就绪的文件描述符再次变成就绪状态，内核会再次进行通知，并且还是只通知一次。ET模式在很大程度上减少了epoll事件被重复触发的次数，因此效率要比LT模式高。

**边沿模式的特点:**

* 读事件：当读缓冲区有新的数据进入，读事件被触发一次，没有新数据不会触发该事件
  * 如果有新数据进入到读缓冲区，读事件被触发，epoll_wait()解除阻塞
  * 读事件被触发，可以通过调用read()/recv()函数将缓冲区数据读出
  * 如果数据没有被全部读走，并且没有新数据进入，读事件不会再次触发，只通知一次
  * 如果数据被全部读走或者只读走一部分，此时有新数据进入，读事件被触发，并且只通知一次

* 写事件：当写缓冲区状态可写，写事件只会触发一次
  * 如果写缓冲区被检测到可写，写事件被触发，epoll_wait()解除阻塞
  * 写事件被触发，就可以通过调用write()/send()函数，将数据写入到写缓冲区中
  * 写缓冲区从不满到被写满，期间写事件只会被触发一次
  * 写缓冲区从满到不满，状态变为可写，写事件只会被触发一次

综上所述：epoll的边沿模式下 epoll_wait()检测到文件描述符有新事件才会通知，如果不是新的事件就不通知，通知的次数比水平模式少，效率比水平模式要高。

#### ET模式的设置

边沿模式不是默认的epoll模式，需要额外进行设置。epoll设置边沿模式是非常简单的，epoll管理的红黑树示例中每个节点都是struct epoll_event类型，只需要将EPOLLET添加到结构体的events成员中即可：

```C
struct epoll_event ev;
ev.events = EPOLLIN | EPOLLET;	// 设置边沿模式
```

#### 设置非阻塞

对于写事件的触发一般情况下是不需要进行检测的，因为写缓冲区大部分情况下都是有足够的空间可以进行数据的写入。对于读事件的触发就必须要检测了，因为服务器也不知道客户端什么时候发送数据，如果使用epoll的边沿模式进行读事件的检测，有新数据达到只会通知一次，那么必须要保证得到通知后将数据全部从读缓冲区中读出。那么，应该如何读这些数据呢？

* 方式1：准备一块特别大的内存，用于存储从读缓冲区中读出的数据，但是这种方式有很大的弊端：
  * 内存的大小没有办法界定，太大浪费内存，太小又不够用
  * 系统能够分配的最大堆内存也是有上限的，栈内存就更不必多言了

* 方式2：循环接收数据

```C
int len = 0;
while((len = recv(curfd, buf, sizeof(buf), 0)) > 0)
{
    // 数据处理...
}
```

这样做也是有弊端的，因为套接字操作默认是阻塞的，当读缓冲区数据被读完之后，读操作就阻塞了也就是调用的read()/recv()函数被阻塞了，当前进程/线程被阻塞之后就无法处理其他操作了。

要解决阻塞问题，就需要将套接字默认的阻塞行为修改为非阻塞，需要使用fcntl()函数进行处理：

通过上述分析就可以得出一个结论：**epoll在边沿模式下，必须要将套接字设置为非阻塞模式**，但是，这样就会引发另外的一个bug，在非阻塞模式下，循环地将读缓冲区数据读到本地内存中，当缓冲区数据被读完了，调用的read()/recv()函数还会继续从缓冲区中读数据，此时函数调用就失败了，返回-1，对应的全局变量 errno 值为 EAGAIN 或者 EWOULDBLOCK如果打印错误信息会得到如下的信息：Resource temporarily unavailable

```c
// 非阻塞模式下recv() / read()函数返回值 len == -1
int len = recv(curfd, buf, sizeof(buf), 0);
if(len == -1)
{
    if(errno == EAGAIN)
    {
        printf("数据读完了...\n");
    }
    else
    {
        perror("recv");
        exit(0);
    }
}
```

