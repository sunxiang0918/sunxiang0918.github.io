---
title: 在MAC下使用beyondcompare比较JAVA Class文件
date: 2014-9-20 10:33:29
tags:
- Mac
- JAVA
toc: false
comments: true
---

#在MAC下使用beyondcompare比较JAVA Class文件

2014年9月1日.beyondCompare终于推出了Mac版了.真的是大快人心的大好事,大了又大.
以前用过很多MAC上的比较工具,像什么`Araxis Merge` `DiffFork` `DiffMerge` `Kaleidoscope`等等. 都没有很好用. 对比Windows平台上的BeyondCompare,差的不是一点半点.

使用beyondcompare对比.class文件的时候,默认是直接对比的二进制文件.这基本上就看不懂.因此,需要在对比的时候自动的反编译为源代码.然后再进行对比.

<!--more-->

1. 打开beyondcompare. 
2. 选择 `BeyondCompare—>File Formats`
3. 然后新建一个文本的 解析格式:  class
	![](/img/2014/09/20/2.png)
4. 在`general` 里面 过滤格式 输入  `*.class`
5. `Conversion`里面  选择  `External Program`.
6. `Loading` 里面输入   `had -p %s >%t`
7. 选择上 `Disable editing`
8. 在[http://varaneckas.com/jad/](http://varaneckas.com/jad/)上面下载适用于MAC使用的JAD软件.并解压到第三步配置的地方.

这样就OK了.使用beyondCompare直接比较JAVA Class文件
