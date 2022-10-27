+++
date = '2016-04-03T13:14:13+08:00'
draft = true
title = 'googletest framework初步上手'

+++


刷leetcode的时候，需要调试代码。
需要本地有开发环境，结果没有，上学的时候还自己弄了一个，lowb的很。近来还是想刷leetcode，发现还是需要个环境，这个时候我的想法就不一样了。上学的时候是，看见问题了就直接去解决这个问题，并不会多想；上一年班后，现在看这个问题，就会寻思下，有没有更好的办法，业界通用做法是什么，业界里的标杆公司和标杆人物又是怎么做的。

我寻思了我的场景，发现我应该是要做单元测试的，写好一个函数或者接口，提交给测试框架（leetcode），那么就找找吧。

所以我第一步是先google，关键词“c 单元测试”，中文信息还是比较少的，直接搜"c unit test"，效果还是不错的。首先是[stackoverflow的结果](http://stackoverflow.com/questions/65820/unit-testing-c-code)，提到很多，junit，cunit等等，最后一个答案是googletest，眼前一亮，github搜一下，[得到结果](https://github.com/google/googletest)，是一个一直活跃着的项目。两篇博文，['testing c++ code with the GoogleTest framework'](https://meekrosoft.wordpress.com/2009/10/04/testing-c-code-with-the-googletest-framework/)和['unit testing c code with the GoogleTest framework'](https://meekrosoft.wordpress.com/2009/11/09/unit-testing-c-code-with-the-googletest-framework/)。

googletest其实由googletest和googlemock组成，前者是单元测试，后者是模拟c模块的。

[googletest](https://github.com/google/googletest/blob/master/googletest/README.md)的使用文档和[陶辉的实践过程](http://blog.csdn.net/russell_tao/article/details/7333226)，还有[coderzh的实践过程](http://www.cnblogs.com/coderzh/archive/2009/04/06/1426755.html)，[IBM的教程](https://www.ibm.com/developerworks/cn/linux/l-cn-cppunittest/)
