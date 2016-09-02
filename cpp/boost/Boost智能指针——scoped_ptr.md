#Boost智能指针——scoped_ptr
`boost::scoped_ptr`和`std::auto_ptr`非常类似，是一个简单的智能指针，它能够保证在离开作用域后对象被自动释放。下列代码演示了该指针的基本应用：
```c++
#include <string>
#include <iostream>
#include <boost/scoped_ptr.hpp>

class implementation
{
public:
    ~implementation() { std::cout <<"destroying implementation\n"; }
    void do_something() { std::cout << "did something\n"; }
};

void test()
{
    boost::scoped_ptr<implementation> impl(new implementation());
    impl->do_something();
}

void main()
{
    std::cout<<"Test Begin ... \n";
    test();
    std::cout<<"Test End.\n";
}
```
输出的结果是：

```c++
Test Begin ...
did something
destroying implementation
Test End.

```

可以看到：当`implementation`类离其开`impl`作用域的时候，会被自动删除，这样就会避免由于忘记手动调用`delete`而造成内存泄漏了。

## **`boost::scoped_ptr`的特点**

`boost::scoped_ptr`的实现和`std::auto_ptr`非常类似，都是利用了一个栈上的对象去管理一个堆上的对象，从而使得堆上的对象随着栈上的对象销毁时自动删除。不同的是，`boost::scoped_ptr`有着更严格的使用限制——不能拷贝。这就意味着：**`boost::scoped_ptr`指针是不能转换其所有权的。**

1. 不能转换所有权。`boost::scoped_ptr`所管理的对象生命周期仅仅局限于一个区间（该指针所在的"`{}`"之间），无法传到区间之外，这就意味着`boost::scoped_ptr`对象是不能作为函数的返回值的（`std::auto_ptr`可以）。

2. 不能共享所有权。这点和`std::auto_ptr`类似。这个特点一方面使得该指针简单易用。另一方面也造成了功能的薄弱——不能用于`stl`的容器中。

3. 不能用于管理数组对象。由于`boost::scoped_ptr`是通过`delete`来删除所管理对象的，而数组对象必须通过`deletep[]`来删除，因此`boost::scoped_ptr`是不能管理数组对象的，如果要管理数组对象需要使用`boost::scoped_array`类。

   ​

   ## **`boost::scoped_ptr`的常用操作**

   可以简化为如下形式：

   ```c++
   namespace boost {

       template<typename T> class scoped_ptr : noncopyable {
       public:
           explicit scoped_ptr(T* p = 0); 
           ~scoped_ptr(); 

           void reset(T* p = 0); 

           T& operator*() const; 
           T* operator->() const; 
           T* get() const; 

           void swap(scoped_ptr& b); 
       };

       template<typename T> 
       void swap(scoped_ptr<T> & a, scoped_ptr<T> & b); 
   }
   ```

   它的常用操作如下：

   |         成员函数          |              功能              |
   | :-------------------: | :--------------------------: |
   |     `operator*()`     |      以引用的形式访问所管理的对象的成员       |
   |    ` operator->()`    |      以指针的形式访问所管理的对象的成员       |
   |       `reset()`       |      释放所管理的对象，管理另外一个对象       |
   | `swap(scoped_ptr& b)` | 交换两个`boost::scoped_ptr`管理的对象 |
   |                       |                              |

   下列测试代码演示了这些功能函数的基本使用方法。

   ```c++
   #include <string>
   #include <iostream>

   #include <boost/scoped_ptr.hpp>
   #include <boost/scoped_array.hpp>

   #include <boost/config.hpp>
   #include <boost/detail/lightweight_test.hpp>

   void test()
   {
       // test scoped_ptr with a built-in type
       long * lp = new long;
       boost::scoped_ptr<long> sp ( lp );
       BOOST_TEST( sp.get() == lp );
       BOOST_TEST( lp == sp.get() );
       BOOST_TEST( &*sp == lp );

       *sp = 1234568901L;
       BOOST_TEST( *sp == 1234568901L );
       BOOST_TEST( *lp == 1234568901L );

       long * lp2 = new long;
       boost::scoped_ptr<long> sp2 ( lp2 );

       sp.swap(sp2);
       BOOST_TEST( sp.get() == lp2 );
       BOOST_TEST( sp2.get() == lp );

       sp.reset(NULL);
       BOOST_TEST( sp.get() == NULL );

   }

   void main()
   {
       test();
   }
   ```

   ## **`boost::scoped_ptr`和`std::auto_ptr`的选取：**

   `boost::scoped_ptr`和`std::auto_ptr`的功能和操作都非常类似，如何在他们之间选取取决于是否需要转移所管理的对象的所有权（如是否需要作为函数的返回值）。如果没有这个需要的话，大可以使用`boost::scoped_ptr`，让编译器来进行更严格的检查，来发现一些不正确的赋值操作。