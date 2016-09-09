今天，我要写一篇文章，好好来说一下我所理解的`ucontext`族函数。

>**NAME**
>`getcontext`, `setcontext` - get or set the user context
>**SYNOPSIS**
>```c
>#include <ucontext.h>
>int getcontext(ucontext_t *ucp);
>int setcontext(const ucontext_t *ucp);
>```
>**DESCRIPTION**
>In a System V-like environment, one has the two types `mcontext_t` and `ucontext_t` defined in <ucontext.h> and the four functions `getcontext()`, `setcontext()`, `makecontext`(3),  and `swapcontext`(3) that allow user-level context switching between multiple threads of control within a process.
>The `mcontext_t` type is machine-dependent and opaque.  The `ucontext_t` type is  a  structure that has at least the following fields:
>```c
>typedef struct ucontext {
>struct ucontext *uc_link;
>sigset_t         uc_sigmask;
>stack_t          uc_stack;
>mcontext_t       uc_mcontext;
>...
>} ucontext_t;
>```
>
>with  `sigset_t`  and  `stack_t` defined in <signal.h>.  Here `uc_link` points to the context that will be resumed when the current context terminates (in case the  current  context was  created  using  `makecontext`(3)),  uc_sigmask is the set of signals blocked in this context (see `sigprocmask`(2)), uc_stack is the stack used by this context  (see  sigaltstack(2)), and uc_mcontext is the machine-specific representation of the saved context,that includes the calling thread's machine registers.
>
>The function `getcontext()` initializes the structure pointed at by ucp to the  currently active context.
>
>The  function  `setcontext()`  restores the user context pointed at by ucp.  A successful call does not return.  The context should have been obtained by a call of `getcontext()`, or `makecontext`(3), or passed as third argument to a signal handler.
>
>If  the  context was obtained by a call of `getcontext()`, program execution continues as if this call just returned.
>
>If the context was obtained by a call of `makecontext`(3), program execution continues by
>a  call  to the function func specified as the second argument of that call to `makecontext`(3).  When the function func returns, we continue with the uc_link  member  of  the structure  ucp  specified  as  the first argument of that call to makecontext(3).  When this member is `NULL`, the thread exits.
>
>If the context was obtained by a call to a signal handler, then old standard text  says
>that  "program  execution continues with the program instruction following the instruction interrupted by the signal".  However, this sentence was removed in SUSv2, and  the present verdict is "the result is unspecified".
>
>**RETURN VALUE**
>When  successful,  `getcontext()`  returns 0 and `setcontext()` does not return.  On error,
>both return -1 and set errno appropriately.
>
>**ERRORS**
>None defined.


关于另外的一组函数在man手册上的说明：
> **NAME**
> `makecontext`, `swapcontext` - manipulate user context
>
> **SYNOPSIS**
>
> ```c
> #include <ucontext.h>
> void makecontext(ucontext_t *ucp, void (*func)(), int argc, ...);
> int swapcontext(ucontext_t *oucp, const ucontext_t *ucp);
> ```
> **DESCRIPTION**
> In a System V-like environment, one has the type `ucontext_t` defined in <ucontext.h> and the four functions `getcontext`(3), `setcontext`(3), `makecontext`() and  `swapcontext`()  that allow  user-level  context  switching  between  multiple  threads  of  control within a process.
>
> For the type and the first two functions, see `getcontext`(3).
>
> The `makecontext`() function modifies the context pointed to by ucp (which  was  obtained from a call to getcontext(3)).  Before invoking makecontext(), the caller must allocate a new stack for this context and assign its address to `ucp->uc_stack`, and define a successor context and assign its address to `ucp->uc_link`.
>
> When  this  context is later activated (using `setcontext`(3) or `swapcontext()`) the function func is called, and passed the series of integer (int) arguments that follow argc;the  caller  must  specify  the  number of these arguments in argc.  When this function returns, the successor context is activated.  *If the successor context pointer is NULL,the thread exits.*
>
> The  `swapcontext`()  function  saves  the current context in the structure pointed to by oucp, and then activates the context pointed to by ucp.
>
> **RETURN VALUE**
> When successful, `swapcontext()` does not return.  (But we may return later, in case oucp is activated, in which case it looks like `swapcontext()` returns 0.)  On error, `swapcontext()` returns -1 and sets errno appropriately.
>
> **ERRORS**
> ENOMEM Insufficient stack space left.
>



上面的东西大家读一读就好了，我这里稍微来说一下我的感受吧！

这一族函数在使用感受上非常类似于我们使用过的`goto`语句。

```c
int swapcontext(ucontext_t *oucp, const ucontext_t *ucp);
```

通过`swapcontext`函数，可以十分方便地实现从这个执行点跳跃到另外一个执行点。可是这个东西确实和`goto`语句有非常大的不同。因为它们可以记录下跳跃点的上下文，这就非常有意思了，我举一个不恰当的比喻，这就是魔法世界中的时间静止的魔法，巫师挥一挥魔杖，在这个点的一切信息都保留了下来，然后巫师跑到另外一个世界继续冒险。厌倦了之后，回到这个世界，一切又从原来静止的点开始执行。

```c
void makecontext(ucontext_t *ucp, void (*func)(), int argc, ...);
```

`makecontext`这个函数是用来干什么的呢？其实这个函数和构造线程的函数非常类似，不过，它构造的是一个线程，而你是制造一个新的`context`。这个context是你将来要进入的一个世界，所以，在进入之前，你首先要准备一些什么，`func`是你你要执行的函数(你的任务)，`argc`是这个函数的参数个数，而后面的可变部分，是你要传递给`func`的实参。在调用`makecontext`函数之前，我们首先要初始化`ucp`指向的`ucontext_t`结构体，正如前面所说的，`ucontext_t`结构只是一个规范而已，不同平台下定义的域都有可能有所不同，所以我们只能用`getcontext`函数来初始化这样一个结构体。另外一个比较重要的在`ucp`上要设置的一个域叫做`uc_link`，这个玩意也指向一个`ucontext_t`结构，这个玩意是在询问我们，这个`context`运行完成后，你要回到哪里？(当这个世界湮灭后，你要去哪里？)如果你不设置，相当于说，我们结束吧(你要和这个世界共同进退)。不运行了。如果设置了，那么会跳转到`uc_link`指向的`context`继续运行。

另外需要注意的一点是，我们还需要设置`ucontext_t`的`uc_stack`域，说白了，就是要给这个玩意配置一个栈，为什么要这么干，我们看一下下面的例子：

```c
#include <stdio.h>

void ping();
void pong();

void ping(){
    printf("ping\n");
    pong();
}

void pong(){
    printf("pong\n");
    ping();
}

int main(int argc, char *argv[]){
    ping();
    return 0;
}
```

上面的程序是一个循环的调用，假设我们不断地在子程序里调用子程序(当然，上面调用的层次还很浅)，我们知道，这样很快就会将调用栈耗尽,抛出`Segmental Fault`。

而一般，我们使用`ucontext`族函数，就不会出现这种情况。

```c
#include <ucontext.h>
#include <stdio.h>

#define MAX_COUNT (1<<30)

static ucontext_t uc[3];
static int count = 0;

void ping();
void pong();

void ping(){
    while(count < MAX_COUNT){
        printf("ping %d\n", ++count);
        // yield to pong
        swapcontext(&uc[1], &uc[2]); // 保存当前context于uc[1],切换至uc[2]的context运行
    }
}

void pong(){
    while(count < MAX_COUNT){
        printf("pong %d\n", ++count);
        // yield to ping
        swapcontext(&uc[2], &uc[1]);// 保存当前context于uc[2],切换至uc[1]的context运行
    }
}

char st1[8192];
char st2[8192];

int main(int argc, char *argv[]){
   

    // initialize context
    getcontext(&uc[1]);
    getcontext(&uc[2]);

    uc[1].uc_link = &uc[0]; // 这个玩意表示uc[1]运行完成后，会跳至uc[0]指向的context继续运行
    uc[1].uc_stack.ss_sp = st1; // 设置新的堆栈
    uc[1].uc_stack.ss_size = sizeof st1;
    makecontext (&uc[1], ping, 0);

    uc[2].uc_link = &uc[0]; // 这个玩意表示uc[2]运行完成后，会跳至uc[0]指向的context继续运行
    uc[2].uc_stack.ss_sp = st2; // 设置新的堆栈
    uc[2].uc_stack.ss_size = sizeof st2;
    makecontext (&uc[2], pong, 0);

    // start ping-pong
    swapcontext(&uc[0], &uc[1]); // 将当前context信息保存至uc[0],跳转至uc[1]保存的context去执行
  // 这里我稍微要多说几句，因为我迷惑过，我曾经困惑的一点在于uc[0]，为什么uc[0]不需要设置堆栈的信息？因为swapcontext已经帮我们做好了一切，swapcontext函数会将当前点的信息保存在uc[0]中，当然我们没有设置的话，默认的堆栈一定是主堆栈啦

    return 0;
}
```

我们现在应该可以了解设置`uc_stack`的缘由了，因为跳转至`uc[1]`或者`uc[2]`的`context`继续运行时的数据会保存在我们所指定的堆栈中，并不会占用原来堆栈的空间，所以不会出现主堆栈一般不会出现溢出的情况。

当然，还有`setcontext`函数。

```c
int setcontext(const ucontext_t *ucp);
```

这个函数其实很简单啦。那就是到`ucp`指向的那个`context`去执行。

借用一个很经典的例子：

```c
#include <stdio.h>
#include <ucontext.h> 
#include <unistd.h> 
int main(int argc, char *argv[]) 
{ 
  ucontext_t context; 
  getcontext(&context); 
  puts("Hello world"); 
  sleep(1); 
  setcontext(&context); 
  return 0; 
}
```

这个函数会不断地打印`Hello,world`。原因也很简单，因为上面的`getcontext`函数将那个点的上下文信息保存到了`context`中，下面调用`setcontext`会返回到记录的点处继续执行，因此也就出现了不断地输出。

利用这组函数，我们可以实现一个`coroutine`，感兴趣的，可以看一下我添加了注释的`coroutine`库：

[coroutine](https://github.com/Yihulee/SRC/tree/master/coroutine)



