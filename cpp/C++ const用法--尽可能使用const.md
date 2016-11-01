# C++ const用法--尽可能使用const



C++ `const`允许指定一个语义约束，编译器会强制实施这个约束，允许程序员告诉编译器某值是保持不变的。如果在编程中确实有某个值保持不变，就应该明确使用`const`，这样可以获得编译器的帮助。

1.   **`const`修饰成员变量**

     ```cpp
     #include<iostream>
     using namespace std;
     int main()
     {
         int a1=3;   ///non-const data
      
       ///可以这么来看待const int a2 = a1,那就是const修饰的是整个式子,也就是int a2,代表值

       const int a2=a1;    ///const data

       int * a3 = &a1;   ///non-const data,non-const pointer
       ///const int *a4中,const修饰的是整个式子,也就是int *a4,也就是说这个式子代表的东西是const
       const int * a4 = &a1;   ///const data,non-const pointer
       ///int * const a5中,const修饰的是a5,代表a5这个指针是const
       int * const a5 = &a1;   ///non-const data,const pointer
       ///加了两个const的玩意自然不必多说什么
       int const * const a6 = &a1;   ///const data,const pointer
       const int * const a7 = &a1;   ///const data,const pointer

       return 0;
     }
     ```


   `const`修饰指针变量时：

   　　(1)只有一个`const`，如果`const`位于`*`左侧，表示指针所指数据是常量，不能通过解引用修改该数据；指针本身是变量，可以指向其他的内存单元。

   　　(2)只有一个`const`，如果`const`位于`*`右侧，表示指针本身是常量，不能指向其他内存地址；指针所指的数据可以通过解引用修改。

   　　(3)两个`const`，`*`左右各一个，表示指针和指针所指数据都不能修改。

2. **`const`修饰函数参数**

   传递过来的参数在函数内不可以改变，与上面修饰变量时的性质一样。

   ```cpp
   void testModifyConst(const int _x) {
        _x=5;　　　///编译出错
   }
   ```

3. **`const`修饰成员函数**

   (1)`const`修饰的成员函数不能修改任何的成员变量(**mutable修饰的变量除外**)

   (2)`const`成员函数不能调用非`const`成员函数，因为非`const`成员函数可以会修改成员变量

   ```cpp
   #include <iostream>
   using namespace std;
   class Point{
       public :
       Point(int _x):x(_x){}

       void testConstFunction(int _x) const{ ///这里的const修饰的是整个函数

           ///错误，在const成员函数中，不能修改任何类成员变量
           x=_x;

           ///错误，const成员函数不能调用非onst成员函数，因为非const成员函数可以会修改成员变量
           modify_x(_x);
       }

       void modify_x(int _x){
           x=_x;
       }

       int x;
   };
   ```

4.**`const`修饰函数返回值**

(1)指针传递

如果返回`const data`,`non-const pointer`，返回值也必须赋给`const data`,`non-const pointer`。因为指针指向的数据是常量不能修改。

```cpp
const int * mallocA(){  ///const data,non-const pointer
    int *a=new int(2);
    return a; ///const int *表示的是常量的数据,但是非常量的指针
}

int main()
{
    const int *a = mallocA();
    ///int *b = mallocA();  ///编译错误
    return 0;
}
```

(2)值传递

如果函数返回值采用“值传递方式”，由于函数会把返回值复制到外部临时的存储单元中，加`const`修饰没有任何价值。所以，**对于值传递来说，加const没有太多意义。**

所以：

　　不要把函数`int GetInt(void)` 写成`const int GetInt(void)`。
　　不要把函数`A GetA(void)` 写成`const A GetA(void)`，其中`A`为用户自定义的数据类型。

**在编程中要尽可能多的使用const，这样可以获得编译器的帮助，以便写出健壮性的代码。**





