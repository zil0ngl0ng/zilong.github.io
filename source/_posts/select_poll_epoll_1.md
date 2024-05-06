---
title: select/poll/epoll —— 套接字通信
date: 2024-05-05 20:36:17
categories:
- C/C++
tags:
- I/O多路复用
mathjax: true
---
I/O多路复用前传 —— 套接字通信
<!-- more -->

服务器端有监听文件描述符和通信文件描述符，每个文件描述符都有读缓冲区和写缓冲区。

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/images/IO_1.png" width = "60%" alt=""/>
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
      figure 1: 套接字通信的文件描述符
  	</div>
</center> 

* 监听文件描述符

读缓冲区存储客户端的连接请求，所有客户端的连接请求都会记录到读缓冲区。

调用accept( )，会检测读缓冲区是否有请求数据，有的话就解除阻塞并建立连接；没有的话，就会一直阻塞。

* 通信文件描述符

读缓冲区是存储客户端发送来的通信数据，调用read() 方法可以读出，没有数据就堵塞。

写缓冲区存储用来发送的数据，写满了write() 阻塞。



accept()、read()、write()工作在同一个线程，互相排斥，其中一个阻塞了其他的函数就没法工作了。

**==>**  I/O多路复用技术，由内核负责检测多个文件描述符，不需要程序员来维护

**内核**来通知文件描述符的缓冲区是否可用，然后再调用accept()、read()、write()方法，就不会阻塞了

**优势：**系统开销小，系统不必创建进程/线程，也不用维护进程/线程