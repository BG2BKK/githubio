+++
date = '2016-10-19T11:18:04+08:00'
draft = true
title = 'cpp类型转换中C++11标准的不同'

+++

学习cpp内存布局和虚函数表相关的内容时，[图说C++对象模型：对象内存布局详解](http://www.cnblogs.com/QG-whz/p/4909359.html)这篇文章很不错，学习到如何通过对象地址找到虚函数表并调用其中的虚函数，然后我遇到了一个有意思的点，在C++11标准下编译不通过，但是在之前的C++标准可以编译通过。具体如下：

```cpp
// p为对象

cout << "虚函数表第一个函数的地址：" << (void *)*((int*)(&p)) << endl;
cout << "析构函数的地址:" << (void* )*(int *)*((int*)(&p)) << endl;
cout << "虚函数表中，第二个虚函数的地址：" << ((void *)*(int*)(&p) + 1) << endl;
```

```cpp
// std=c++11编译不通过
cout << "虚函数表第一个函数的地址：" << (void *)*((void *)(&p)) << endl;
```

[代码](https://github.com/BG2BKK/daily-programming/blob/master/cpp/class_memory.cpp)
