title: Macos 通过安装Hexo 来搭建 GitHub Pages 博客系统
date: 2015-07-25 18:06:02
updated: 2015-07-25 18:06:02
comments: true
tags: 
- Hexo
- 博客
- github
categories:
- 其他
description: 这篇文章记录了如何使用Hexo和github搭建个人博客
toc: false
---

现在越来越多的人愿意使用独立的技术博客.如果自己搭建Wordpress等,需要涉及到服务器的问题.所以,很多人选择了GitHub提供的Pages来搭建个人博客,我也赶一回潮流.在MacOS 上 使用Hexo来搭建GitHubPages博客.

## Hexo
hexo出自台湾大学生 [tommy351](http://twitter.com/tommy351) 之手，是一个基于Node.js的静态博客程序，其编译上百篇文字只需要几秒。hexo生成的静态网页可以直接放到GitHub Pages，BAE，SAE等平台上。

### 安装Hexo

要安装`Hexo`需要先安装`Npm`以及`NodeJs`.
我在MacOS上,是使用[Brew](http://brew.sh)安装的.

 1. 安装**NodeJS**
 
    ``` bash
    $ brew install node
    ```

 2. 安装**Git**
    由于我安装了XCode的,并且安装了`Command Line Tool`,因此,这一步可以省略了.否则还是在终端中输入:

    ``` bash
    $ brew install git
    ```
    <!--more-->
    
 3. 安装**npm**
    ``` bash
    $ brew install npm
    ```

 4. 安装**hexo**  
    这个就使用nodeJS的安装程序了.
    同样在终端输入:
    ``` bash
    $ npm install -g hexo
    ```
    这个步骤比较慢.因为你懂的
    
 5. 这个时候可以验证一下是否安装好了.
    在终端中输入:
    ``` bash
    $ node -v
v0.12.7
$ npm -v
    2.12.1
    $ hexo -v
    hexo: 3.1.1
os: Darwin 14.4.0 darwin x64
http_parser: 2.3
node: 0.12.7
v8: 3.28.71.19
uv: 1.6.1
zlib: 1.2.8
modules: 14
openssl: 1.0.1p
    ```
    这样就说明安装完成了.
    但是如果是显示的: ```hexo: command not found ```. 说明环境变量没有设置.我也不知道为什么.但是只要补上环境变量就可以了.
    > **hexo环境变量的设置:** 在`~/` 用户的根目录下创建一个目录:`.bash_profile`.其中的内容为:`export PATH="/usr/local/Cellar/node/0.12.7/libexec/npm/lib/node_modules/hexo/bi$`  其中的路径就是hexo的安装路径

### 使用Hexo创建博客
当安装完成后,就可以开始创建博客了.

 1. 在本地创建博客文件夹
    这一步的目的是在你的本地创建一个博客的文件夹.以后博客的source以及编译后的静态文件都会在这个目录中.

    ``` bash
    $ cd ~/
    $ mkdir blog
    ```
    
 2. 初始化博客文件夹
    这一步是用于初始化hexo的一些文件的

    ``` bash
    $ cd ~/  
    $ hexo init blog  
    $ cd blog  
    $ ls  
    _config.yml	node_modules	public		source
db.json		package.json	scaffolds	themes
    ```

 3. 初始化上下文
 
    ``` bash
    $ cd ~/
    $ npm install
    ```

上面的步骤完成后,就完成了hexo的初始化的过程了.
接着就可以开始关注于博客的编写了.

``` bash
$ hexo new "新博客的名字"
```

这样就在`_posts`文件夹里面新增加了一个**md**文件.直接对这一篇文档进行内容的编写就可以了.

而后,就在命令行中执行 `$ hexo generate` 就可以生成新的静态文件.新生成的文件全部放在`public`文件夹中的.

### 使用Hexo部署博客到github
要使用github的pages功能的话,就需要创建一个 `xxxx.github.io`的**repository**. 其中`xxxx`表示的是你的github账号.这样github就会为你分配一个`xxxx.github.io`的地址.以后你的博客的访问地址也就是这个了.

有了这个地址以后,就要开始使用hexo部署了.
修改博客文件夹下的`_config.yml`

主要是:
``` bash
deploy:
  type: git
  repo: https://github.com/xxxx/xxxx.github.io.git
```
这个部分.
而后就在命令行中输入:

``` bash
$ hexo deploy
```
而后他就会自动的部署到你的github的pages中了. 
如果报错说 未找到部署类型的话. 就需要安装`hexo-deployer-git`

同样是在博客的目录中执行:

``` bash
$ npm install hexo-deployer-git --save
```

---
###后续
这样就搭建完毕了, 后续我会慢慢的把以前记录到 evernote的东西 精选一些转过来. 