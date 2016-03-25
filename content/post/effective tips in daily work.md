+++
date = "2016-03-24T16:36:33+08:00"
draft = false
title = "effective tips in daily work"

+++

rpmbuild环境的快速初始化
-----------------------------------

需要将代码打包为CentOS的RPM包时，可以先自己在本地新建一个环境

```bash
1. mkdir -p ~/rpmbuild/{SOURCES,BUILD,BUILDROOT,RPMS,SRPMS,SPECS}
2. 将代打包的代码压缩包 software.tar.gz 放入SOURCES文件夹
3. 将 software.spec 放入SPECS文件夹
4. rpmbuild -ba path/to/software.spec 即可
```

git记住密码，不用每次都输密码才登入
--------------------------------------

git有两种方式，一种是ssh方式，配置公钥私钥，对于新手而言还是比较麻烦的；另一种是http方式，这里有一个办法可以让git记住密码，避免每次都需要输入密码

```bash
1. touch ~/.git-credentials
2. 将  https://{username}:{password}@github.com  写入该文件
3. git config --global credential.helper store  就可以使得git记住密码了
4. 此时查看 ~/.gitconfig，发现多了一项
    
    [credential] 
    helper = store 
```

centos系统上某些软件，比如gcc、python等版本过低的解决方案
----------------------------------------------------------

在CentOS Server上，经常会遇到某些软件依赖版本过低的问题，比如CentOS 6.5的python是2.7版本的，gcc是4.2版本的，那么我们如何获得一个干净的、与原版本无冲突的运行环境呢。CentOS系提供了一个叫SCL的工具，可以帮我们实现目的

```bash
$ sudo wget http://people.centos.org/tru/devtools-1.1/devtools-1.1.repo -P /etc/yum.repos.d
$ sudo sh -c 'echo "enabled=1" >> /etc/yum.repos.d/devtools-1.1.repo'
$ sudo yum install devtoolset-1.1
$ scl enable devtoolset-1.1 bash
$ gcc --version
# 通过devtoolset工具可以暂时提高gcc版本，而不更改之前服务器的配置，这个很有效果，高版本的gcc会智能保留symbol。
```

```bash
# CentOS 6.5
sudo yum install centos-release-SCL
sudo yum install python27
scl enable python27 bash
python --version
```
ubuntu系统上某些软件，比如gcc等版本过高的解决方案
---------------------------------------------------

与CentOS相反，debian系发行版的软件版本都很高，Ubuntu 16.04的gcc 版本已经到了5.2，然而编译一些早期linux内核的话，需要gcc-4.7左右的版本，这时候我们怎么办呢，有两个方法：
* 通过apt安装低版本gcc
    * sudo apt-get install gcc-4.7
    * 在编译linux 内核时， make CC=gcc-4.7 即可
* update-alternatives可以帮忙更改符号链接，指向不同版本的gcc
    * [参考链接1](http://www.metsky.com/archives/607.html)
    * [参考链接2](http://blog.csdn.net/zyxlinux888/article/details/6708775) [附赠](http://blog.csdn.net/zyxlinux888/article/details/6709036)
    


python的matplotlib库实现绘制图标
---------------------------------------

* sudo apt-get install python-matplotlib

[参考链接](http://matplotlib.org/index.html)
[example](http://matplotlib.org/examples/index.html)

python使用requests库发送http请求
--------------------------------------

[参考链接](http://cn.python-requests.org/zh_CN/latest/user/quickstart.html#json)

python解析命令行参数：argparse
-----------------------------------

[参考链接](http://blog.xiayf.cn/2013/03/30/argparse/)

git比较两次commit的差异
----------------------------

通过比较两次commit的代码差异，能够快速理解此次commit的目的，理解作者意图

* git log
    * 查看commit历史
```bash
commit 2279c3f4a8a42e696a0f34e6e9b6289487da92c1
Author: bg2bkk <bg2bkk@gmail.com>
Date:   Sun Mar 13 09:12:26 2016 +0800

    add SO_REUSEADDR和SO_REUSEPORT.md

commit 2b9d85f8427c5ca9e4f9c128c22acd280eb94405
Author: bg2bkk <bg2bkk@gmail.com>
Date:   Sat Mar 12 01:16:00 2016 +0800

    add 采用二级指针实现单链表操作 单链表翻转 删除单链表结点

```

* git diff commit 2279c3f4a8a42e696a0f34e6e9b6289487da92c1 2b9d85f8427c5ca9e4f9c128c22acd280eb94405
    
git返回强制返回某次提交
----------------------------

* git log
* git reset 5f4769a98985b5acfea45462df27830e51a75145 --hard
    * 可见commit号很重要

iptables允许端口被外网访问
------------------------------

防火墙设置，配置1985端口可以被外网访问 

* sudo iptables -A INPUT -m state --state NEW -m tcp -p tcp --dport 1985 -j ACCEPT


tcpdump过滤指定标志的packet
------------------------------

```bash
# tcp包里有个flags字段表示包的类型，tcpdump可以根据该字段抓取相应类型的包：
# tcp[13] 就是 TCP flags (URG,ACK,PSH,RST,SYN,FIN)
# Unskilled 32
# Attackers 16
# Pester     8
# Real       4
# Security   2
# Folks      1

#抓取fin包：
tcpdump -ni any port 9001 and 'tcp[13] & 1 != 0 ' -s0  -w fin.cap -vvv
#抓取syn+fin包：
tcpdump -ni any port 9001 and 'tcp[13] & 3 != 0 ' -s0  -w syn_fin.cap -vvv
#抓取rst包：
tcpdump -ni any port 9001 and 'tcp[13] & 4 != 0 ' -s0  -w rst.cap -vvv
```
[参考链接](http://babyhe.blog.51cto.com/1104064/1395489)


查看进程的内存占用情况
--------------------------

用Ternary Search Tree代替Trie Tree后，我想知道我的进程内存占用有多大区别。

* ps -e -o 'pid,comm,args,pcpu,rsz,vsz,stime,user,uid' | grep MyDict
    * rsz是实际占用内存，单位是KB

* pmap -d pid
