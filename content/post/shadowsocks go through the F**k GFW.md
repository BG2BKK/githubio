+++
date = '2016-04-11T00:33:17+08:00'
draft = true
title = 'shadowsocks go through the Fuck GFW'

+++



* 买VPS，vultr的设备，使用起来还是蛮简单的。信用卡一张，注册并扣费0.1美元会送50美元，两个月内用完。我开始部署了一个日本的最便宜的5美元一月的vps，使用起来还不错，youtube 480P没问题；但是我通过这送的50美元实验了以下几个配置的机器速度，洛杉矶机房20美元，洛杉矶机房5美元，日本机房5美元，日本机房10美元，发现ping vps-ip时美国机房都是170ms，日本机房是220ms，原因可能是即使日本离得近，去日本的路由也要绕道美国才到日本的，所以我们直接选择美国机房好了。位于西海岸的洛杉矶机房，让你打开youtube 1080P毫无压力，白天晚上都没问题。

* 如何手动搭建shadowsocks服务呢，网上有太多的教程了
    * python版shadowsocks
        * ss服务的鼻祖，支持多用户多端口配置
        * 参考教程：[py版ss服务](https://pypi.python.org/pypi/shadowsocks)

```bash
wget --no-check-certificate https://raw.githubusercontent.com/tennfy/shadowsocks-libev/master/debian_shadowsocks_tennfy.sh
chmod a+x debian_shadowsocks_tennfy.sh

sudo ./debian_shadowsocks_tennfy.sh
```
        * /etc/init.d/shadowsocks-libev start

* shadowsocks客户端
    * 全平台：windows、linux、OSX；android、ios；
    * [客户端](https://shadowsocks.com/client.html)

    * [windows](https://github.com/shadowsocks/shadowsocks-windows/releases)
    * [linux](https://github.com/shadowsocks/shadowsocks-qt5/wiki/Installation)
        * ubuntu用户建议使用apt-get安装
        * 其他发行版用户。。。我还不知道
    * [OSX](https://github.com/shadowsocks/shadowsocks-iOS/releases)
        * 我没用过
    * [android](http://pan.baidu.com/s/1YbQTg)
    * [ios](http://www.iyingsuo.com/ios-shadowsocks-tutorials.html)

* [privoxy将sock5转为http连接](https://gist.github.com/Alexniver/9a4f1791fe4305b0750a)
    * [macos](http://tblog.im/2015/09/23/shi-yong-privoxyzhong-zhuan/)

* 如果您觉得shadowsocks很容易搭建起来，想试一下的话，可以通过的推荐链接来注册，这样会有一定奖励:http://www.vultr.com/?ref=6870148


