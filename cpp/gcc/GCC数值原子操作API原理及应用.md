文章来自[GCC数值原子操作API原理及应用](http://blog.zheezes.com/gcc-numerical-atomic-api-principles-and-applications.html)

## 一.前言

`C/C++`中数值操作，如自加`(n++)`自减`(n–-)`及赋值`(n=2)`操作都不是原子操作，如果是多线程程序需要使用全局计数器，程序就需要使用锁或者互斥量，对于较高并发的程序，会造成一定的性能瓶颈。

## **二**.gcc****原子操作****api

**1.**概要

为了提高赋值操作的效率，gcc提供了一组api，通过汇编级别的代码来保证赋值类操作的原子性，相对于涉及到操作系统系统调用和应用层同步的锁和互斥量，这组api的效率要高很多。

**2.**n++**类**

```c
type __sync_fetch_and_add(type *ptr, type value, ...); // m+n
type __sync_fetch_and_sub(type *ptr, type value, ...); // m-n
type __sync_fetch_and_or(type *ptr, type value, ...);  // m|n
type __sync_fetch_and_and(type *ptr, type value, ...); // m&n
type __sync_fetch_and_xor(type *ptr, type value, ...); // m^n
type __sync_fetch_and_nand(type *ptr, type value, ...); // (~m)&n
/* 对应的伪代码 */
{ tmp = *ptr; *ptr op= value; return tmp; }
{ tmp = *ptr; *ptr = (~tmp) & value; return tmp; }   // nand
```

**3.++n**类

```c
type __sync_add_and_fetch(type *ptr, type value, ...); // m+n
type __sync_sub_and_fetch(type *ptr, type value, ...); // m-n
type __sync_or_and_fetch(type *ptr, type value, ...); // m|n
type __sync_and_and_fetch(type *ptr, type value, ...); // m&n
type __sync_xor_and_fetch(type *ptr, type value, ...); // m^n
type __sync_nand_and_fetch(type *ptr, type value, ...); // (~m)&n
/* 对应的伪代码 */
{ *ptr op= value; return *ptr; }
{ *ptr = (~*ptr) & value; return *ptr; } // nand
```

**4.CAS**类

```c
bool __sync_bool_compare_and_swap (type *ptr, type oldval, type newval, ...);
type __sync_val_compare_and_swap (type *ptr, type oldval, type newval, ...);
/* 对应的伪代码 */
{ if (*ptr == oldval) { *ptr = newval; return true; } else { return false; } }
{ if (*ptr == oldval) { *ptr = newval; } return oldval; }
```

## **三**.**程序实例**

**1.test.c**

例子不是并发的程序，只是演示各`api`的使用参数和返回。由于是`gcc`内置`api`，所以并不需要任何头文件。

```c
#include <stdio.h>

int main() {
    int num = 0;

    /*
     * n++;
     * __sync_fetch_and_add(10, 3) = 10
     * num = 13
     */
    num = 10;
    printf("__sync_fetch_and_add(%d, %d) = %d\n", 10, 3, __sync_fetch_and_add(&num, 3));
    printf("num = %d\n", num);

    /*
     * ++n;
     * __sync_and_add_and_fetch(10, 3) = 13
     * num = 13
     */
    num = 10;
    printf("__sync_and_add_and_fetch(%d, %d) = %d\n", 10, 3, __sync_add_and_fetch(&num, 3));
    printf("num = %d\n", num);

    /*
     * CAS, match
     * __sync_val_compare_and_swap(10, 10, 2) = 10
     * num = 2
     */
    num = 10;
    printf("__sync_val_compare_and_swap(%d, %d, %d) = %d\n", 10, 10, 2, __sync_val_compare_and_swap(&num, 10, 2));
    printf("num = %d\n", num);

    /*
     * CAS, not match
     * __sync_val_compare_and_swap(10, 3, 5) = 10
     * num = 10
     */
    num = 10;
    printf("__sync_val_compare_and_swap(%d, %d, %d) = %d\n", 10, 3, 5, __sync_val_compare_and_swap(&num, 1, 2));
    printf("num = %d\n", num);

    return 0;
}
```

##  **四**.Reference

(1)[gcc Atomic-Builtins doc](http://gcc.gnu.org/onlinedocs/gcc-4.1.2/gcc/Atomic-Builtins.html)
(2)[CAS(Compare-and-swap)](http://en.wikipedia.org/wiki/Compare-and-swap)