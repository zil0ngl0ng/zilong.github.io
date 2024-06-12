---
title: select/poll/epoll —— poll
date: 2024-05-06 13:36:17
categories:
- C/C++
tags:
- I/O多路复用
mathjax: true
---
I/O多路复用 —— poll()函数
<!-- more -->

# poll函数

poll的机制与select类似，与select在本质上没有多大差别，使用方法也类似，下面的是对于二者的对比：

* 内核对应文件描述符的检测也是以线性的方式进行轮询，根据描述符的状态进行处理
* poll和select检测的文件描述符集合会在检测过程中频繁的进行用户区和内核区的拷贝，它的开销随着文件描述符数量的增加而线性增大，从而效率也会越来越低。
* select检测的文件描述符个数上限是1024，poll没有最大文件描述符数量的限制
* select可以跨平台使用，poll只能在Linux平台使用

poll函数的函数原型如下：

```C
#include <poll.h>
// 每个委托poll检测的fd都对应这样一个结构体
struct pollfd {
    int   fd;         /* 委托内核检测的文件描述符 */
    short events;     /* 委托内核检测文件描述符的什么事件 */
    short revents;    /* 文件描述符实际发生的事件 -> 传出 */
};

struct pollfd myfd[100];
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

<table align="center"><tr>    <td>事件</td>    <td>常量</td>    <td>作为events的值</td>    <td>作为revents的值</td>    <td>说明</td></tr><tr>    <td rowspan="4">读事件</td>    <td>POLLIN</td>    <td>√</td>    <td>√</td>    <td>普通或优先带数据可读</td></tr><tr>    <td>POLLRDNORM</td>    <td>√</td>    <td>√</td>    <td>普通数据可读</td></tr><tr>    <td>POLLRDBAND</td>    <td>√</td>    <td>√</td>    <td>优先级带数据可读</td></tr><tr>    <td>POLLPRI</td>    <td>√</td>    <td>√</td>    <td>高优先级数据可读</td></tr><tr>    <td rowspan="3">写事件</td>    <td>POLLOUT</td>    <td>√</td>    <td>√</td>    <td>普通或优先带数据可读</td></tr><tr>    <td>POLLWRNORM</td>    <td>√</td>    <td>√</td>    <td>普通数据可读</td></tr><tr>    <td>POLLWRBAND</td>    <td>√</td>    <td>√</td>    <td>优先级带数据可读</td></tr><tr>    <td rowspan="3">错误事件</td>    <td>POLLERR</td>    <td> </td>    <td>√</td>    <td>发生错误</td></tr><tr>    <td>POLLHUP</td>    <td> </td>    <td>√</td>    <td>发生挂起</td></tr><tr>    <td>POLLNVAL</td>    <td> </td>    <td>√</td>    <td>描述不是打开的文件</td></tr>

*  函数参数

   * fds：这是一个<font color="red">struct pollfd</font>类型的数组, 里边存储了待检测的文件描述符的信息，这个数组中有三个成员：
     * fd：委托内核检测的文件描述符
     * events：委托内核检测的fd事件（输入、输出、错误），每一个事件有多个取值
     * revents：这是一个传出参数，数据由内核写入，存储内核检测之后的结果

   * nfds：这是第一个参数数组中最后一个有效元素的下标 + 1（也可以指定参数1数组的元素总个数）
   * timeout：指定poll函数的阻塞时长
     * -1：一直阻塞，直到检测的集合中有就绪的文件描述符（有事件产生）解除阻塞
     * 0：不阻塞，不管检测集合中有没有已就绪的文件描述符，函数马上返回
     * 大于0：阻塞指定的毫秒（ms）数之后，解除阻塞

*  函数返回值

   * 失败：返回-1
   * 成功：返回一个大于0的整数，表示检测的集合中已就绪的文件描述符的总个数






