# I/O 演变

## Java ServerSocket()

追踪一个线程，会生成out文件，文件名是out，扩展名是正在执行的线程pid，例 out.1800。
```shell script
strace -ff -o out /execute/runable/file
```

这个文件中会记录下程序运行时执行的系统调用。

java 程序执行new ServerSocket() 时会执行如下系统调用（按先后顺序）。

1. socket() = 3
socket函数生成一个套接字描述符
2. bind(3, ...port) = 0
将套接字描述符关联起来
3. listen(3)
监听这个描述符
4. accept(3, 
等待来自客户端的连接，这里会阻塞线程（没有返回），直到有客户端请求连接。 

一旦客户端连接了server

accept(3,)

## 系统调用 accept() 函数

```c
#include <sys/types.h>          /* See NOTES */#include <sys/socket.h>int accept(int
sockfd, struct sockaddr *addr, socklen_t *addrlen);#define _GNU_SOURCE
            /* See feature_test_macros(7) */#include <sys/socket.h>int accept4(int
sockfd, struct sockaddr *addr,            socklen_t *addrlen, int flags);
```

### 描述

_**accept()**_ 系统调用被用来连接socket。它会从正在监听中的socket 队列中获取第一个connection，创建一个已连接的socket，并且返回这个socket
新的文件描述符（file descriptor），新的socket不再是监听状态。

>The _**accept()**_ system call is used with connection-based socket types (**SOCK_STREAM**, **SOCK_SEQPACKET**). It extracts the first
>connection request on the queue of pending connections for the listening socket, _sockfd_, creates a new connected socket,
> and returns a new file descriptor referring to that socket. The newly created socket is not in the listening state. 
>  The original socket _sockfd_ is unaffected by this call.  

上面提到的 _sockfd_ 是由 _**socket(2)**_ 创建，使用 _**bind(2)**_ 绑定一个本地地址，并且通过 _**listen(2)**_ 监听连接。

>The argument _sockfd_ is a socket that has been created with _**socket(2)**_, bound to a local address with _**bind(2)**_, 
>and is listening for connections after a _**listen(2)**_.  

### 返回值

成功返回一个非负整数，即接收到的socket 描述符。
失败返回-1。

>On success, these system calls return a nonnegative integer that is a descriptor for the accepted socket. On error, -1 is returned, and errno is set appropriately.  


## BIO

**BIO** 是同步阻塞IO，服务器会阻塞直到有客户端连接 ServerSocket#accept()。

### BIO 缺点

1. 一直等待客户端连接，会**阻塞**当前线程，直到有客户端连接才会向下继续执行。
2. 如果每次开辟新的线程去处理客户端的数据又会导致线程使用太多。

## NIO 

非阻塞IO

通过系统调用 _**fcntl()**_ 设置非阻塞，设置之后，后面的accept()函数就不会再阻塞了，如果没有客户端连接会直接返回-1

例：文件描述符是 5， accept的时候没有返回客户端的文件描述符，返回了-1

>fcntl(5, ...,O_NON_BLOCK) = 0  
>accept(5, ...) = -1  

### 优点

1. 非阻塞，可以不阻塞主流程，现在最少只需要两个线程即可

### 缺点

1. 如果客户端连接过多，最终的客户端集合会很多，每次都要逐个遍历才行，这样会浪费性能，过多无用的系统调用（_**recv()**_）
2. 上面这个轮询还会导致线程多了之后性能下降

