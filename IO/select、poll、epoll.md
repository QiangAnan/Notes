[https://blog.csdn.net/luojian5900339/article/details/54581852](https://blog.csdn.net/luojian5900339/article/details/54581852)

reactor:

[https://www.cnblogs.com/dawen/archive/2011/05/18/2050358.html](https://www.cnblogs.com/dawen/archive/2011/05/18/2050358.html)

[https://www.cnblogs.com/winner-0715/p/8733787.html](https://www.cnblogs.com/winner-0715/p/8733787.html)

## select
监视并等待多个文件描述符的属性变化（可读、可写或错误异常），调用后 select() 函数会阻塞，直到有描述符就绪（有数据可读、可写、或者有错误异常），或者超时（ timeout 指定等待时间），函数才返回。
```cpp
#include <sys/select.h>
#include <sys/time.h>

//返回值：就绪描述符的数目，超时返回0，出错返回-1
//maxfdp1: 表示待测试的描述符个数，取值为被测试描述符最大值+1
//readset/writeset/exceptset: 指定我们要让内核测试读、写和异常条件的描述字集合
//timeout：超时等待时间; 
//	若timeout为NULL，一直等待; timeout中字段为0，立即返回
int select(int maxfdp1, fd_set *readset, fd_set *writeset, fd_set *exceptset, const struct timeval *timeout);

// timeval结构体
struct timeval{
	long tv_sec;   //seconds
	long tv_usec;  //microseconds
};

// fd_set结构体操作函数
void FD_ZERO(fd_set *fdset);           // 清空集合
void FD_SET(int fd, fd_set *fdset);    // 将一个给定的文件描述符加入集合之中
void FD_CLR(int fd, fd_set *fdset);    // 将一个给定的文件描述符从集合中删除
int FD_ISSET(int fd, fd_set *fdset);   // 检查集合中指定的文件描述符是否可以读写 
```

- 优缺点
  - 优点: 同时监听进程中的多个IO状态
  - 缺点
    - 单个进程可监视的fd数量被限制，即能监听端口的大小有限
	- 对socket进行扫描时是线性扫描，即采用轮询的方法，效率较低
	- 需要维护一个用来存放大量fd的数据结构，这样会使得用户空间和内核空间在传递该结构时复制开销大

- 参考代码: [select_service.cpp](./select_service.cpp)  [select_client.cpp](./select_client.cpp)

- 参考： 

[https://www.cnblogs.com/Anker/p/3258674.html](https://www.cnblogs.com/Anker/p/3258674.html)

---
## poll
- poll管理多个文件描述符，和select一样，采用轮训方式，相对于select没有最大文件描述符限制，但也存在大量文件描述符数组被整体复制于内核态和用户态地址之间.
- 函数原型
```cpp
#include <poll.h>
// fds指向数组结构体第一个元素指针， nfds为数组元素个数，timeout为超时等待时间(毫秒)。
int poll(struct pollfd *fds, nfds_t nfds, int timeout);

// pollfd结构
struct pollfd{
	int fd;		    //文件描述符
	short events;   //等待的事件
	short revents;	//实际发生的事件（输出）
};
```
- pollfd事件变化

![pollfd事件变化](../pic/poll_event.png)

- 超时timeout

  - `-1` ：永远等待直到事件发生
  - `0 `：立即返回
  - `> 0`：等待指定毫秒后返回

- poll返回值
  - 成功时，poll()返回结构体中revents域不为0的文件描述符个数
  - 如果在超时前没有任何事件发生，poll()返回 0
  - 失败时，poll() 返回 -1。

- 优缺点
  - 优点：相比select没有最大描述符数量限制（链表）
  - 缺点
    - 仍然是线性扫描，随着描述符数量增多而时间变多
	- 仍然需要将大量描述符整体复制于用户态和内核的地址空间之间
 - 代码： [poll_service.cpp](./poll_service.cpp) [poll_client.cpp](./poll_client.cpp)
 - 参考
 
 [https://blog.csdn.net/lianghe_work/article/details/46534029](https://blog.csdn.net/lianghe_work/article/details/46534029)
 
 [https://www.cnblogs.com/Anker/p/3261006.html](https://www.cnblogs.com/Anker/p/3261006.html)
 
 ---
 
 ## epoll
 linux特有，epoll是在2.6内核中提出的，是之前的select和poll的增强版本。相对于select和poll来说，epoll更加灵活，没有描述符限制。epoll使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的copy只需一次。

 
- 函数原型
```cpp
#include <sys/epoll.h>
// 创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大
int epoll_create(int size);

// epoll的事件注册函数, 不同与select()是在监听事件时告诉内核要监听什么类型的事件，而是在这里先注册要监听的事件类型。
//     1. epfd是epoll_create()的返回值; 2. op表示动作:EPOLL_CTL_ADD/EPOLL_CTL_MOD/EPOLL_CTL_DEL; 3. fd表示要监听的描述符
//     4. event 告诉内核需要监听什么事：EPOLLIN(可读)、EPOLLOUT(可写)、EPOLLPRI(紧急可读)、EPOLLERR(错误)、EPOLLHUP(被挂起)、
//        EPOLLET(边缘触发)、EPOLLONESHOT(只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里)
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);

// 等待事件的产生, 函数返回需要处理的事件数目，如返回0表示已超时. maxevents表示events有多大，不能大于创建时的size
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);


// epoll_event结构体
struct epoll_event {
  __uint32_t events;  /* Epoll events */
  epoll_data_t data;  /* User data variable */
};
```
- 内部数据结构： 一棵红黑数据保存所有文件描述符，当某个描述符发生状态变化时，将其拷贝至一个链表中，epoll_wait返回这个链表。
[https://www.cnblogs.com/pluser/p/epoll_principles.html](https://www.cnblogs.com/pluser/p/epoll_principles.html)

- 优点：
  - 非线性扫描
  - 不用频繁拷贝大量文件描述符与用户态和内核空间之间
- 代码 [epoll_client.cpp](./epoll_client.cpp) [epoll_client.cpp](./epoll_client.cpp)

- 参考
[https://www.cnblogs.com/Anker/p/3263780.html](https://www.cnblogs.com/Anker/p/3263780.html)


---
 
 ## 参考：
 
 [https://www.cnblogs.com/aspirant/p/9166944.html](https://www.cnblogs.com/aspirant/p/9166944.html)
 	
 
