

原文在这里[ 浅析open函数O_CLOEXEC模式和fcntl函数FD_CLOEXEC选项 ](http://blog.chinaunix.net/uid-24907956-id-3969651.html)

关于`open`函数`O_CLOEXEC`模式，`fcntl`函数`FD_CLOEXEC`选项，总结为如下几点：

1. 调用`open`函数`O_CLOEXEC`模式打开的文件描述符在执行`exec`调用新程序中关闭，且为原子操作。

2. 调用`open`函数不使用`O_CLOEXEC`模式打开的文件描述符，然后调用`fcntl` 函数设置`FD_CLOEXEC`选项，效果和使用`O_CLOEXEC`选项`open`函数相同，但分别调用`open`、`fcnt`两个l函数，不是原子操作，多线程环境中存在竞态条件，故用`open`函数`O_CLOEXEC`选项代替之。

3. 调用`open`函数`O_CLOEXEC`模式打开的文件描述符，或是使用`fcntl`设置`FD_CLOEXEC`选项，这二者得到（处理）的描述符在通过`fork`调用产生的子进程中均不被关闭。

4. 调用`dup`族类函数得到的新文件描述符将清除`O_CLOEXEC`模式。

简单的测试代码如下：
```c
#include <unistd.h>
#include <sys/types.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>

#define err_sys(fmt, arg...) \
do { \
    printf(fmt, ##arg);\
    printf("\nerrno:%d %s\n", errno, strerror(errno));\
    exit(EXIT_FAILURE);\
} while (0)


int main()
{
    int fd, fd2, val;
    pid_t pid;
#ifdef _O_CLOEXEC
    if ((fd = open("my.txt", O_RDWR | O_CREAT | O_CLOEXEC, 0600)) < 0) 
#else
    if ((fd = open("my.txt", O_RDWR | O_CREAT, 0600)) < 0) 
#endif
        err_sys("open error");

#ifdef _DUP
    if ((fd2 = dup(fd)) < 0)
        err_sys("dup error");
    if (write(fd2, "123", 3) < 0)
        err_sys("write error");
#endif

#ifdef _FCNTL_CLOEXEC
    if ((val = fcntl(fd, F_GETFD)) < 0)
        err_sys("fcntl(F_GETFD) error");

    val |= FD_CLOEXEC;
    if (fcntl(fd, F_SETFD, val) < 0)
        err_sys("fcntl( F_SETFD) error");
#endif

#ifndef _FORK 
    if (execl("/bin/sleep", "sleep", "10000", (void*)0) < 0)
        err_sys("execl error");
#else
 switch ((pid = fork())) {
        case -1:
            err_sys("fork error");
        case 0:
            sleep(10000);
            break;
        default:
            sleep(10000);
            break;
    }
#endif

    return 0;
}

```
通过宏`_O_CLOEXEC`编译进`O_CLOEXEC`选项
`# gcc -D_O_CLOEXEC -o cloexec cloexec.c`
执行程序
`#  ./cloexec`

查看进程和进程资源，注意执行`execl`后进程名字变为`sleep`了。
```bash
# ps aux|grep sleep 
root      7900  0.0  0.0   4200   280 pts/0    S+   11:45   0:00 sleep 10000
root      7915  0.0  0.1   4384   836 pts/1    S+   11:49   0:00 grep --color=auto sleep

# lsof -p 7900     
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF   NODE NAME
sleep   7900 root  cwd    DIR    8,1     4096 163741 /home/zozy/AUP
sleep   7900 root  rtd    DIR    8,1     4096      2 /
sleep   7900 root  txt    REG    8,1    26156 131621 /bin/sleep
sleep   7900 root  mem    REG    8,1  2919792     12 /usr/lib/locale/locale-archive
sleep   7900 root  mem    REG    8,1  1730024  15701 /lib/i386-linux-gnu/libc-2.15.so
sleep   7900 root  mem    REG    8,1   134344  15713 /lib/i386-linux-gnu/ld-2.15.so
sleep   7900 root    0u   CHR  136,0      0t0      3 /dev/pts/0
sleep   7900 root    1u   CHR  136,0      0t0      3 /dev/pts/0
sleep   7900 root    2u   CHR  136,0      0t0      3 /dev/pts/0

```
可以看出进程资源中没有文件`my.txt`。

不带宏`_O_CLOEXEC`编译
`# gcc  -o nocloexec cloexec.c`
执行程序
`# ./nocloexec` 

查看进程和进程资源:
```bash
# ps aux|grep sleep
root      7925  0.0  0.0   4200   280 pts/0    S+   11:51   0:00 sleep 10000
root      7928  0.0  0.1   4384   836 pts/1    S+   11:52   0:00 grep --color=auto sleep

# lsof -p 7925
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF   NODE NAME
sleep   7925 root  cwd    DIR    8,1     4096 163741 /home/zozy/AUP
sleep   7925 root  rtd    DIR    8,1     4096      2 /
sleep   7925 root  txt    REG    8,1    26156 131621 /bin/sleep
sleep   7925 root  mem    REG    8,1  2919792     12 /usr/lib/locale/locale-archive
sleep   7925 root  mem    REG    8,1  1730024  15701 /lib/i386-linux-gnu/libc-2.15.so
sleep   7925 root  mem    REG    8,1   134344  15713 /lib/i386-linux-gnu/ld-2.15.so
sleep   7925 root    0u   CHR  136,0      0t0      3 /dev/pts/0
sleep   7925 root    1u   CHR  136,0      0t0      3 /dev/pts/0
sleep   7925 root    2u   CHR  136,0      0t0      3 /dev/pts/0
sleep   7925 root    3u   REG    8,1        0 163759 /home/zozy/AUP/my.txt # 就是这里
```

可以看出进程资源中有文件`my.txt`。


测试`fcntl`函数可以通过设置编译宏`-D_FCNTL_CLOEXEC`测试，编译测试过程同上，同理通过开启`-D_O_CLOEXEC -D_FORK`编译选项测试使用`O_CLOEXEC`模式的描述符在子进程中的状态，通过宏`-D_DUP`编译选项测试`dup`函数对`O_CLOEXEC`的影响，编译测试过程略。

