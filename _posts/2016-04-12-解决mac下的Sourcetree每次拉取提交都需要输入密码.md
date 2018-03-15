title: 解决mac下的Sourcetree每次拉取提交都需要输入密码
date: 2016-04-12 16:58:20
tags:
- Mac
- GIT
---

# 解决mac下的Sourcetree每次拉取提交都需要输入密码

最近重装了一次mac,并且重做了一下开发环境,结果以前的sourceTree项目的GIT密码始终保存不到Mac的钥匙串中,明明在钥匙串中是存在的.但是在使用sourceTree pull/push代码的时候还是需要再输入密码,很是繁琐. 

<!--more-->

于是,网上搜索了一下,说的在https模式下,Mac需要使用osxkeychain凭据助手,并在Git中设置使用. 并且如果已经安装了`brew`的应该会自带了`osxkeychain`.但是奇怪的是,我安装了brew的,使用brew安装应用也没有问题.那就只能手动的再设置一次了.

## 方法

1. 先使用命令下载 `git-credential-osxkeychain`
    `curl http://github-media-downloads.s3.amazonaws.com/osx/git-credential-osxkeychain -o git-credential-osxkeychain`

2. 把`git-credential-osxkeychain` 放入 bin目录
    `mv git-credential-osxkeychain /usr/local/bin`

3. 给`git-credential-osxkeychain`赋权限
    `chmod u+x /usr/local/bin/git-credential-osxkeychain`

4. 在Git全局配置中进行设置(也可以在某一个项目里面设置):
    `git config --global credential.helper osxkeychain`
    
经过上面的设置，下次访问https的项目时只需要输入一次密码,就会存储到osx的钥匙串中了,以后再也不会在Git中询问了.

问题解决.



