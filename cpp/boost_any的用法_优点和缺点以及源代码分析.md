博文来自[boost::any的用法、优点和缺点以及源代码分析](http://blog.csdn.net/hityct1/article/details/4186962),做了一部分补充。

## `boost::any`用法示例:

```c++
#include <iostream>   
#include <list>   
#include <boost/any.hpp>     
typedef std::list<boost::any> list_any;   
 
//关键部分：可以存放任意类型的对象   
void fill_list(list_any& la)   
{       
	la.push_back(10);//存放常数    
	la.push_back(std::string("dyunze"));//存放字符串对象；注意la.push_back(“dyunze”)错误，因为会被当错字符串数组   
}   

//根据类型进行显示   
void show_list(list_any& la)   
{   
	list_any::iterator it;   
	boost::any anyone;   
 
	for (it = la.begin(); it != la.end(); it++)   
	{      
		anyone = *it;   
 
		if (anyone.type() == typeid(int))   
			std::cout<<boost::any_cast<int>(*it) << std::endl;   
		else if (anyone.type() == typeid(std::string))   
			std::cout<<boost::any_cast<std::string>(*it).c_str() << std::endl;   

	}   

}   
  
int main()   
{   
	list_any la;   
	fill_list(la);   
	show_list(la);     
	return 0;   
	
}  
```

我们来看一下输出的结果：

```c++
10
dyunze
```

## **`boost::any`的优点：**

对设计模式理解的朋友都会知道合成模式。因为多态只有在使用指针或引用的情况下才能显现，所以`std`容器中只能存放指针或引用（但实际上只能存放指针，无法存放引用，这个好像是`c++`的不足吧）。如：

```c++
std::list<BaseClass*> mylist;
```

这样，我们就要对指针所指向内容的生存周期操心(可能需要程序员适时删除申请的内存；但是由于存放指针，插入/删除的效率高)，而使用`boost::any`就可能避免这种情况，因为我们可以存放类型本身（当然存放指针也可以）。这是`boost::any`的优点之一。

`boost::any`另一个优点是可以存放任何类型。而前面提到的`mylist`只能存放`BaseClass`类指针以及其继承类的指针。

 

## **`boost::any`的缺点：**

由于`boost::any`可以存放任何类型，自然它用不了多态特性，没有统一的接口，所以在获取容器中的元素时需要实现判别元素的真正类型，这增加了程序员的负担。与面向对象编程思想有些矛盾，但整个标准`c++`模板库何尝不是如此，用那些牛人的话来说，是“有益补充”。

总之，有利必有弊，没有十全十美的。

   

## **分析并模仿`boost::any`：**

读了一下`boost::any`的源代码，并模仿一下其实现(相当一部分时拷贝原代码)，下面是代码(只包含必要功能)。

实现`any`的功能主要由三部分组成：

实现`any`的功能主要由三部分组成：
**1）**`any`类

**2）**真正保存数据的`holder`类及其基类`placeholder`
**3）**获取真正数据的模板函数`any_cast`，类型转换的功能

```c++
#include <iostream>
#include <list>
#include <cassert>
#include <typeinfo>

//自定义的any类
class any
{
public:
	
	//保存真正数据的接口类
	class placeholder // 这里算是一个内嵌类
	{
	public:     
		virtual ~placeholder()
		{
		}
	public: 

		virtual const std::type_info & type() const = 0; // 类型信息
		virtual placeholder * clone() const = 0;    
	};

	//真正保存和获取数据的类。
	template<typename ValueType>
		class holder : public placeholder
		{
		public:         
			holder(const ValueType & value) // 能够根据传入的参数推断出来ValueType
				: held(value)
			{
			}

		public: 

			virtual const std::type_info & type() const
			{
				return typeid(ValueType); // 好吧,typeid返回的是type_info对象的一个引用
			}

			virtual placeholder * clone() const
			{
				return new holder(held);//使用了原型模式
			}

		public: 

				//真正的数据，就保存在这里
			ValueType held; // 好吧，其实还是会复制一份数据的！
		};

public:

	any()
		: content(NULL)   
	{       
	}

	//模板构造函数，参数可以是任意类型，真正的数据保存在content中
	template<typename ValueType>
		any(const ValueType & value) // 传入的其实是任意对象的一个引用，是吧！
			: content(new holder<ValueType>(value))
		{
		}  

	//拷贝构造函数
	any(const any & other)
		: content(other.content ? other.content->clone() : 0)
	{
	}

	//析构函数，删除保存数据的content对象
	~any()
	{
		if (NULL != content)
			delete content;
	}
	
	// 重载了赋值运算符
	any& operator =(const any & other)
	{

		if (this == &other)
		{
			return *this;
		}

		if (NULL != content) // content是一个指针
		{
			delete content; 
		}

		content = other.content ? other.content->clone() : 0;

		return *this;
	}

private:
	//一个placeholde对象指针，指向其子类folder的一个实现
	// 即content( new holder<ValueType>(value) )语句
	placeholder* content;

	template<typename ValueType> friend ValueType any_cast(const any& operand); // 居然存在一个友元函数
	template<typename ValueType> friend ValueType * any_case(const any* operand);
public: 

	//查询真实数据的类型。
	const std::type_info & type() const
	{
		return content ? content->type() : typeid(void);
	}
};


//获取content->helder数据的方法。用来获取真正的数据
template<typename ValueType>
	ValueType any_cast(const any& operand)
	{
		assert(operand.type() == typeid(ValueType)); // 好吧，这里首先要判断两者类型信息是否一致
		return static_cast<any::holder<ValueType> *>(operand.content)->held; // held是真正的持有数据的变量
		// 返回的值很值得回味,operand.content确实是一个any::holder<ValueType>类型的指针
	}

template<typename ValueType>
	ValueType* any_case(const any* operand) // 这个函数十分重要
	{
		// 如果传入any类型的指针，那就传回ValueType类型的指针
		assert(operand->type() == typeid(ValueType)); // 判断类型是否一致
		return &static_cast<any::holder<ValueType> *>(operand->content)->held; // 返回指针
	}

	//下代码是使用示例

typedef std::list<any> list_any;

void fill_list(list_any& la)
{    
	la.push_back(10);//存放常数；调用了any的模板构造函数，下同
	la.push_back(std::string("I am string"));//存放字符串对象；注意la.push_back(“dyunze”)错误，因为会被当错字符串数组

	char* p = "I am const char* abc";
	la.push_back(p);//可以存放指针，但要注意指针的失效问题
}

//根据类型进行显示
void show_list(list_any& la)
{
	list_any::iterator it;

	for (it = la.begin(); it != la.end(); it++)
	{	

		if ((*it).type() == typeid(int)) // 这个比较就很爽啦！RIIT果然是一件大块人心的好事
			std::cout<<any_cast<int>(*it) << std::endl;
		else if ((*it).type() == typeid(std::string))
			std::cout<<any_cast<std::string>(*it).c_str() << std::endl;
		else if ((*it).type() == typeid(char*))
			std::cout<<any_cast<char*>(*it) << std::endl;
	}
}

int main()
{
	list_any la;
	fill_list(la);
	show_list(la);
	return 0;
}
```

运行结果如下：

```c++
10
I am string
I am const char* abc
```

