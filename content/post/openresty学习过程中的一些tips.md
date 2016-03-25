+++
date = "2016-03-14T16:36:14+08:00"
draft = true
title = "openresty学习过程中的一些tips"

+++


- [thread](https://groups.google.com/forum/#!topic/openresty/fQvG_TvDAvU)

    - 虽然init_worker_by_lua阶段不能使用cosocket，不过可以先通过一个timer（定时时间为0让其立即调用）来发出对外的socket io操作，以实现一些初始化的目的。
    - openresty的两个缓存中，ngx shared dict是跨worker共享的，是一个单纯的kv缓存；预计接下来会有patch能够支持lpush等redis操作；lua-resty-lrucache是每个worker的Lua VM空间内缓存，不能跨worker共享，优点是可以存储所有lua对象，比如table，而不需要序列化和反序列化

- worker 启动时，upstream 是空的，即 _M.data={}，所以这个时候是不能提供服务的。所以每次 reload config 都会导致一段时间内服务不可访问。
    - 在 init_worker_by_lua 执行 cosocket 相关的 API 是不允许的（后期可能会添加支持），但可以调用标准 SOCKET 完成初始化加载，例如借助 luasocket 完成数据源获取并初始化 _M 。

- 不清楚 init_worker_by_lua 里是否可以进行文件操作？
    - 是可以的，这个确定。

