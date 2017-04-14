#`const`在函数前与函数后的区别
转载自[http://blog.csdn.net/clozxy/article/details/5679887](http://blog.csdn.net/clozxy/article/details/5679887)
## 一   `const`基础 

如果`const`关键字不涉及到指针，我们很好理解，下面是涉及到指针的情况：
```c++
int b =  500;   
const int* a = &b;         		// [1]   
int const *a = &b;              // [2]   
int* const a = &b;              // [3]   
const int* const a = &b;	   	// [4]   
```
如果你能区分出上述四种情况，那么，恭喜你，你已经迈出了可喜的一步。不知道，也没关系，我们可以参考`《effective c++》` `item21`上的做法，如果`const`位于星号的左侧，则`const`就是用来修饰指针所指向的变量，即**指针指向为常量**；如果`const`位于星号的 右侧，`const`就是修饰指针本身，即**指针本身是常量**。因此，[1]和[2]的情况相同，都是指针所指向的内容为常量，这种情况下不允许对内容进行更改操 作，如不能`*a = 3`   ；[3]为指针本身是常量，而指针所指向的内容不是常量，这种情况下不能对指针本身进行更改操作，如`a++`是错误的；[4]为指针本身和指向的内容均为常量。 

另外`const `的一些强大的功能在于它在函数声明中的应用。在一个函数声明中，`const `可以修饰函数的返回值，或某个参数；对于成员函数，还可以修饰是整个函数。有如下几种情况，以下会逐渐的说明用法：
```c++
A& operator=(const A& a);   
void fun0(const A* a);   
void fun1() const;   // fun1()为类成员函数   
const A fun2();   
```
#二   `const`的初始化 
先看一下`const`变量初始化的情况
1)   非指针`const`常量初始化的情况:
```c++
A b;
const A a = b;   
```

2)   指针(引用)`const`常量初始化的情况：
```c++
A* d = new A(); 
const A* c = d; // c指向一个常量,也就是说,通过c不能对其指向的对象做任何修改.
```
或者：```const A* c = new A();```   

引用：

```c++
A f;
const A& e = f; // 这样作e只能访问声明为const的函数，而不能访问一般的成员函数； 
```
[思考1]：   以下的这种赋值方法正确吗？ 
```c++
const A* c = new A();   
A* e = c;   
```
[思考2]：   以下的这种赋值方法正确吗？
```c++
A* const c = new A();   
A* b = c;
```
#三   作为参数和返回值的`const`修饰符 
其实，不论是参数还是返回值，道理都是一样的，参数传入时候和函数返回的时候，初始化`const`变量   

1. 修饰参数的`const`，如 
```
void   fun0(const A*  a);
void   fun1(const A&  a);
```
调用函数的时候，用相应的变量初始化`const`常量，则在函数体中，按照`const`所修饰的部分进行常量化，如形参为`const A* a`，则不能对传递进来的指针的内容进行改变，保护了原指针所指向的内容；如形参为`const A& a`，则不能对传递进来的引用对象进行改变，保护了原对象的属性。 
[注意]：参数`const`通常用于参数为指针或引用的情况; 

2. 修饰返回值的`const`，如
```c++
const A fun2();
const A* fun3();
```
这样声明了返回值后，`const`按照"修饰原则"进行修饰，起到相应的保护作用。
```c++
const rational operator*(const rational& lhs,const rational& rhs)   
{   
  return rational(lhs.numerator() * rhs.numerator(), lhs.denominator() * rhs.denominator());   
}   
```
返回值用`const`修饰可以防止这样的操作发生:
```c++
rational a,b;   
radional c;   
(a*b) = c;
```
一般用`const`修饰返回值为对象本身的情况多用于二目操作符重载函数并产生新对象的时候。   

[总结]   一般情况下，函数的返回值为某个对象时，如果将其声明为`const`时，多用于操作符的重载。通常，不建议用`const`修饰函数的返回值类型为某个对象或对 某个对象引用的情况。  

原因如下：  

如果返回值为某个对象为`const`或某个对象的引用为`const` ,则返回值具有`const`属性，则返回实例只能访问类`a`中的公有数据成员和`const`成员函数，并且不允许对其进行赋值操作，这在一般情况下很少用 到。 

[思考3]：   这样定义赋值操作符重载函数可以吗？   
```c++
const A& operator=(const A& a);   
```

# 四 类成员函数中`const`的使用 
一般放在函数体后，形如：
```
void fun() const;
```
如果一个成员函数的不会修改数据成员，那么最好将其声明为`const`，因为`const`成员函数中不允许对数据成员进行修改，如果修改，编译器将报错，这大 大提高了程序的健壮性。

#五 使用`const`的一些建议
1. 要大胆的使用`const`，这将给你带来无尽的益处，但前提是你必须搞清楚原委；   
2. 要避免最一般的赋值操作错误，如将`const`变量赋值，具体可见思考题；   
3. 在参数中使用`const`应该使用引用或指针，而不是一般的对象实例，原因同上；   
4. `const`在成员函数中的三种用法要很好的使用；   
5. 不要轻易的将函数的返回值类型定为`const`;   
6. 除了重载操作符外一般不要将返回值类型定为对某个对象的`const`引用;  

# 六 一个简单的例子
```c++
#include <iostream>
using namespace std;

class A {
public:
	int a;
	A() : a(0)
	{}

	//
	// add 对成员变量进行了修改,所以该函数不能够被const修饰.
	//
	void add() {
		a++;
	}

	//
	// doNoting 不被const修饰的函数,在这个函数里,可以对成员变量进行修改,因此const A的实体不能调用这个函数.
	//
	void doNothing() {

	}

	//
	// doNotingConst 被const修饰的函数,也就是说在这个函数里面不能对成员变量有任何修改.
	// 否则的话,编译是通不过的,这个函数可以别一般的A实体,或者const A的实体调用.
	//
	void doNothingConst() const {

	}
};

int main() {
	//
	// 第一组,a是一个const A的实体,b是一个A的实体.
	//
	A b;
	const A a = b;
	//a.doNothing(); // error.
	
	//
	// 第二组,d是一个A的指针, c是一个const A的指针.
	//
	A* d = new A();
	const A* c = d; // c指向一个常量,具体而言,就是说通过c,不能够对其指向的对象做任何的修改.
	b.doNothing(); // ok.
	b.doNothingConst(); // ok.
	cout << c->a << endl; // ok.
	//c->add(); // error
	//c->doNothing(); // error
	c->doNothingConst(); // right,也就是说,一个const对象仅能够调用那些不改变A对象实体的函数,也就是带const版本的函数

	//
	// 第三组,f是一个A对象的实体,e是const A的引用.
	//
	A f;
	const A& e = f;
	//e.doNothing(); // error

	//
	// 第四组,g是一个const A对象的指针,h是一个A对象的指针.
	//
	const A* g = new A();
	//A* h = g; // error,不要妄想通过h对g所指向的const A实体进行修改.

	//
	// 第五组,i是一个A对象的指针常量, j是一个A对象的指针.
	//
	A* const i = new A();
	A* j = i; // ok

	getchar();
	return 0;
}
```