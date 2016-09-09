文章来自这里：[gcc内联汇编......](http://blog.csdn.net/slvher/article/details/8864996)

在阅读Linux内核源码或对代码做性能优化时，经常会有在C语言中嵌入一段汇编代码的需求，这种嵌入汇编在CS术语上叫做`inline assembly`。本文的笔记试图说明`Inline Assembly`的基本语法规则和用法（建议英文阅读能力较强的同学直接阅读本文参考资料中推荐的技术文章` ^_^`）。
注意：由于`gcc`采用AT&T风格的汇编语法（与Intel Syntax相对应，二者的区别参见这里），因此，本文涉及到的汇编代码均以AT&T Syntax为准。

##1. 基本语法规则
内联汇编（或称嵌入汇编）的基本语法模板比较简单，如下所示（为使结构更清晰，这里特意做了换行，其实完全可以全部写到单行中）：
```c
 asm [ volatile ] (  
         assembler template
         [ : output operands ]                /* optional */
         [ : input operands  ]                /* optional */
         [ : list of clobbered registers ]    /* optional */
         );
```
备注：本文遵从linux系统的统一风格，以`[ ]`来表示其对应的内容为可选项。
由代码模板可以看到，基本语法规则由5部分组成，下面分别进行说明。

**1）**关键字`asm`和`volatile`
`asm`为`gcc`关键字，表示接下来要嵌入汇编代码。为避免`keyword asm`与程序中其它部分产生命名冲突，`gcc`还支持`__asm__`关键字，与`asm`的作用等价。


`volatile`为可选关键字，表示不需要`gcc`对下面的汇编代码做任何优化。同样出于避免命名冲突的原因，`__volatile__`也是`gcc`支持的与`volatile`等效的关键字。


`BTW`: C语言中也经常用到volatile关键字来修饰变量（不熟悉的同学，请参考[这里](http://en.wikipedia.org/wiki/Volatile_variable)）

**2）**`assembler template`
这部分即我们要嵌入的汇编命令，由于我们是在`C`语言中内联汇编代码，故需用双引号`""`将命令括起来，以便`gcc`以字符串形式将这些命令传给汇编器`AS`。例如可以写成这样：`movl %eax, %ebx`。


有时候，汇编命令可能有多个，则通常分多行写，每行的命令都用双引号括起来，命令后紧跟`\n\t`之类的分隔符（当然，也可以只用1对双引号将多行命令括起来，从语法来说，两种写法均有效，我们可自行决定用哪种格式来写）。示例代码如下所示：

```c
__asm__ __volatile__ ( "movl %eax, %ebx\n\t"  
                       "movl %ecx, 2(%edx, %ebx, $8)\n\t"  
                       "movb %ah, (%ebx)"  
                     );  
```

 还有时候，根据程序上下文，嵌入的汇编代码中可能会出现一些类似于魔数（[Magic Number](http://en.wikipedia.org/wiki/Magic_number_(programming)) ）的操作数，比如下面的代码：

```c
int a=10, b;  
asm ("movl %1, %%eax;   /* NOTICE: 下面会说明此处用%%eax引用寄存器eax的原因 
      movl %%eax, %0;" 
      :"=r"(b)          /* output 该字段的语法后面会详细说明，此处可无视，下同 */  
      :"r"(a)           /* input   */  
      :"%eax"           /* clobbered register */  
    );     
```

我们看到，`movl`指令的操作数（operand）中，出现了`%1`、`%0`，这往往让新手摸不着头脑。其实只要知道下面的规则就不会产生疑惑了：
在内联汇编中，操作数通常用数字来引用，具体的编号规则为：**若命令共涉及n个操作数，则第1个输出操作数（the first output operand）被编号为0，第2个output operand编号为1，依次类推，最后1个输入操作数（the last input operand）则被编号为n-1。**

具体到上面的示例代码中，根据上下文，涉及到`2`个操作数变量`a`、`b`，这段汇编代码的作用是将`a`的值赋给`b`，可见，`a`是`input operand`，而`b`是`output operand`，那么根据操作数的引用规则，不难推出，`a`应该用`%1`来引用，`b`应该用`%0`来引用。

还需要说明的是：当命令中同时出现寄存器和以`%num`来引用的操作数时，会以`%%reg`来引用寄存器（如上例中的`%%eax`），以便帮助`gcc`来区分寄存器和由`C`语言提供的操作数。  

**3）**`output operands`
该字段为可选项，用以指明输出操作数，典型的格式为：
```c
`: "=a" (out_var)`
```
其中，`"=a"`指定`output operand`的应遵守的约束（constraint），`out_var`为存放指令结果的变量，通常是个`C`语言变量。本例中，`“=”`是`output operand`字段特有的约束，表示该操作数是只写的（write-only）；`“a”`表示先将命令执行结果输出至`%eax`，然后再由寄存器`%eax`更新位于内存中的`out_var`。关于常用的约束规则，本文后面会给出说明。

若输出有多个，则典型格式示例如下：
```c
asm ( "cpuid"  
      : "=a" (out_var1), "=b" (out_var2), "=c" (out_var3)  
      : "a" (op)  
     );  
```
可见，我们可以为每个output operand指定其约束。

**4）**`input operands`
该字段为可选项，用以指明输入操作数，其典型格式为：
               `: "constraints" (in_var)`
其中，`constraints`可以是`gcc`支持的各种约束方式，`in_var`通常为C语言提供的输入变量。

与`output operands`类似，当有多个`input`时，典型格式为：
```c
 : "constraints1" (in_var1), "constraints2" (in_var2), "constraints3" (in_var3), ...
```

当然，`input operands + output operands`的总数通常是有限制的，考虑到每种指令集体系结构对其涉及到的指令支持的最多操作数通常也有限制，此处的操作数限制也不难理解。此处具体的上限为`max(10, max_in_instruction)`，其中`max_in_instruction`为`ISA`中拥有最多操作数的那条指令包含的操作数数目。

需要明确的是，在指明`input operands`的情况下，即使指令不会产生`output operands`，其:也需要给出。例如`asm ("sidt %0\n" : :"m"(loc))`; 该指令即使没有具体的`output operands`也要将`:`写全，因为有后面跟着`: input operands`字段。

**5）**`list of clobbered registers` 
该字段为可选项，用于列出指令中涉及到的且没出现在`output operands`字段及`input operands`字段的那些寄存器。若寄存器被列入`clobber-list`，则等于是告诉`gcc`，这些寄存器可能会被内联汇编命令改写。因此，执行内联汇编的过程中，这些寄存器就不会被`gcc`分配给其它进程或命令使用。

##2. 常用约束（`commonly used constraints`）
前面介绍`output operands`和`input operands`字段过程中，我们已经知道这些operands通常需要指明各自的`constraints`，以便更明确地完成我们期望的功能（试想，如果不明确指定约束而由gcc自行决定的话，一旦代码执行结果不符合预期，调试将变得很困难）。

下面开始介绍一些常用的约束项。
**1）**寄存器操作数约束（`register operand`  constraint`,` r`）
当操作数被指定为这类约束时，表明汇编指令执行时，操作数被将存储在指定的通用寄存器（`General Purpose Registers`, `GPR`）中。例如： 
```c
asm ("movl %%eax, %0\n" : "=r"(out_val));
```
该指令的作用是将`%eax`的值返回给`%0`所引用的C语言变量`out_val`，根据`"=r`"约束可知具体的操作流程为：先将`%eax`值复制给任一`GPR`，最终由该寄存器将值写入`%0`所代表的变量中。`"r"`约束指明`gcc`可以先将`%eax`值存入任一可用的寄存器，然后由该寄存器负责更新内存变量。


通常还可以明确指定作为“中转”的寄存器，约束参数与寄存器的对应关系为：

```c
    a : %eax, %ax, %al
    b : %ebx, %bx, %bl
    c : %ecx, %cx, %cl
    d : %edx, %dx, %dl
    S : %esi, %si
    D : %edi, %di
```
例如，如果想指定用%ebx作为中转寄存器，则命令为：
```c
asm ("movl %%eax, %0\n" : "=b"(out_val));
```

**2）**内存操作数约束（`Memory operand constraint`,` m`）
当我们不想通过寄存器中转，而是直接操作内存时，可以用"m"来约束。例如：
```c
asm volatile ( "lock; decl %0" : "=m" (counter) : "m" (counter));
```
该指令实现原子减一操作，输入、输出操作数均直接来自内存（也正因如此，才能保证操作的原子性）。    

**3）**关联约束（`matching constraint`）
在有些情况下，如果命令的输入、输出均为同一个变量，则可以在内联汇编中指定以`matching constraint`方式分配寄存器，此时，`input operand`和`output operand`共用同一个“中转”寄存器。例如：
```c
asm ("incl %0" :"=a"(var):"0"(var));
```
该指令对变量`var`执行`incl`操作，由于输入、输出均为同一变量，因此可用`"0"`来指定都用`%eax`作为中转寄存器。注意`"0"`约束修饰的是`input operands`。

**4）**其它约束
除上面介绍的3中常用约束外，还有一些其它的约束参数（如`"o"`、`"V"`、`"i"`、`"g"`等），感兴趣的同学可以参考[这里](http://www.ibiblio.org/gferg/ldp/GCC-Inline-Assembly-HOWTO.html)。

##3. 实例剖析
前面介绍了很多理论性的规则，这里通过分析一个实例来加深对`inline assembly`的理解。
下面的代码是`Linux`内核`i386`版本中的`syscall0`定义：
```c
    #define _syscall0(type, name)          \
    type name(void)                        \
    {                                      \
        long __res;                        \
        __asm__ volatile ( "int $0x80"     \
          : "=a" (__res)                   \
          : "0" (__NR_##name));            \
        __syscall_return(type, __res);     \
    }
```
对于系统调用`fork`来说，上述宏展开为：
```c
    pid_t fork(void)
    {
        long __res;                       
        __asm__ volatile ( "int $0x80"    
        : "=a" (__res)                  
        : "0" (__NR_fork));           
        __syscall_return(pid_t, __res);    
    }
```
根据前面对`inline assembly`语法及使用方法的说明，我们不难理解这段代码的含义。将这段内联汇编翻译更可读的伪码形式为：
```c
pid_t fork(void)  
{  
    long __res;                         
    %eax = __NR_fork   /* __NR_fork为内核分配给系统调用fork的调用号 */  
    int $0x80          /* 触发中断，内核根据%eax的值可知，引起中断的是fork system call */  
    __res = %eax       /* 中断返回值保持在%eax中 */  
    __syscall_return(pid_t, __res);      
}  
```

【参考资料】
1. [GCC-Inline-Assembly-HOWTO](http://www.ibiblio.org/gferg/ldp/GCC-Inline-Assembly-HOWTO.html) 
2. [Inline assembly for x86 in Linux](http://www.ibm.com/developerworks/library/l-ia/index.html) 
3. 《程序员的自我修养—链接、装载与库》，第12章
4. [Using Assembly Language in Linux ](http://asm.sourceforge.net/articles/linasm.html)

