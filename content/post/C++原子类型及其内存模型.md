+++
date = '2016-08-24T08:50:11+08:00'
draft = true
title = 'C++原子类型及其内存模型'

+++

C++的原子类型atomic types提供了多种数据类型的原子类型，在[学习《C++并发编程》时](https://chenxiaowei.gitbooks.io/cpp_concurrency_in_action/content/content/chapter5/chapter5-chinese.html)，我知道C++的互斥锁基于原子类型std::atomic_flag实现，而原子类型是基于其内存访问模型的。

atomic_flag是最简单的原子模型，表示一个布尔标志，只有两种状态：设置和清除；该类型的对象必须被ATOMIC_FLAG_INIT初始化，初始化状态是“清除”；使用该类型实现自旋互斥锁非常简单：

```cpp
class spinlock_mutex
{
  std::atomic_flag flag;
public:
  spinlock_mutex():
    flag(ATOMIC_FLAG_INIT)
  {}
  void lock()
  {
    while(flag.test_and_set(std::memory_order_acquire));
  }
  void unlock()
  {
    flag.clear(std::memory_order_release);
  }
};
```
