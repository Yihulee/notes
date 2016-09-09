关于gcc内联代码约束的测试！

所谓一切都是虚妄，我们来看一看生成的代码吧！

```c
int foo = 20, bar = 15;
	int x = 10;
	__asm__ __volatile__("addl %0, %1"
		: "=c" (x) /* 好吧，我们来看一下吧，x对应0,然后foo对应1，bar对应2 */
		: "a"(foo), "b"(bar)
	);
```

对应的汇编如下:

```shell
 21         movl    $20, -28(%rbp) # foo
 22         movl    $15, -24(%rbp) # bar
 23         movl    $10, -20(%rbp) # x
 24         movl    -28(%rbp), %eax # foo -> eax
 25         movl    -24(%rbp), %edx # bar -> ebx
 26         movl    %edx, %ebx
 27 #APP
 28 # 8 "01.c" 1
 29         addl %ecx, %eax # result in eax
 30 # 0 "" 2
 31 #NO_APP
 32         movl    %ecx, %eax # ecx -> x
 33         movl    %eax, -20(%rbp) 
```

我稍微来说一下吧，我上面指定`foo`放入`eax`寄存器，`bar`放入`ebx`寄存器（虽然经过了中转，但是最终还是达成了）。然后对于输出的约定是，要将`ecx`的值放入`x`，我们也看得到，虽然经过了中转，但是最终也还是成功了。

好了，稍微来改变一下：

```c
int foo = 20, bar = 15;
	int x = 10;
	__asm__ __volatile__("addl %0, %1"
		: "=m" (x) /* 好吧，我们来看一下吧，x对应0,然后foo对应1，bar对应2 */
		: "a"(foo), "b"(bar)
	);
```

对应的汇编如下：

```shell
 24         movl    $20, -32(%rbp) # foo
 25         movl    $15, -28(%rbp) # bar
 26         movl    $10, -36(%rbp) # x
 27         movl    -32(%rbp), %eax # foo -> eax
 28         movl    -28(%rbp), %edx # bar -> ebx
 29         movl    %edx, %ebx
 30 #APP
 31 # 8 "01.c" 1
 32         addl -36(%rbp), %eax # 这里是直接从内存中取数
 33 # 0 "" 2
 34 #NO_APP
```

我们这里要看到`m`这个约束的作用，那就是，要求直接从内存中取数。

继续来改:

```c
int foo = 20, bar = 15;
	int x = 10;
	__asm__ __volatile__("addl %1, %2"
		: "=m" (x) /* 好吧，我们来看一下吧，x对应0,然后foo对应1，bar对应2 */
		: "a"(foo), "b"(bar)
	);
```

对应的汇编如下：

```shell
 24         movl    $20, -32(%rbp) # foo
 25         movl    $15, -28(%rbp) # bar
 26         movl    $10, -36(%rbp) # x
 27         movl    -32(%rbp), %eax # foo -> eax
 28         movl    -28(%rbp), %edx # bar -> ebx
 29         movl    %edx, %ebx
 30 #APP
 31 # 8 "01.c" 1
 32         addl %eax, %ebx # foo + bar
 33 # 0 "" 2
 34 #NO_APP
```

我这个例子无非就是想说明，如果我们用`m`约束的话，对应的变量的读取，存储会直接操作内存，而不会通过寄存器。

我们继续来看：

```c
int foo = 20, bar = 15;
	int x = 10;
	__asm__ __volatile__("addl %0, %1"
		: "=r" (x) /* 好吧，我们来看一下吧，x对应0,然后foo对应1，bar对应2 */
		: "a"(foo), "b"(bar)
	);
```

汇编如下：

```shell
 21         movl    $20, -28(%rbp) # foo
 22         movl    $15, -24(%rbp) # bar
 23         movl    $10, -20(%rbp) # x
 24         movl    -28(%rbp), %eax # foo -> eax
 25         movl    -24(%rbp), %edx # bar -> ebx
 26         movl    %edx, %ebx
 27 #APP
 28 # 8 "01.c" 1
 29         addl %eax, %eax
 30 # 0 "" 2
 31 #NO_APP
 32         movl    %eax, -20(%rbp)
```

好吧，看到了吧，如果你用`r`约束的话，`gcc`会帮你选择一个寄存器当做中介，这里是选择了`eax`，最终还要将`eax`的结果更新到`x`。

好吧，我们现在来看一下对应的数字吧，我们首先来审视一下规则：

>数字`%n`的用法：数字表示的寄存器是按照出现和从左到右的顺序映射到用`"r"`或`"q"`请求的寄存器．如果要重用`"r"`或`"q"`请求的寄存器的话，就可以使用它们。

我来举个例子：

```c
int foo = 20, bar = 15;
	int x = 10;
	int y = 20;
	__asm__ __volatile__("addl %0, %1"
		: "=r" (x) /* 好吧，我们来看一下吧，x对应0,然后foo对应1，bar对应2 */
		: "0"(foo), "1"(bar)
	);
```

这样的代码，编译是通不过的，为什么呢？我们看到，上面的汇编中，我们只使用了一个`r`约束，而下面的，我们使用了两个数字，因此，`bar`对应的那个数字映射不到寄存器，所以就会出错，这么来改：

```c
int foo = 20, bar = 15;
	int x = 10;
	int y = 20;
	__asm__ __volatile__("addl %0, %1"
		: "=r" (x), "=r"(y)/* 好吧，我们来看一下吧，x对应0,然后foo对应1，bar对应2 */
		: "0"(foo), "1"(bar)
	);
```

现在就可以了，我们来看一下对应的汇编代码吧：

```shell
 19         movl    $20, -16(%rbp) # foo
 20         movl    $15, -12(%rbp) # bar
 21         movl    $10, -8(%rbp) # x
 22         movl    $20, -4(%rbp) # y
 23         movl    -16(%rbp), %edx # foo -> edx
 24         movl    -12(%rbp), %eax # bar -> eax
 25 #APP
 26 # 9 "01.c" 1
 27         addl %edx, %eax # 
 28 # 0 "" 2
 29 #NO_APP
 30         movl    %edx, -8(%rbp) # edx -> x,这里x对应的寄存器是edx,恰好是复用了
 31         movl    %eax, -4(%rbp) # eax -> y,这里y对应的寄存器是eax,恰好是复用了
```

这也是下面这段经典代码的思想：

```c
int var = 10;
__asm__ __volatile__(
  "incl %0" 
  : "=a"(var) 
  : "0"(var));
```

对应的汇编代码如下：

```shell
 15         movl    $10, -4(%rbp) # var
 16         movl    -4(%rbp), %eax # var -> eax
 17 #APP
 18 # 15 "01.c" 1
 19         incl %eax
 20 # 0 "" 2
 21 #NO_APP
 22         movl    %eax, -4(%rbp) # eax -> var

```

我们继续修改：

```c
int var1 = 10;
int var2 = 20;
__asm__ __volatile__(
  "incl %0; \n"
  "incl %1; \n"
  : "=b"(var1), "=a"(var2)
  : "0"(var1), "1"(var2));
```

对应的汇编代码如下

```shell
 17         movl    $10, -16(%rbp) # var1
 18         movl    $20, -12(%rbp) # var2
 19         movl    -16(%rbp), %edx # var1 -> ebx
 20         movl    -12(%rbp), %eax # var2 -> eax
 21         movl    %edx, %ebx
 22 #APP
 23 # 16 "01.c" 1
 24         incl %ebx; 
 25 incl %eax; # 在这里我们看到了不写\t的后果了，那就是输出的汇编中格式不整齐
 26 
 27 # 0 "" 2
 28 #NO_APP
 29         movl    %ebx, %edx # ebx -> var1
 30         movl    %edx, -16(%rbp) 
 31         movl    %eax, -12(%rbp) # eax -> var2
```

我个人觉得，所谓的数字约束，出现在输入或者输出操作数中作为约束的数字不光是映射了`r`,`q`这些约束对应的寄存器，推而广之，是映射了所有的不是数字的约束对应的寄存器，从上面生成的代码中，你可以看到。当然，毕竟我没有测试，不敢乱说。



其实，我觉得我写得差不多了，**难啃的点，测试一下，就知道了**。