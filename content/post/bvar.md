## brpc的bvar性能采集库-学习笔记

[brpc](https://github.com/brpc/brpc)是百度开源的rpc框架，具有性能卓越、设计独到、实现优雅和协议丰富的特点

由于我做过性能采集和监控相关的工作，因此对bvar的实现非常感兴趣，通过精读代码，学习了bvar的实现细节，这里做个记录。个人水平有限，有错误的地方请指正。

[bvar](https://github.com/brpc/brpc/blob/master/docs/cn/bvar.md)是一个相对独立的组件，不依赖外部，你可以只使用bvar而不用将整个brpc库接入。bvar的接入使用也十分简单，官方文档也很全面，通过本文也是想让读者有个直观的认识，直接上手。

### bvar的基本特性

最简单来说，想象有一个累加器类型的bvar，它将有两个成员，一个是name，另一个是统计值，存储统计值的内存空间是从本类型的tls内存池申请而来；它还有一个方法，将每次的新值累加起来，bvar对象重载***operator<<***运算符，接受并聚合统计值，类似于“写”语义。

bvar支持线程安全的数值统计与采集，并且属于写远大于读的应用场景；它从tls内存池获取内存，聚合本线程的统计值；在每次被读取时再将统计数值聚合起来，相当于将写操作的同步开销转移到读操作，所以性能损耗少。

bvar中，**Variable**是基类，派生出**Reducer**、**IntRecorder**、**Status**等子类，**Window**也由**Variable**派生，他们共有行为主要有：

* **expose**：向进程全局VarMap注册自己，以expose的name作为索引。后台dump线程遍历VarMap读值
* **hide**：与expose相反，从VarMap注销
* **describe**：描述Variable的名字和值，以字符串返回

此外，**LatencyRecorder**将其它类型组合而成，功能十分强大（具有类似于FIO的长尾IO分布功能，99%、99.5%%、99.9%、99.99%，以及CDF，这个功能我觉得在运行时不好实现），本篇暂不包含它。

bvar类型与Variable可看作语义相同。

#### bvar的类型

* **Reducer**派生出**Adder**、**Maxer**、**Miner**等子类，用于简单的聚合、极大值极小值等监控项，Reducer的语义是聚合。
* **IntRecorder**用于记录一段时间内的均值
* **Status**仅存储数值或变量，比如某个变量的值，不同线程修改后立即可见；**PassiveStatus**的特点是被动读取，有需求时才读值
* **LatencyRecorder**实现**时延类**性能采集

bvar各类型拥有自己的聚合方法**Op**和**InvOp**，在模板中使用，比如Adder的Op是**bvar::detail::AddTo<Tp>**, InvOp是**MinusFrom**；在此之上，还要考虑到如何**采集**聚合结果，因此bvar提出**Sampler**来采样。

#### bvar的采集

Sampler的语义是采样，对Variable在某时刻或时段做读值或者快照操作；bvar将Variable的ReducerSampler成员挂载在SamplerCollector的队列中，启动后台线程，定期遍历队列，驱动Sampler采集Variable的值。(针对Reducer、IntRecorder和Window类型)

可以列举以下场景：

1. 每秒读一次Variable的聚合结果，得到某指标的秒级数据
2. 对Variable设置window，得到一个时间段内的聚合结果，在窗口内求最大值、最小值、平均值等
3. 记录现在到之前一分钟内每秒的平均值
4. 记录最近60s内的秒级结果，最近60分钟内的分钟级结果，最近24小时内的小时级结果，最近30天内的天级结果，将这些采样结果维持在一个序列里

* 针对场景1

`ReducerSampler`类型对Variable做每秒采样，后台线程调用`ReducerSampler::take_sample()`方法，驱动Reducer调用`get_value()`等将各线程的tls值聚合出结果，将结果存入自身确定大小的`BoundedQueue`中，该Queue结构使用类似于ringbuffer。另外，`get_value()`也可能会在bvar_dump线程中调用，总之是用于读取Variable的当前聚合结果。

* 针对场景2

说到Window，通俗来讲就是设置一个窗口，将历史值存起来，存就存在Variable的Sampler的Queue中。因此Queue size的可以增大。当然，大部分Variable使用时不考虑Window方式，因此size为1。
举例说明，对一个`bvar::Maxer<int> max`，获取从现在到之前1秒内的最大值，可以设置窗口为1，即`bvar::Window<bvar::Maxer<int>> max_per_second(&max, 1)`，获取从现在到之前1分钟内的最大值，设置窗口60，即`bvar::Window<bvar::Maxer<int>> max_per_minute(&max, 60)`。`bvar::Window`的默认窗口大小是`bvar_dump_interval`，默认10s，用户可自定义。窗口大小将设置ReducerSampler的`BoundedQueue`的大小，且只会增大，不会缩小window。

Window的使用是非常重要的，很多概念借助于它，设想我们要统计流量，首选是`bvar::Adder<uint64_t>`，由于Adder一直是累加的，我们拿到的是累计值。如果我们使用Window，则可以得到一段时间内的流量，因此Window类型的`get_value(time_t window_size)`方法是要指定时间维度的。那么想获得一段时间内的每秒流量呢？这就是场景3.

* 针对场景3

另一种Window的用法是，计算Variable的一段时间内每秒的平均值，类型为`bvar::PerSecond<>`。比如`bvar::PerSecond<bvar::Adder<int>> average_per_second(&adder, 60)`可以得到最近一分钟内的每秒平均值。

Window和PerSecond的使用场景需要多体会，建议多参考[官方文档](https://github.com/apache/incubator-brpc/blob/master/docs/cn/bvar_c++.md#%E5%92%8Cwindow%E7%9A%84%E5%B7%AE%E5%88%AB)

* 针对场景4

**SeriesSampler**将Variable采样的值存成序列，内部Series分配60+60+24+30大小的空间，用于存储秒、分、时和天的采样值，这些聚合值是性能监控最在意的点，极大提高了程序的可观察性。将性能数据聚合工作放在运行时做，用很小的开销实现即时观察功能，试想这部分工作如果放到后台做，需要用flink等平台做聚合，会需要多少台机器，将产生多大成本？还有些性能指标，如果根据同样思路，通过实例单位聚合成进程的，进程的聚合成机器的，也可以省去后端聚合的工作。

> SeriesSampler和ReducerSampler用途不同，所以理解时可以先把SeriesSampler放下。

了解这些语义后，可以通过bvar的单测来使用它。

#### bvar的使用示例

初次接触brpc，跑最简单的echo_c++例程，我就惊艳于brpc完善的监控项，并且在进程中内置http接口，能直接从浏览器读到进程实时更新的各项指标，这就是把活做到位了。如下图所示，图一为echo_server的status监控，图二为echo_server的bvar监控

<div align="center"><img src="https://raw.githubusercontent.com/BG2BKK/githubio/master/static/echo_server_status.png" ><p>图一 echo_server的status</p></div>


<div align="center"><img src="https://raw.githubusercontent.com/BG2BKK/githubio/master/static/echo_server_bvar.png" ><p>图二 echo_server的bvar指标</p></div>


在没有brpc server的情况下，brpc提供一个`brpc::StartDummyServerAt(8080/*port*/);`开启监控端口，因此我们在无brpc server的时候也可以享受这一便利。

bvar模块设计很独立，使用非常简单，可以单独使用。通过`FLAGS_bvar_dump`和`FLAGS_bvar_dump_interval`配置bvar是否dump到文件以及频率，通过`FLAGS_bvar_log_dumpped`配置是否打印在日志中；另外就是http接口。

得益于bvar的使用文档和单测用例，我们可以写一个最小demo，如下所示。

* 打开`http://localhost:8080/vars`可以观察到自己定义的bvar指标
* 在本目录下的monitor目录中，adder、adderWindow和adderPersecond在文件`monitor/bvar.demo_bvar.data`中，其余bvar，比如`a_latency`是dump在`monitor/bvar.demo_bvar.latency.data`中，bvar会根据变量名将bvar分流。


```cpp
#include "bvar/detail/agent_group.h"
#include "bvar/bvar.h"
#include "butil/time.h"
#include "bvar/reducer.h"
#include "bvar/passive_status.h"
#include "brpc/server.h"

#include "gflags/gflags.h"

void dump() {
    bvar::Adder<int> a1("a_latency");
    bvar::Adder<int> a2("a_qps");
    bvar::Adder<int> a3("a_error");
    bvar::Adder<int> a4("process_*");
    bvar::Adder<int> a5("default");

    bvar::Adder<int> c("demo");

    bvar::Adder<int> adder("adder");
    bvar::Window<bvar::Adder<int>> adder_window("adderWindow", &adder, 3);
    bvar::PerSecond<bvar::Adder<int>> adder_persecond("adderPersecond", &adder, 1);

    int cnt = 0;
    while(cnt++ < 1000) {
        c << 1 << 2 << 3;
        adder << 1000 - cnt;
        adder << 1000 - cnt-cnt;
        sleep(1);
    }
}

int main(int argc, char *argv[]) {

    GFLAGS_NS::SetCommandLineOption("bvar_dump_interval", "1");
    GFLAGS_NS::SetCommandLineOption("bvar_dump", "true");

    brpc::StartDummyServerAt(8080/*port*/);

    dump();
}

```

```cpp
export LD_LIBRARY_PATH=/usr/local/lib
g++ -g -O0 var.cpp -lbrpc -std=c++11 -I/pathto/brpc/output/include/ -lpthread -lgflags -DGFLAGS_NS=google -o demo_bvar

```

浏览器打开`http://localhost:8080/vars`


<div align="center"><img src="https://raw.githubusercontent.com/BG2BKK/githubio/master/static/demo_bvar.png" ><p>图三 demo的bvar指标</p></div>


### 最后

bvar是一个性能高、功能全、设计好的性能采集库，它的设计和实现值得深入学习，尤其是把事情做到位的精神。

通读代码的过程中，我能感受到brpc在实现的过程尽力的满足需求，基本没有拒绝过需求。江海不择细流，故能成其大，实现用户需求的过程中丰富基础设施的功能，然后接更多的需求，这是一个正向反馈的过程，这种态度也是值得鼓励学习的。
