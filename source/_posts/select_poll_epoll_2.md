---
title: select/poll/epoll —— select
date: 2024-05-06 11:36:17
categories:
- C/C++
tags:
- I/O多路复用
mathjax: true
---
I/O多路复用 —— select()函数
<!-- more -->

# select函数

## 函数原型

使用select这种IO多路转接方式需要调用一个同名函数select，这个函数是跨平台的，Linux、Mac、Windows都是支持的。调用此函数，可以委托内核帮助我们检测若干个文件描述符的状态，<font color="red">其实就是检测文件描述符读写缓冲区的状态</font>。

1个监听的描述符，N个通信的描述符。

* 读缓冲区：检测里边有没有数据，如果有数据该缓冲区对应的文件描述符就绪

* 写缓冲区：检测写缓冲区是否可以写(有没有容量)，如果有容量可以写，缓冲区对应的文件描述符就绪

* 读写异常：检测读写缓冲区是否有异常，如果有该缓冲区对应的文件描述符就绪

委托检测的文件描述符被遍历检测完毕之后，已就绪的这些满足条件的文件描述符会通过select()的参数分3个集合传出，程序猿得到这几个集合之后就可以分情况依次处理了。

```c
#include <sys/select.h>
struct timeval{
	time_t		tv_sec; /* seconds */
	suseconds_t tv_usec; /* microseconds */
};

int select(int nfds, fd_set *readfds, fd_set *writefds,
          fd_set *exceptfds, struct timeval * timeout);
```

* 函数参数

  * nfds：委托内核检测的三个集合中最大的文件描述符 + 1

    * 内核需要线性遍历这些集合中的文件描述符，这个值是循环结束的条件

    * 在windows中无效，置为-1即可
  * readfds：文件描述符的集合，内核只检测这个集合中文件描述符号对应的读缓冲区
    * <font color="red">传入传出参数</font>，读集合一般情况下都需要检测，这样才知道 
  * writefds：文件描述符的集合，内核只检测这个集合中文件描述符号对应的写缓冲区
    *  <font color="red">传入传出参数</font>，如果不需要这个参数可以指定为NULL
  * exceptfds：文件描述符集合，内核检测该集合中的文件描述符是否有异常
    * <font color="red">传入传出参数</font>，如果不需要这个参数可以指定为NULL
  * timeout：超时时长，用来强制解除select()函数的阻塞的
    * NULL：函数检测不到就绪的文件描述符会一直阻塞
    * 等待固定时长（秒）：函数检测不到就绪的文件描述符，在指定时长之后强制解除阻塞，函数返回0
    * 不等待：函数不会阻塞，直接将该参数对应的结构体初始化为0即可
* 函数返回值：
  * 大于0：成功，返回集合中满足条件的文件描述符的总个数
  * 等于-1：函数调用失败
  * 等于0：超时，没有检测到就绪的文件描述符

* 具体分两步：
  * select()  把数据给到内核，内核会把三个集合拷贝一份，基于线性表检测三个集合里对应的读、写以及异常事件，
  * 检测到会传出对应的集合（写入指针指向的内存地址，数据被内核修改了）。


另外初始化fd_set类型的参数还需要使用相关的一些列操作函数，具体如下：

```C
// 将文件描述符fd从set集合中删除 == 将fd对应的标志位设置为0        
void FD_CLR(int fd, fd_set *set);
// 判断文件描述符fd是否在set集合中 == 读一下fd对应的标志位到底是0还是1
int  FD_ISSET(int fd, fd_set *set);
// 将文件描述符fd添加到set集合中 == 将fd对应的标志位设置为1
void FD_SET(int fd, fd_set *set);
// 将set集合中, 所有文件文件描述符对应的标志位设置为0, 集合中没有添加任何文件描述符
void FD_ZERO(fd_set *set);
```

## 细节描述

在`select()`函数中第2、3、4个参数都是`fd_set`类型，它表示一个文件描述符的集合，类似于信号集 `sigset_t`，这个类型的数据有128个字节，也就是1024个标志位，和内核中文件描述符表中的文件描述符个数是一样的。

```c
sizeof(fd_set) = 128 字节 * 8 = 1024 bit      // int [32]
```

这并不是巧合，而是故意为之。这块内存中的每一个bit 和 文件描述符表中的每一个文件描述符是一一对应的关系，这样就可以使用最小的存储空间将要表达的意思描述出来了。

下图中的fd_set中存储了要委托内核检测读缓冲区的文件描述符集合。

* 如果集合中的标志位为0代表不检测这个文件描述符状态
* 如果集合中的标志位为1代表检测这个文件描述符状态

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/images/IO2_f1.png" width = "50%" alt=""/>
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
      <!-- figure 1: 套接字通信的文件描述符 -->
  	</div>
</center> 

内核在遍历这个读集合的过程中，如果被检测的文件描述符对应的读缓冲区中没有数据，内核将修改这个文件描述符在读集合`fd_set`中对应的标志位，改为`0`，如果有数据那么这个标志位的值不变，还是`1`。

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/images/IO2_f2.png" width = "50%" alt=""/>
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
      <!-- figure 1: 套接字通信的文件描述符 -->
  	</div>
</center> 

当select()函数解除阻塞之后，被内核修改过的读集合通过参数传出，此时集合中只要标志位的值为1，那么它对应的文件描述符肯定是就绪的，我们就可以基于这个文件描述符和客户端建立新连接或者通信了。

## 并发处理

如果在服务器基于select实现并发，其处理流程如下：

1、创建监听的套接字 lfd = socket()；

2、将监听的套接字和本地的IP和端口绑定 bind()；

3、给监听的套接字设置监听 listen()；

4、创建一个文件描述符集合 fd_set，用于存储需要检测读事件的所有的文件描述符

* 通过 FD_ZERO() 初始化

* 通过 FD_SET() 将监听的文件描述符放入检测的读集合中

5、循环调用select()，周期性的对所有的文件描述符进行检测

6、select() 解除阻塞返回，得到内核传出的满足条件的就绪的文件描述符集合

* 通过FD_ISSET() 判断集合中的标志位是否为 1
  * 如果这个文件描述符是监听的文件描述符，调用 accept() 和客户端建立连接
    * 将得到的新的通信的文件描述符，通过FD_SET() 放入到检测集合中
  * 如果这个文件描述符是通信的文件描述符，调用通信函数和客户端通信
    * 如果客户端和服务器断开了连接，使用FD_CLR()将这个文件描述符从检测集合中删除
    * 如果没有断开连接，正常通信即可

7、重复第6步

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/images/IO2_f3.png" width = "50%" alt=""/>
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
      <!-- figure 1: 套接字通信的文件描述符 -->
  	</div>
</center> 

客户端不需要使用IO多路转接进行处理，因为客户端和服务器的对应关系是 1：N，也就是说客户端是比较专一的，只能和一个连接成功的服务器通信。

虽然使用select这种IO多路转接技术可以降低系统开销，提高程序效率，但是它也有局限性：

1. 待检测集合（第2、3、4个参数）需要频繁的在用户区和内核区之间进行数据的拷贝，效率低
2. 内核对于select传递进来的待检测集合的检测方式是线性的
   * 如果集合内待检测的文件描述符很多，检测效率会比较低
   * 如果集合内待检测的文件描述符相对较少，检测效率会比较高
3. 使用select能够检测的最大文件描述符个数有上限，默认是1024，这是在内核中被写死了的。
















