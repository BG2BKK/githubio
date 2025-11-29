+++
date = '2016-12-24T19:13:26+08:00'
draft = true
title = 'container_of和list的讲解'

+++


对linux kernel中的container_of总是理解了又忘记，这次还是一次搞明白吧。

在[参考文章](http://blog.csdn.net/yuzhihui_no1/article/details/38356393)中说的很明白，讲到问题的关键点。

```cpp
     #define list_entry(ptr, type, member) /
                     ((type *)((char *)(ptr)-(unsigned long)(&((type *)0)->member)))
```

对宏list_entry来说，type是结构体定义，member是结构体定义中的一个成员，ptr是一个结构体变量的member成员地址，list_entry的目的是，在已知某个结构体变量中某个成员地址的情况下，如何根据结构体类型type得出该结构体变量的地址


```cpp
typedef struct container {
    char dummy;
    int a;
    int b;
} container_t ;

int main() {

    container_t c;

    int *p = &(c.b);

    printf("offset of p: %u\n", &(((container_t *)0)->b));
    printf("address of p: %p\n", (container_t *)( (char *)p - (unsigned long)&(((container_t *)0)->b)) );
    printf("address of c: %p\n", &c );

}
```

在以上代码中，container_t是结构体类型type，c是结构体变量，指针p是c中成员b的地址；(container_t * )0 表示将地址0强制转换为container_t类型，***表示0地址处存放了container_t类型的结构体变量***，则 ((container_t * )0)->b指向b成员，那么它的地址 &(((container_t * )0)->b)不光是b的地址，由于结构体变量的地址是0，***那么这个地址减去结构体变量的地址就是b成员相对于结构体定义的偏移***，而对于另一个结构体变量c来说，用p减去这个偏移也能得到c的地址。

理解到&(((container_t * )0)->b)是地址也是偏移，就能抓住container_of的意思了。
