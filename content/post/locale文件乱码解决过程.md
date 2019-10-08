
# locale造成乱码后的解决方案

## 问题发现过程
突然想使用orgmode，配置vim都正确，但是在\sa插入时间的时候，报错

```bash
Inserting 2019-09-18 三 | Modify date: Traceback (most recent call last):
  File "<string>", line 1, in <module>
  File "/Users/huang/.vim/bundle/vim-orgmode/ftplugin/orgmode/plugins/Date.py", line 254, in insert_timestamp
    timestamp = u'<%s>' % newdate if active else u'[%s]' % newdate
UnicodeDecodeError: 'ascii' codec can't decode byte 0xe4 in position 11: ordinal not in range(128)
```

开始慌，然后去看代码，觉得可以print抢救一下，发现真的可以print，然后就能看到结果，vim采用py做插件这点真好，这就是个天然的调试器啊。

```python
newdate = cls._modify_time(today, modifier)

# format
if isinstance(newdate, datetime):
	newdate = newdate.strftime(
		u_decode(u_encode(u'%Y-%m-%d %a %H:%M')))
else:
	newdate = newdate.strftime(
		u_decode(u_encode(u'%Y-%m-%d %a')))
print 'huang'
print newdate
newdate_decode = newdate.decode('utf-8')
timestamp = u'<%s>' % newdate_decode if active else u'[%s]' % newdate_decode

insert_at_cursor(timestamp)
```
一般这种情况就是中文编码的问题了。
可以看出是date.today().strftime("")导致中文出现，比如说"Inserting 2019-09-18 三"，因此我只要解决汉字即可，
```python
newdate_decode = newdate.decode('utf-8')
timestamp = u'<%s>' % newdate_decode if active else u'[%s]' % newdate_decode
```
这样虽然可以，但是hard code很难看，也不会被上游接受

换个角度来说，我不需要去改代码，我不要让中文出现即可。那么，为什么这里是中文呢

在Date.py模块的init中加入
在终端执行date
```python
def __init__(self):
	u""" Initialize plugin """
	object.__init__(self)
	# menu entries this plugin should create
	self.menu = ORGMODE.orgmenu + Submenu(u'Dates and Scheduling')
	+ print os.system("date")
	+ print(date.today().strftime(u'%Y-%m-%d %a'))

	# key bindings for this plugin
```

再次打开 org文件，vim首先初始化插件模块，在启动页面下方显示

```bash
"org/index.org" 17L, 625C2019年 9月25日 星期三 11时18分33秒 CST

2019-09-25 三
Press ENTER or type command to continue
```
本地调用date
```bash
╰─➤  date                                                                                                                                146 ↵
2019年 9月25日 星期三 11时24分13秒 CST
```
可以肯定是本地字符编码的问题。而这个问题的原因，在于locale。

## locale简单配置

locale，这个在ubuntu时代每次装机必须倒腾的东西，在mac上也要改了。考虑到影响范围，只在** ~/.bash_profile**中将LC_ALL改下。

```bash
export LC_ALL=en_US.UTF-8
export LANG=en_US.UTF-8
```
这样在vim下是英文编码；而不影响整个系统的配置。

然后执行
```bash
source ~/.zshrc
```
用vim打开org文件，输出就变化了

```bash
"org/index.org" 17L, 625CWed Sep 25 11:23:26 CST 2019

2019-09-25 Wed
```

对我而言，日期采用星期三还是Wed来说并无区别，因此采用这种方式规避问题，而不是去改vim-orgmode源码去解决问题，实在是衡量投入产出之后的选择。vim-orgmode的功能比较残，因此最后我也没有再用。

但是关于locale，对于中文环境下的程序员来说，很有必要了解下。

## locale影响

关于locale的影响，有时候会比较蛋疼，在解决问题的过程中，发现两个有趣的帖子:

1. 服务器是ubuntu，用Mac的iterm2 ssh连上去，终端显示中文乱码，也不能输入中文，然而本地终端可以显示和输入。[链接地址](https://gist.github.com/slow-is-fast/1b0e9e8bbcb7e8b8966d177730c46ba4)
2. java调用三方库时出现乱码，这个比较严重，但是还好只是本地会乱码，想来服务端都是英文字符编码的。[链接地址](https://adolphor.com/blog/2018/01/09/configuration-system-encoding-on-macos.html)
3. 调用aliyun oss sdk错误，因为本地locale是zh_CN的，导致python的strftime输出中文，程序报错，和本例很像。[链接地址](https://www.cnblogs.com/nisen/p/6101196.html)

关于locale的影响，我想再举个例子，对于如下代码

```python
# -*- coding: utf-8 -*-
from datetime import date
print(date.today().strftime(u'%Y-%m-%d %a'))
```

无论在zh_CN还是在en_US设置下，都输出

```bash
2019-09-25 Wed
```

难道locale不起作用？确实没起作用。参考[文章](https://www.cnblogs.com/nisen/p/6101196.html)中所讲

> 当在shell里启动python repl(交互器)时,默认的环境local设置为'C', 也就是没有本地化设置，这时候可以通过 locale.getdefaultlocale() 来查看shell当前环境的locale设置， 并通过 locale.setlocale(locale.LC_ALL, '') 将python解释器的locale设置成shell环境的locale

也就是说终端执行py脚本是不会检查本地字符编码的，而vim插件是会设置本地编码的。修改脚本如下

```python
# -*- coding: utf-8 -*-
import locale
from datetime import date

print locale.getlocale(locale.LC_TIME)
locale.setlocale(locale.LC_ALL, '')
print locale.getlocale(locale.LC_TIME)

print(date.today().strftime(u'%Y-%m-%d %a'))
```

可见，由于我们可爱的中文，很多时候中文环境的程序员会受到很多无妄之灾。那么，什么是locale呢？

## locale简介