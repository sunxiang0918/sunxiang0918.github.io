---
title: 升级CentOS7的Linux内核
date: 2015-11-08 15:56:29
tags:
- Linux
toc: false
---

#升级CentOS7的Linux内核

默认刚安装的CentOS7的内核是3.10的.

这个可以在 终端中输入:

```bash
uname -r
3.10.0-229.el7.x86_64
```
来确认.

由于我需要试验docker1.9中的新功能.而它的新功能需要`vxlan`的相关功能.而这个在3.16以上版本才正确.因此,就需要给CentOS7升级内核.

<!--more-->

##步骤

1. 首先在命令行中输入:`uname -r` 来确定你现在的版本是什么

2. 而后输入:`rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org` 导入Key

3. 输入:`rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm`来安装elrepo的yum源

4. 在这个源中,保留了内核的最新两个版本.应用名字叫:`kernel-ml`.因此 我们输入:`yum --enablerepo=elrepo-kernel install kernel-ml -y`.就可以安装最新的内核了.

5. 而后输入`awk -F\' '$1=="menuentry " {print $2}' /etc/grub2.cfg` 可以查看启动CentOS的顺序.

6. 如果想自动的进入第一个内核就是最新的话,可以输入:`grub2-set-default 0`

7. 这样,重启的话.系统就会是最新的内核了.

PS: 如果不想安装最新的内核版本.而是想选一个早先的版本的话.可以访问这个地址:[dfw.mirror.rackspace.com](http://dfw.mirror.rackspace.com).他里面有很多版本的镜像.找到你想要的版本的rpm.直接下载下来.然后上传到Centos中.然后执行`yum localinstall xxxx.rpm -y` 就可以本地安装内核了. 剩下的操作和在线的是一样的.


