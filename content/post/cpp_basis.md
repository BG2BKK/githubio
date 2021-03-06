+++
date = "2016-08-23T19:52:25+08:00"
draft = false 
title = "C++基础知识学习"

+++

lamba学习
---------------

1. [直观认识](http://www.lellansin.com/c-lambda表达式基本用法.html)
2. [cppreference](http://en.cppreference.com/w/cpp/language/lambda)
3. [微软cpp](https://msdn.microsoft.com/zh-cn/library/dd293608.aspx)
4. [function bind 和lambda的关系](http://blog.csdn.net/crayondeng/article/details/9996625)


static和const关键字各自的作用
--------------------------------


* static关键字
	* 在函数体内声明的话，static变量只分配一次，作用域范围也在该函数体内；当下次调用该函数时，该值保持不变
	* 模块内的static变量的作用域是模块内
	* 类的static变量属于整个类，而不是某个对象，所有对象都共有这一份拷贝

* const关键字
	* 定义常量；首次定义必须初始化，之后不能赋值
	* 函数声明时，用const修饰形参，可以保证不被函数体修改
	* 可以修饰类的成员函数，保证其返回值不为“左值”

拷贝构造函数和赋值构造函数
-----------------------------

C++的拷贝构造函数、重载赋值构造函数，以及析构函数，属于C++赋值控制的范畴

如果没有手动实现，编译器会自动生成一个；编译器会自动生成以下四个成员函数

* 1. 构造函数
* 2. 析构函数
* 3. 拷贝构造函数
* 4. 赋值构造函数

```cpp

class String {
public:
	String(const char *);			// 构造函数
	String(const String &other);	// 拷贝构造函数
	String & operator=( const String &other); // 赋值构造函数
	~String();						// 析构函数
private:
	char *m_data;
}
```

如果有手动实现，则会替代编译器的行为；除了析构函数，析构函数用于完成对象的释放操作，即使我们手动实现，编译器也会实现一份，这时析构函数可以让我们用来释放动态分配的内存.

* 拷贝构造函数

```cpp
String::String(const String &other) {
	int len = strlen(other.m_data);
	m_data = new char[len + 1];
	strcpy(m_data, other.m_data);
}
```

	* 如下几种情况下，拷贝构造函数被调用:
		* 1. 定义新对象，并用已有对象初始化新对象：即 String obj = other，或者 String obj(other)时，此时String(const String &other)被调用
		* 2. 对象作为参数传递时，函数将建立对象的临时拷贝
		* 3. 对象作为函数的返回值时，函数建立临时拷贝，并将其返回

* 赋值构造函数

```cpp
String & operator=(const String &other) {
	if( this == &other)
		return *this;
	delete []m_data;

	int len = strlen(other.m_data);
	m_data = new char[len + 1];
	strcpy(m_data, other.m_data);

	return *this;
}
```
	* 赋值构造函数的用法

```cpp
String obj;
obj = other;

```

	* 拷贝构造函数

```cpp
String obj = other;   //或者
String obj(other);
```

而在前者的***obj = other*** 和后者的***String obj=other***不同，前者表示obj是一个未初始化的对象，通过***=***进行赋值，后者中的***=***是使用other对obj进行初始化。

在赋值构造函数中，***=***缺省操作是将成员变量的值赋值，这时函数成员的旧值自然被丢弃，比如指针被赋予新值，旧值丢弃；然而指针旧值指向的内存却并未释放；因此包含动态分配成员的类提供拷贝构造函数外，还应该考虑重载***=***赋值操作符


抽象类与纯虚函数
----------------------

纯虚函数是在基类中声明的虚函数，在基类中没有定义，声明方法是在函数原型后加***=0***，例如

```cpp
virtual void function() = 0;
```

纯虚函数要求基类的所有派生类都要定义自己的实现方法，除非派生类也是抽象类，因此纯虚函数主要功能是实现接口声明。至少含有一个纯虚函数的类被称为抽象类。


基类的虚析构函数
-----------------------

在多态使用时，如果new一个子类对象，而用父类类型指针指向它来使用，delete父类类型指针的时候，只释放父类类型的资源，子类的虚构函数并不存在，导致资源不能及时释放。因此将父类的析构函数设置为虚函数，释放派生类对象时，链式调用析构函数，派生类析构函数先调用，父类的析构函数后调用。[参考链接](http://blog.csdn.net/starlee/article/details/619827)

一句话，基类的析构函数设置为虚，可以在释放时链式释放子类对象内存，防止内存泄露。不过这个机制，还是无法释放类中的动态分配的内存，所以良好的编程习惯是不挖坑的基础。


