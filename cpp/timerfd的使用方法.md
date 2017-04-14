# `timerfd`的使用方法

文章来自:[http://blog.csdn.net/yusiguyuan/article/details/22936707?utm_source=tuicool&utm_medium=referral](http://blog.csdn.net/yusiguyuan/article/details/22936707?utm_source=tuicool&utm_medium=referral)

`timerfd`是`Linux`为用户程序提供的一个定时器接口。这个接口基于文件描述符，通过文件描述符的可读事件进行超时通知，所以能够被用于`select/poll`的应用场景。`timerfd`是`linux`内核`2.6.25`版本中加入的接口。

`timerfd`、`eventfd`、`signalfd`配合`epoll`使用，可以构造出一个零轮询的程序，但程序没有处理的事件时，程序是被阻塞的。这样的话在某些移动设备上程序更省电。

`clock_gettime`函数可以获取系统时钟，精确到纳秒。需要在编译时指定库：`-lrt`。可以获取两种类型事件：

`CLOCK_REALTIME`：相对时间，从`1970.1.1`到目前的时间。更改系统时间会更改获取的值。也就是，它以系统时间为坐标。

`CLOCK_MONOTONIC`：与`CLOCK_REALTIME`相反，它是以绝对时间为准，获取的时间为系统重启到现在的时间，更改系统时间对齐没有影响。

`timerfd_create`：

生成一个定时器对象，返回与之关联的文件描述符。接收两个入参，一个是`clockid`，填写`CLOCK_REALTIME`或者`CLOCK_MONOTONIC`，参数意义同上。第二个可以传递控制标志：`TFD_NONBLOCK`（非阻塞），`TFD_CLOEXEC`（同`O_CLOEXEC`）

注：`timerfd`的进度要比`usleep`要高。

`timerfd_settime`：能够启动和停止定时器；可以设置第二个参数：`flags`，`0`表示是相对定时器，`TFD_TIMER_ABSTIME`表示是绝对定时器。

第三个参数设置超时时间，如果为`0`则表示停止定时器。定时器设置超时方法：

1. 设置超时时间是需要调用`clock_gettime`获取当前时间，如果是绝对定时器，那么需要获取`CLOCK_REALTIME`，在加上要超时的时间。如果是相对定时器，要获取`CLOCK_MONOTONIC`时间。
2. 数据结构： 

```c++
struct timespec {
  time_t tv_sec;                /* Seconds */
  long   tv_nsec;               /* Nanoseconds */
};

struct itimerspec {
  struct timespec it_interval;  /* Interval for periodic timer */
  struct timespec it_value;     /* Initial expiration */
};
```

 `it_value`是首次超时时间，需要填写从`clock_gettime`获取的时间，并加上要超时的时间。 `it_interval`是后续周期性超时时间，是多少时间就填写多少。

 注意一个容易犯错的地方：**`tv_nsec`加上去后一定要判断是否超出`1000000000`（如果超过要秒加一），否则会设置失败。**

     

     `it_interval`不为`0`则表示是周期性定时器。

     `it_value`和`it_interval`都为`0`表示停止定时器。

注：`timerfd_create`第一个参数和`clock_gettime`的第一个参数都是`CLOCK_REALTIME`或者`CLOCK_MONOTONIC`，`timerfd_settime`的第二个参数为`0`（相对定时器）或者`TFD_TIMER_ABSTIME`，三者的关系：

1. 如果`timerfd_settime`设置为`TFD_TIMER_ABSTIME`（决定时间），则后面的时间必须用`clock_gettime`来获取，获取时设置`CLOCK_REALTIME`还是`CLOCK_MONOTONIC`取决于`timerfd_create`设置的值。
2. 如果`timerfd_settime`设置为`0`（相对定时器），则后面的时间必须用相对时间，就是：

```c++
new_value.it_value.tv_nsec = 500000000;

new_value.it_value.tv_sec = 3;

new_value.it_interval.tv_sec = 0;

new_value.it_interval.tv_nsec = 10000000;
```

`read`函数可以读`timerfd`，读的内容为`uint_64`，表示超时次数。

看一段代码例子：

```c++
#include <sys/timerfd.h>
#include <time.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>
#include <stdint.h>  // Definition of uint64_t

#define handle_error(msg) \
               do { perror(msg); exit(EXIT_FAILURE); } while (0)

static void
print_elapsed_time(void) // 用于输出消逝的时间
{
	static struct timespec start; // 开始的时间
	struct timespec curr; // 现在的时间
	static int first_call = 1; // first_call是一个静态变量,只会初始化一次
	int secs, nsecs;

	if (first_call) { 
		first_call = 0; // 也就是说,仅第1次调用print_elapsed_time函数才会运行到这里
      	// 并且记录下第1次调用该函数时的时间,以后再次调用该函数,就可以获得差值了.
		if (clock_gettime(CLOCK_MONOTONIC, &start) == -1)
			handle_error("clock_gettime");
	}

	if (clock_gettime(CLOCK_MONOTONIC, &curr) == -1)
		handle_error("clock_gettime");

	secs = curr.tv_sec - start.tv_sec;
	nsecs = curr.tv_nsec - start.tv_nsec; // 得到curr和start之间的差值
	if (nsecs < 0) {
		secs--;
		nsecs += 1000000000;
	}
	printf("%d.%03d: ", secs, (nsecs + 500000) / 1000000);
}

int
main(int argc, char *argv[])
{
	struct itimerspec new_value;
	int max_exp, fd;
	struct timespec now;
	uint64_t exp, tot_exp;
	ssize_t s;

	if ((argc != 2) && (argc != 4)) {
		fprintf(stderr,
			"%s init-secs [interval-secs max-exp]\n",
			argv[0]);
		exit(EXIT_FAILURE);
	}

	if (clock_gettime(CLOCK_REALTIME, &now) == -1) // 得到现在的时间
		handle_error("clock_gettime");

  	// Create a CLOCK_REALTIME absolute timer with initial
  	// expiration and interval as specified in command line
	//printf("%s %s %s\n", argv[1], argv[2], argv[3]);
	new_value.it_value.tv_sec = now.tv_sec + atoi(argv[1]); // it_value指的是第一次到期的时间
	new_value.it_value.tv_nsec = now.tv_nsec; 
	if (argc == 2) { // 两个参数,代表不用重复
		new_value.it_interval.tv_sec = 0;  
		max_exp = 1; // 代表下面的read仅调用1次
	}
	else { // 一共有三个参数
		new_value.it_interval.tv_sec = atoi(argv[2]); // 接下来是隔argv[2]秒后到期
		max_exp = atoi(argv[3]);
	}
	new_value.it_interval.tv_nsec = 0;

	fd = timerfd_create(CLOCK_REALTIME, 0); // 构建了一个定时器
	if (fd == -1)
		handle_error("timerfd_create");

	if (timerfd_settime(fd, TFD_TIMER_ABSTIME, &new_value, NULL) == -1)
		handle_error("timerfd_settime");

	print_elapsed_time();
	printf("timer started\n"); // 定时器开启啦！

	for (tot_exp = 0; tot_exp < max_exp;) { // max_exp原来是次数，是吧！
		s = read(fd, &exp, sizeof(uint64_t)); // 也就是说，如果fd不是非阻塞的,那么程序会阻塞在这里
		if (s != sizeof(uint64_t))
			handle_error("read");

		tot_exp += exp;
		print_elapsed_time();
		printf("read: %llu; total=%llu\n",
			(unsigned long long) exp,
			(unsigned long long) tot_exp);
	}

	exit(EXIT_SUCCESS);
}
```

```shel
root@node1:/home/c_test/unix_test# ./timerfd 20 3 4
printTime:  current time:1396594376.746760 timer started
printTime:  current time:1396594396.747705 read: 1; total=1
printTime:  current time:1396594399.747667 read: 1; total=2
printTime:  current time:1396594402.747728 read: 1; total=3
printTime:  current time:1396594405.746874 read: 1; total=4
```

第一个参数为第一次定时器到期间隔，第二个参数为定时器的间隔，第三个参数为定时器多少次则退出。 

`timerfd`简单的性能测试：

申请`1000`个定时器，超时定为`1s`，每秒超时一次，发现`cpu`占用率在`3.0G`的`cpu`上大概为`1%`，`10000`个定时器的话再`7%`左右，而且不会出现同时超时两个的情况，如果有`printf`到前台，则一般会出现定时器超时多次（`3-5`）才回调。 

`PS`:**`linux`内核新添加的`API` `timerfd`、`signalfd`、`eventfd`都有异曲同工之妙，都可以将本来复杂的处理转化思维变得简单。**