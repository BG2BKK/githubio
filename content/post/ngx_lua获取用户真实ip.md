+++
date = "2016-09-08T17:06:19+08:00"
draft = true
title = "ngx_lua获取用户真实ip"

+++


* toa, option字段，淘宝lvs，换成直连

* x-real-ip的错误，线上的错误日志

* https://imququ.com/post/x-forwarded-for-header-in-http.html

proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $remote_addr;

nginx作为proxy server时，将remote_addr附加到x-real-ip或者x-forwarded-for头部字段中

一般做法是在x-forwarded-for字段中添加remote_addr，nginx只用到后一句配置；阿里云和新浪的7层都是，所以从xff字段中获取用户ip比较靠谱

```bash
curl http://t1.imququ.com/ -H 'X-Forwarded-For: 1.1.1.1' -H 'X-Real-IP: 2.2.2.2'

remoteAddress: 127.0.0.1
x-forwarded-for: 1.1.1.1, 114.248.238.236
x-real-ip: 114.248.238.236
```

真正的转发，xff字段是逗号分隔的我去
