---
title: MacOS下JAVA开发使用经验(二)
date: 2015-10-11 21:39:59
tags:
- JAVA
- MAC
comments: true
---

# MacOS下JAVA开发使用经验(二)

[上一篇文章](http://sunxiang0918.cn/2015/09/21/MacOS下JAVA开发使用经验(一))写了最常用的一些工具,接下来再介绍一下提高开发生产力的工具.

## 软件

### BetterZip

这个是MAC下很常用的压缩/解压的软件.它对于Windows下的中文支持的比较好.
基本上没有遇到在Windows下压缩的中文字符,到Mac下就无法打开的问题了.
同时他也支持rar格式. 对于我们JAVA的各种`ear`,`war`,`jar`等 都是可以直接打开,然后编辑的.当双击某个jar中的xml文件后会自动打开关联的文件编辑器,然后保存文本编辑器也会自动的更新jar包.

![](/img/2015/10/11/1.png)

### Beyond Compare

这个是我们搞IT的大杀器.以前只有在Windows上有,当时为了这一个软件,我不得不装上了虚拟机.
不得不说,用过这么多文件比较工具,这个始终是神话般的存在.什么`DiffMerge` `FileMerge` `Araxis Merge` `Kaleidoscope` 都弱爆了.
这个工具可以配合你的`sourceTree`等代码管理工具使用,直接清晰地对比和合并代码.同时也可以和`JD`反编译工具配合直接对比`class`文件的差异(这个可以参考我的[另一篇文章](http://sunxiang0918.cn/2014/09/20/在MAC下使用beyondcompare比较JAVA-Class文件/)).

![](/img/2015/10/11/2.png)

<!--more-->

### Dash

这个又是`Mac Only`的一个福利,是一款功能单一却精准的API文档浏览器,以及代码片段管理工具.它采用了单窗口的模式,最方便的提供了API查询的需求.我们平时写东西的时候,最常用的就是查看源码和API了,IDE提供了源码的查看,而`Dash`则提供了`API查询`的功能. 它可以和绝大多数的IDE合作,包括我们JAVA最常用的`Intellij` `eclipse`.只需要在需要查看API的代码上按住`Command+Shift+左键`(这个可以自己设置),那么就会自动的跳转到它的界面上并显示这段代码的API.不需要我们到处去翻了.
它基本上集成了绝大多数的API文档,包括`JDK` `Spring` `Hibernate`等等,同时也支持三方的推送,只要是符合标准的url地址,都可以添加到它里面进行管理.

![](/img/2015/10/11/3.png)

### JD-GUI

这个是Mac下的可视化的`Java`反编译工具.使用很简单,直接双击打开界面后,把`class`文件或者文件夹直接拖入他的窗口就可以了.然后就会自动的反编译好,如果是一个文件夹的话,他还支持代码间的跳转.这对有些时候来说有奇效.

![](/img/2015/10/11/4.png)

### MongoHub

这个是在Mac下非常好用的`Mongo`可视化客户端.它基本上提供了`Mongo`的所有操作,并且支持Mongo的集群连接,同时,它是免费[开源的](https://github.com/jeromelebel/MongoHub-Mac).如果在开发中需要使用`Mongo`数据库.那么这个基本上是不二的选择.

![](/img/2015/10/11/5.png)

### Navicat Premium

这个是多平台下的多数据库管理工具,能支持`Oracle`,`Mysql`,`Sqlite`,`PostgreSQL`.不过我主要是拿它来管理`Oracle`和`Sqlite`.它也提供了很全面的数据库管理功能.比如导入导出,执行函数,事件,任务等等.但是这个工具使用起来有点慢,而且容易崩溃(我也不知道为什么,已经遇到很多次了),所以不怎么常用.不过用来临时操作下`Oracle`或`Sqlite`还是推荐.

![](/img/2015/10/11/6.png)

### OmniDiskSweeper

这个工具也是平时使用Mac必不可少的,主要是用来查看磁盘空间被什么文件使用的.通过这个软件可以很容易的看到你的磁盘到底是被什么占用了这么多.自从又一次无意间发现企鹅聊天占用了我系统盘10个G空间存放莫名其妙的东西后,我就养成了定时执行一次这个软件的习惯,谁叫我们是屌丝只买得起256G的SSD喃..空间还是要节约到用..

![](/img/2015/10/11/7.png)

### OmniGraffle&OmniPlan

这两个工具是用来画UML和项目甘特图的,也是在Mac上算是比较好用的了.它们对应了Windows上的`Visio`和`Project`.如果你们是一个传统点的软件开发公司,相信采用瀑布开发模型的肯定是需要这两个软件的.

![](/img/2015/10/11/8.png)

![](/img/2015/10/11/9.png)

### Sequel Pro
这个是在Mac环境下的Mysql的管理软件.完全免费的,并且是原生的Mac软件,使用速度比`Navicat`快.功能也和`Navicat`是一样的.因此,我平时更倾向于使用这个工具来管理`Mysql`.
它的开源地址是:[https://github.com/sequelpro/sequelpro](https://github.com/sequelpro/sequelpro)

![](/img/2015/10/11/10.jpg)

### Microsoft Remote Desktop

这个工具也是在工作中基本上跑不掉的.它是用来远程Windows系统的,相当于Windows上的远程桌面.没有办法,我们的环境中还是有一半的Windows机器作为媒体处理的执行器.平时需要远程上去调试些东西.不过还算微软厚道,这个工具还是比较好用的,能支持同时打开几个文件.并且能直接粘贴复制远程桌面的东西到本地.

![](/img/2015/10/11/11.png)

### Transmit

非常好用的sftp工具,他提供了`FTP` `SFTP` `S3` 的文件访问.并且支持标签组的操作,以及直接双击打开关联的工具修改文件并自动上传.还支持各种姿势的拖拽操作.总之非常方便,强烈地推荐.

![](/img/2015/10/11/12.png)

### Brew
brew 又叫Homebrew，是Mac OSX上的软件包管理工具，能在Mac中方便的安装软件或者卸载软件， 只需要一个命令， 非常方便.类似于Ubuntu上的`apt-get`.他能方便的管理我们的各种软件的依赖,这些工具通常都是些比较专业的命令行工具,安装不是需要编译一大堆就是需要下载各种依赖包,总之不简单.用了这个之后,一句话而已,就搞定了.
要安装Brew也很简单,在终端中输入 `ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"`即可.
安装后 就可以在终端中输入`brew` 来进行各种操作了.
比如:

```bash
brew install git
brew install wget
brew search /*get/
brew list
brew update
```

### zsh
Linux/Unix提供了很多种Shell,除了大家最常用的bash以外,`Macos`还预装了一个`zsh`.这个可是一个大杀器,被称为终极Shell,但是由于配置过于复杂,用的人始终不多.直到有一天,有个外国友人搞了一个`oh my zsh`的项目,可以让你很容易的使用上`zsh`.

1. 切换到zsh
	直接在终端中输入 `chsh -s /bin/zsh` 即可.
2. 安装oh my zsh
	这个也直接在终端中输入 `sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"` 即可.它会自动的下载源码并编译然后安装配置一条龙搞完.安装完成后重新打开一个终端窗口,你就可以看到一个彩色的提示了
![](/img/2015/10/11/13.png)
3. 修改 ~/.zshrc配置
	这个配置文件里面就是zsh的自定义配置项了,可以在这里配置zsh的主题,插件,命令行别名等等.
	比如以下就是我常用的别名:
	
	```bash
		alias hexog='hexo generate'
alias hexod='hexo deploy'
alias hexon='hexo new'
alias cls='clear'
alias ll='ls -l'
alias la='ls -a'
alias vi='vim'
alias javac="javac -J-Dfile.encoding=utf8"
alias grep="grep --color=auto"
alias -s html=st   # 在命令行直接输入后缀为 html 的文件名，会在 sublimeText 中打开
alias -s rb=st     # 在命令行直接输入 ruby 文件，会在 sublimeText 中打开
alias -s py=st       # 在命令行直接输入 python 文件，会用 sublimeText 中打开，以下类似
alias -s js=st
alias -s c=st
alias -s java=st
alias -s txt=st
alias -s gz='tar -xzvf'
alias -s tgz='tar -xzvf'
alias -s zip='unzip'
alias -s bz2='tar -xjvf'
	```
	主题的话,推荐使用`agnoster`,颜色比较好看,并且能以标签的形式展示Git的分支等等,就像上面的截图一样.不过这个主题需要安装`Menlo-Powerline.otf`字体.并且使用`d6a36b1`提交上的`agnoster `主题文件,否则是会有乱码的.

4. 使用zsh
	* zsh完全兼容bash,原来怎么使用bash的就可以怎么使用zsh,命令完全一样.
	* 提供了强大的历史记录功能,上下箭头可以翻阅所有你执行过的命令,并且还可以加限制条件,比如先输入一个`cd`,然后上下箭头显示的就是 你所有执行过的`cd` 命令.
	* 各种补全功能,它的Tab补全功能不像bash是严格区分大小写的,他可以自动识别大小写,这样就再也不用不停的大小写切换了.路径补全如果遇到相同开头的路径了,多按几次tab键是可以直接选择的,而不像以前只能再不断的输入字母直到路径唯一.命令补全功能也很强大,它可以理解大多数的命令参数.比如你想杀掉java的进程,以前需要先输入`ps`命令找到`java`进程的`PID`.然后执行`kill PID`来杀掉进程.使用zsh可以直接输入`kill java`+tab键,他会列出现在所有的java进程,一个回车,zsh会自动的替换为进程的pid.
	* 目录浏览和跳转,在命令行中输入d会列出你再这个会话里面访问过的目录列表,并且前面还有序号.接下来你只需要输入序号,就可以直接跳转到当前目录了.这样就可以直达很多地方,非常的方便.以前需要一步一步的`cd`.
	* 支持直接输入`..`或`...`,或者直接输入当前目录的名字,基本上再也不需要cd命令了.以前经常不小心在`cd ..`中间没有加上空格.命令不起作用.这个就再也不会出现这种情况了.
	* 智能跳转,在安装了autojump之后,配合zsh,会自动记录你访问过的目录.然后它自动的构建一个遍历树结构.以后你直接通过 j+目录名 就可以直接进行目录的跳转了.大多数情况下还是比较靠谱的.比如你访问过`/Application/tomcat/bin`.  那么你直接输入`j tbin`它就能自动的跳转到那去,非常的方便.
	* `ctrl+r`可以搜索你以前输入过的命令,并重新执行. 比如你以前输入过`nano ~/.zshrc`.那么现在你只要点击`ctrl+r`后,输入`zsh` 就可以直接显示出`nano ~/.zshrc`,一个回车就重复执行了.


## 总结

其实基于Unix的`MacOs` 真的非常适合我们程序猿的使用(dotNet的除外).它不光提供了方便好用的GUI环境,还提供了强大的类`Unix`的命令行操作.很多Linux上的操作或命令或经验,基本上可以直接在Macos上使用.对于我们Java开发来说,什么`activemq` `elasticsearch` `flume` `hadoop` `JBOSS` `kafka` `mongodb` `neo4j` `redis` `tomcat` `zookeeper`些,基本上都可以直接下载下来然后运行`sh`就可以了.像`redis`这些需要编译的,在你安装过`xcode`的情况下,基本上一个`make`命令就搞定了.
如果没有试过,建议大家可以先装一个黑苹果试一试.我相信用过一段时间后,会喜欢上在Mac上做开发的.
MacOS绝不不仅仅是一个装逼的花瓶!




