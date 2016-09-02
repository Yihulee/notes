## **一**、`struct timeval`结构体

`struct timeval`结构体在`time.h`中的定义为：

```c
struct timeval
{
__time_t tv_sec;        /* Seconds. */
__suseconds_t tv_usec;  /* Microseconds. */
};
```

其中，`tv_sec`为`Epoch`到创建`struct timeval`时的秒数，`tv_usec`为微秒数，即秒后面的零头。比如当前我写博文时的tv_sec为`1244770435`，tv_usec为`442388`，即当前时间距Epoch时间`1244770435`秒，`442388`微秒。

需要注意的是，因为循环过程，新建结构体变量等过程需消耗部分时间，我们作下面的运算时会得到如下结果：

```c
#include <sys/time.h>
#include <stdio.h>
  
int
main(void)
{
        int i;
        struct timeval tv;

        for(i = 0; i < 4; i++){
                gettimeofday(&tv, NULL);
                printf("%d\t%d\n", tv.tv_usec, tv.tv_sec);
                sleep(1);
        }

        return 0;
}
```

```c
329612	1314851429
329782	1314851430
329911	1314851431
330036	1314851432
```

前面为微秒数，后面为秒数，可以看出，在这个简单运算中，只能精确到小数点后面一到两位，或者可以看出，每进行一次循环，均需花费`0.005`秒的时间，用这个程序来作计时器显然是不行的，除非精确计算产生的代码消耗时间。

## **二**、`gettimeofday()`函数

原型：

```c
/* Get the current time of day and timezone information,
   putting it into *TV and *TZ. If TZ is NULL, *TZ is not filled.
   Returns 0 on success, -1 on errors.
   NOTE: This form of timezone information is obsolete.
   Use the functions and variables declared in <time.h> instead. */
extern int gettimeofday (struct timeval *__restrict __tv,
                         __timezone_ptr_t __tz) __THROW __nonnull ((1));
```

`gettimeofday()`功能是得到当前时间和时区，分别写到`tv`和`tz`中，如果`tz`为`NULL`则不向`tz`写入。



### 1、常用的时间存储方式  

1）`time_t`类型，这本质上是一个长整数，表示从`1970-01-01 00:00:00`到目前计时时间的秒数，如果需要更精确一点的，可以使用`timeval`精确到毫秒。  

2）`tm`结构，这本质上是一个结构体，里面包含了各时间字段  

```c
struct tm {  

        int tm_sec;     /* seconds after the minute - [0,59] */  

        int tm_min;     /* minutes after the hour - [0,59] */  

        int tm_hour;    /* hours since midnight - [0,23] */  

        int tm_mday;    /* day of the month - [1,31] */  

        int tm_mon;     /* months since January - [0,11] */  

        int tm_year;    /* years since 1900 */  

        int tm_wday;    /* days since Sunday - [0,6] */  

        int tm_yday;    /* days since January 1 - [0,365] */  

        int tm_isdst;   /* daylight savings time flag */  

  };  

```

其中`tm_year`表示从`1900`年到目前计时时间间隔多少年，如果是手动设置值的话，`tm_isdst`通常取值-1。  

### 2、常用的时间函数  

```c
time_t time(time_t *t); // 取得从1970年1月1日至今的秒数  

char *asctime(const struct tm *tm); // 将结构中的信息转换为真实世界的时间，以字符串的形式显示  

char *ctime(const time_t *timep); // 将timep转换为真是世界的时间，以字符串显示，它和asctime不同就在于传入的参数形式不一样  

struct tm *gmtime(const time_t *timep); // 将time_t表示的时间转换为没有经过时区转换的UTC时间，是一个struct tm结构指针   

struct tm *localtime(const time_t *timep); // 和gmtime类似，但是它是经过时区转换的时间。  

time_t mktime(struct tm *tm); // 将struct tm 结构的时间转换为从1970年至今的秒数  

int gettimeofday(struct timeval *tv, struct timezone *tz); // 返回当前距离1970年的秒数和微妙数，后面的tz是时区，一般不用  

double difftime(time_t time1, time_t time2); // 返回两个时间相差的秒数  
```

### 3、时间与字符串的转换  

  

需要包含的头文件如下  

```c
#include <time.h>  
#include <stdlib.h>  
#include <string.h>  
```

**1）**`unix/windows`下时间转字符串参考代码  

```c
time_t t;  //秒时间  

tm* local; //本地时间   

tm* gmt;   //格林威治时间  

char buf[128]= {0};  

  

t = time(NULL); //获取目前秒时间  

local = localtime(&t); //转为本地时间  

strftime(buf, 64, "%Y-%m-%d %H:%M:%S", local);  

std::cout << buf << std::endl;  

  

gmt = gmtime(&t);//转为格林威治时间  

strftime(buf, 64, "%Y-%m-%d %H:%M:%S", gmt);  

std::cout << buf << std::endl;  
```

**2）**unix字符串转时间参考代码  

```c
tm tm_;  

time_t t_;  

char buf[128]= {0};  

  

strcpy(buf, "2012-01-01 14:00:00");  

strptime(buf, "%Y-%m-%d %H:%M:%S", &tm_); //将字符串转换为tm时间  

tm_.tm_isdst = -1;  

t_  = mktime(&tm_); //将tm时间转换为秒时间  

t_ += 3600;  //秒数加3600  

  

tm_ = *localtime(&t_);//输出时间  

strftime(buf, 64, "%Y-%m-%d %H:%M:%S", &tm_);  

std::cout << buf << std::endl;  
  
```



**简介**
本文旨在为了解`Linux`各种时间类型与时间函数提供技术文档。



### **1、`Linux`下常用时间类型**

`Linux`下常用时间类型有四种：`time_t`、`struct tm`、`struct timeval`、`struct timespec`.

#### 1.1 `time_t`时间类型

`time_t`类型在`time.h`中定义：

```c
#ifndef __TIME_T
#define __TIME_T
typedef  long  time_t;
#endif
```

可见，`time_t`实际是一个长整型。其值表示为从`UTC(coordinated universal time)`时间`1970`年`1`月`1`日`00`时`00`分`00`秒(也称为`Linux`系统的`Epoch`时间)到当前时刻的秒数。由于`time_t`类型长度的限制，它所表示的时间不能晚于`2038`年`1`月`19`日`03`时`14`分`07`秒(`UTC`)。为了能够表示更久远的时间，可用`64`位或更长的整形数来保存日历时间，这里不作详述。

使用`time()`函数获取当前时间的`time_t`值，使用`ctime()`函数将`time_t`转为当地时间字符串。

**备注**：`UTC`时间有时也称为`GMT`时间，其实`UTC`和`GMT`两者几乎是同一概念。它们都是指格林尼治标准时间，只不过`UTC`的称呼更为正式一点。两者区别在于前者是天文上的概念，而后者是基于一个原子钟。

#### 1.2 `struct tm`时间类型

`tm`结构在`time.h`中定义：

```c
#ifndef _TM_DEFINED
struct tm{
    int tm_sec; /*秒 - 取值区间为[0, 59]*/
    int tm_min; /*分 - 取值区间为[0, 59]*/
    int tm_hour; /*时 - 取值区间为[0, 23]*/
    int tm_mday; /*日 - 取值区间为[1, 31]*/
    int tm_mon; /*月份 - 取值区间为[0, 11]*/
    int tm_year; /*年份 - 其值为1900年至今年数*/
    int tm_wday; /*星期 - 取值区间[0, 6]，0代表星期天，1代表星期1，以此类推*/
    int tm_yday; /*从每年的1月1日开始的天数-取值区间为[0, 365]，0代表1月1日*/
    int tm_isdst; /*夏令时标识符，使用夏令时，tm_isdst为正，不使用夏令时，tm_isdst为0，不了解情况时，tm_isdst为负*/
};
#define _TM_DEFINED
#endif
```

`ANSI C`标准称使用`tm`结构的这种时间表示为分解时间(`broken-down time`)。

使用`gmtime( )`和`localtime( )`可将`time_t`时间类型转换为`tm`结构体；

使用`mktime( )`将`tm`结构体转换为`time_t`时间类型；

使用`asctime( )`将`struct tm`转换为字符串形式。

#### 1.3 `struct timeval`时间类型

`timeval`结构体在`time.h`中定义：

```c
Struct tmieval{
    time_t tv_sec; /*秒s*/
    suseconds_t tv_usec; /*微秒us*/
};

```

这个结构体的值记录了从`UTC(coordinated universal time)`时间`1970`年`1`月`1`日`00`时`00`分`00`秒(也称为`Linux`系统的`Epoch`时间)到当前时刻的秒数和微秒数。

设置时间函数`settimeofday( )`与获取时间函数`gettimeofday( )`均使用该事件类型作为传参。

#### 1.4 `struct timespec`时间类型

`timespec`结构体在`time.h`定义：

```c
struct timespec{
    time_t tv_sec; /*秒s*/
    long tv_nsec; /*纳秒ns*/
};
```

### 2、`Linux`下常用时间函数

`Linux`下常用时间函数有：`time( )`、`ctime( )`、`gmtime( )`、`localtime( )`、`mktime( )`、`asctime( )`、`difftime( )`、`gettimeofday( )`、`settimeofday( )`

#### 2.1 `time( )`函数

头文件：`#include <time.h>`
函数定义：`time_t time(time_t *timer)`
功能描述：该函数返回从`1970`年`1`月`1`日`00`时`00`分`00`秒至今所经过的秒数。如果`time_t *timer`非空指针，函数也会将返回值存到`timer`指针指向的内存。
返回值：成功则返回秒数，失败则返回`((time_t)-1)`值，错误原因存于`errno`中。
例：

```c
time_t seconds;
seconds = time((time_t *)NULL);
```

#### 2.2 `ctime( )`函数

头文件：`#include <time.h>`
函数定义：`char *ctime(const time_t *timep);`
功能描述：`ctime( )`将参数`timep`指向的`time_t`时间信息转换成实际所使用的时间日期表示方法，并以字符串形式返回。字符串格式为："`Wed Jun 20 21:00:00 2012\n`"。
例：

```c
time_t timep;
tmep = time(NULL);
printf("%s\n", ctime(&timep));
```

#### 2.3 `gmtime( )`函数

头文件：`#include <time.h>`
函数定义：`struct tm *gmtime(const time_t *timep)`
功能描述：`gmtime( )`将参数`timep`指向的`time_t`时间信息转换成以`tm`结构体表示的`GMT`时间信息，并以`struct tm*`指针返回。
`GMT`：`GMT`是中央时区,北京在东`8`区,相差`8`个小时，所以北京时间=`GMT`时间+`8`小时。
例：

```c
int main(void)
{
    char *wday[] = {"Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"};
    time_t timep;
    struct tm *p_tm;
    timep = time(NULL);
    p_tm = gmtime(&timep); /*获取GMT时间*/
    printf("%d-%d-%d ", (p_tm->tm_year+1900), (p_tm->mon+1), p_tm->tm_mday);
    printf("%s %d:%d:%d\n", wday[p_tm->tm_wday], p_tm->tm_hour, p_tm->tm_min, p_tm->tm_sec);
}
```

#### 2.4 `localtime( )`函数

头文件：`#include <time.h>`
函数定义：`struct tm *localtime(const time_t *timep);`
功能描述：`localtime( )`将参数`timep`指向的`time_t`时间信息转换成以`tm`结构体表示的本地时区时间(如北京时间= `GMT`+`8`小时)。
例：

```c
int main(void)
{
    char *wday[] = {"Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"};
    time_t timep;
    struct tm *p_tm;
    timep = time(NULL);
    p_tm = localtime(&timep); /*获取本地时区时间*/
    printf("%d-%d-%d ", (p_tm->tm_year+1900), (p_tm->mon+1), p_tm->tm_mday);
    printf("%s %d:%d:%d\n", wday[p_tm->tm_wday], p_tm->tm_hour, p_tm->tm_min, p_tm->tm_sec);
    return 0;
}
```

**2.5 `mktime( )`函数**
头文件：`#include <time.h>`
函数定义：`time_t mktime(struct tm *p_tm);`
功能描述：`mktime( )`将参数`p_tm`指向的`tm`结构体数据转换成从`1970`年`1`月`1`日`00`时`00`分`00`秒至今的`GMT`时间经过的秒数。

```c
int main(void)
{
    time_t timep:
    struct tm *p_tm;
    timep = time(NULL);
    pintf("time( ):%d\n", timep);
    p_tm = local(&timep);
    timep = mktime(p_tm);
    printf("time( )->localtime( )->mktime( ):%d\n", timep);
    return 0;
}
```

#### 2.6 `asctime( )`函数

头文件：`#include <time.h>`
函数定义：`char *asctime(const struct tm *p_tm);`
功能描述：`asctime( )`将参数`p_tm`指向的`tm`结构体数据转换成实际使用的时间日期表示方法，并以字符串形式返回(与`ctime`函数相同)。字符串格式为："`Wed Jun 20 21:00:00 2012\n`"。
例：

```c
int main(void)
{
    time_t timep;
    timep = time(NULL);
    printf("%s\n", asctime(gmtime(&timep)));
    return 0;
}
```

#### 2.7 `difftime( )`函数

头文件：`#include <time.h>`
函数定义：`double difftime(time_t timep1, time_t timep2);`
功能描述：`difftime( )`比较参数`timep1`和`timep2`时间是否相同，并返回之间相差秒数。
例：

```c
int main(void)
{
    time_t timep1, timep2;
    timep1 = time(NULL);
    sleep(2);
    timep2 = time(NULL);
    printf("the difference is %f seconds\n", difftime(timep1, timep2));
    return 0;
}
```



#### 2.8 `gettimeofday( )`函数

头文件：

````c
#include <sys/time.h>
#include <unistd.h>  
````

函数定义：`int gettimeofday(struct timeval *tv, struct timezone *tz);`
功能描述：`gettimeofday( )`把目前的时间信息存入`tv`指向的结构体，当地时区信息则放到`tz`指向的结构体。
`struct timezone`原型：

```c
struct timezone{
    int tz_minuteswest; /*miniutes west of Greenwich*/
    int tz_dsttime; /*type of DST correction*/
};
```

例:

```c
struct timeval tv;
struct timeval tz;
gettimeofday(&tv, &tz);
```

附：
使用`time`函数族获取时间并输出指定格式字符串例子（`strftime( )`函数）：

```c
int main(void)
{
    char strtime[20] = {0};
    time_t timep;
    struct tm *p_tm;
    timep = time(NULL);
    p_tm = localtime(&timep);
    strftime(strtime, sizeof(strtime), "%Y-%m-%d %H:%M:%S", p_tm);
    return 0;
}
```

#### 2.9 `settimeofday( )`函数

头文件：

```c
#include <unistd.h>
#include <sys/time.h> 
```

函数定义：`int settimeofday(const struct timeval *tv, const struct timezone *gz);`
功能描述：`settimeofday( )`把当前时间设成由`tv`指向的结构体数据。当前地区信息则设成`tz`指向的结构体数据。
例：

```c
int main(void)
{
    char t_string[] = "2012-04-28 22:30:00";
    struct tm time_tm;
    struct timeval time_tv;
    time_t timep;
    int ret = 0;

    sscanf(t_string, "%d-%d-%d %d:%d:%d", &time_tm.tm_year, &time_tm.tm_mon, &time_tm.tm_mday, &time_tm.tm_hour, &time_tm.tm_min, &time_tm.tm_sec);
    time_tm.tm_year -= 1900;
    time_tm.tm_mon -= 1;
    time_tm.tm_wday = 0;
    time_tm.tm_yday = 0;
    time_tm.tm_isdst = 0;

    timep = mktime(&time_tm);
    time_tv.tv_sec = timep;
    time_tv.tv_usec = 0;

    ret = settimeofday(&time_tv, NULL);
    if(ret != 0)
    {
        fprintf(stderr, "settimeofday failed\n");
        return -1;
    }
    return 0;
}
```







