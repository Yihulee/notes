

## 三种新的`fd`加入`linux`内核的的版本

`signalfd`：2.6.22

`timerfd`：2.6.25

`eventfd`：2.6.22



## 三种`fd`的意义

`signalfd`：传统的处理信号的方式是注册信号处理函数；由于信号是异步发生的，要解决数据的并发访问，可重入问题。`signalfd`可以将信 号抽象为一个文件描述符，当有信号发生时可以对其`read`，这样可以将信号的监听放到`select`、`poll`、`epoll`等监听队列中。

`timerfd`：可以实现定时器的功能，将定时器抽象为文件描述符，当定时器到期时可以对其`read`，这样也可以放到监听队列的主循环中。

`eventfd`：实现了线程之间事件通知的方式，`eventfd`的缓冲区大小是`sizeof(uint64_t)`；向其`write`可以递增这个计数 器，`read`操作可以读取，并进行清零；`eventfd`也可以放到监听队列中，当计数器不是`0`时，有可读事件发生，可以进行读取。



**三种新的fd都可以进行监听，当有事件触发时，有可读事件发生。**

## 对应的`API`

**1)** `sigalfd`

```c
#include <sys/signalfd.h>  
int signalfd(int fd, const sigset_t *mask, int flags);  
```

参数`fd`：如果是`-1`则表示新建一个，如果是一个已经存在的则表示修改`signalfd`所关联的信号；

参数`mask`：信号集合；

参数`flag`：内核版本`2.6.27`以后支持`SFD_NONBLOCK`、`SFD_CLOEXEC`；

成功返回文件描述符，返回的`fd`支持以下操作：`read`、`select(poll、epoll)`、`close`

**2)** timerfd

###`timerfd_create` 
```c
#include <sys/timerfd.h>
int timerfd_create(int clockid, int flags);  
```
创建一个`timerfd`，返回的`fd`可以进行如下操作：`read`、`select(poll、epoll)`、`close`  ，关于`clockid`，可以取已下的参数：

+ `CLOCK_REALTIME` 参考系统的实时时间，如果系统时间被改变，定时器的参考时间也改变

+ `CLOCK_MONOTONIC` 参考建定时器时的时间，如果之后系统时间被修改， 定时器里的参考时间不变


`flags` 参数在 `linux 2.6.26` 版本之后，必须为`0`。

函数执行成功返回一个文件描述符，失败的话返回`-1`,错误码需要查`errno`。

错误码如下:

- `EINVAL` 不能识别`clockid`
- `EINVAL flags` 无效， 或者，在 `2.6.26 `及其前，`flags` 非零
- `EMFILE` 达到单个进程打开的文件描述上限。
- `ENFILE` 达到可打开文件个数的系统全局上限。
- `ENODEV` 不能挂载（内部）匿名结点设备。
- `ENOMEM` 内存不足


### `timerfd_settime`  
```c
#include <sys/timerfd.h>
int timerfd_settime(int fd, int flags,  
                    const struct itimerspec *new_value,  
                    struct itimerspec *old_value);  
```
- 用于设置`timer`的周期，以及起始间隔。
- `fd` 就是我们的定时器文件描述符。
- `flags` `1`代表设置的是绝对时间；为`0`代表相对时间。绝对时间的意思是到那个时间点触发事件,相对时间是多少时间之后触发事件
- new_value 需要设置的时间
- old_value 需要返回的上次设置的时间，一般传NULL指针

###`timerfd_gettime`
```c
#include <sys/timerfd.h>
int timerfd_gettime(int fd, struct itimerspec *curr_value);  
```
获取到期时间。 有时候，我们需要手动定时器时，可能需要查询上次设置的时间，这时可以使用 timerfd_gettime 函数。

函数参数中数据结构如下：  

```c
struct timespec  
{  
    time_t tv_sec;                /* Seconds */  
    long   tv_nsec;               /* Nanoseconds */  
};  
  
struct itimerspec  
{  
    struct timespec it_interval;  /* Interval for periodic timer */  
    struct timespec it_value;     /* Initial expiration */  
};  
```
一段简单的代码如下：
```c
//创建定时器
timer_id = timerfd_create(CLOCK_REALTIME, 0);
//设置非阻塞
flags = fcntl(fd, F_GETFL, 0);
flags |= O_NONBLOCK;
fcntl(fd, F_SETFL, flags)
//设置超时时间
double timer_internal = 3.2;
struct itimerspec ptime_internal;
memset(&ptime_internal, 0, sizeof(ptime_internal));
ptime_internal.it_value.tv_sec = (int) timer_internal;
ptime_internal.it_value.tv_nsec = (timer_internal - (int) timer_internal) * 1000000;
timerfd_settime(timer_id, 0, &ptime_internal, NULL);
//其他操作
```

**3)** `eventfd`

```c
#include <sys/eventfd.h>
int eventfd(unsigned int initval, int flags);  
```

创建一个`eventfd`，这是一个计数器相关的`fd`，计数器不为零是有可读事件发生，`read`以后计数器清零，`write`递增计数器；返回的`fd`可以进行如下操作：`read`、`write`、`select(poll、epoll)`、`close`

