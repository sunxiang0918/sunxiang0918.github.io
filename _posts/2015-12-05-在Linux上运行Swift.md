---
title: 在Linux上运行Swift
date: 2015-12-05 23:26:26
tags:
- Swift
toc: false
---

# 在Linux上运行Swift

盼星星盼月亮,等了半年,终于在12月的头几天,苹果开源了`Swift`语言.并建立了一个[Swift.org](http://swift.org)社区以及[Github](http://github.com/apple)来维护.开源以后,最大的好处当然是有更多的人来参与Swift语言的发展,让Swift语言增加更多的新的特性,更多的开源框架,工作在更多的平台上面.这对于一个开发语言来说无疑是一个很好的消息.而对于我来说,除了在编写代码的时候能更了解Swift某些函数的参数意义和工作的原理(作为一个JAVA Coder,平时如果遇到搞不定的问题或者是不明白的地方,习惯了直接翻它的源码来了解原委的)外,让我基本上抛弃了`Python`脚本,平时有什么小东西小程序需要写一下的话,现在可以直接写一个Swift文件,然后在命令行直接调用`swift xxx.swift`或者`swiftc -O xxxx.swift`即可.

## 在Linux上安装Swift
前面废话说了这么多.现在就来看看如何在Linux上安装Swift的运行环境.MACOS上的就不用说了,直接安装一个Xcode就可以了.

<!--more-->

### 环境
目前Swift提供了`Ubuntu`上编译好了的安装包. 因此需要Ubuntu14.04以上的操作系统.
同时,Swift的编译环境还需要`clang`.这个也是需要安装的.

1. 由于`clang`目前才出来,比较的新.因此在`cn.archive.ubuntu.com`上还没有,需要切换到官方源上去.
	1. 	备份原来的源列表 `sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak`
	2. 使用编辑器打开`sources.list`文件,修改里面的内容为:
	
		```bash
	deb http://archive.ubuntu.com/ubuntu/ vivid main restricted universe multiverse  
	deb http://archive.ubuntu.com/ubuntu/ vivid-security main restricted universe multiverse  
	deb http://archive.ubuntu.com/ubuntu/ vivid-updates main restricted universe multiverse  
	deb http://archive.ubuntu.com/ubuntu/ vivid-proposed main restricted universe multiverse  
	deb http://archive.ubuntu.com/ubuntu/ vivid-backports main restricted universe multiverse  
	deb-src http://archive.ubuntu.com/ubuntu/ vivid main restricted universe multiverse  
	deb-src http://archive.ubuntu.com/ubuntu/ vivid-security main restricted universe multiverse  
	deb-src http://archive.ubuntu.com/ubuntu/ vivid-updates main restricted universe multiverse  
	deb-src http://archive.ubuntu.com/ubuntu/ vivid-proposed main restricted universe multiverse  
	deb-src http://archive.ubuntu.com/ubuntu/ vivid-backports main restricted universe multiverse  
		```
	3. 然后更新一下源 `sudo apt-get update`
	4. 安装`clang`

		```bash
		sudo apt-get install clang libicu-dev
		```
2. 下载,并安装`Swift`
	1. 在[官方网站](https://swift.org/download/)上下载与你操作系统对应的安装包.比如我用的`Ubuntu 14.10`操作系统,那么就下载[swift-2.2-SNAPSHOT-2015-12-01-b-ubuntu14.04.tar.gz](https://swift.org/builds/ubuntu1404/swift-2.2-SNAPSHOT-2015-12-01-b/swift-2.2-SNAPSHOT-2015-12-01-b-ubuntu14.04.tar.gz)
	2. 下载后,解压到本地目录.由于它都是安排好了目录的.都是`/usr`下面.因此.只需要拷贝到/usr下面即可.

		```bash
		cp -R /swift/usr/ /
		```
	3. 这样,Swift的运行环境就算是安装完成了.我们可以输入`swift --version`来做验证

	```bash
	root@Parallels-Virtual-Platform:~$ swift --version	Swift version 2.2-dev (LLVM 46be9ff861, Clang 4deb154edc, Swift 778f82939c)	Target: x86_64-unknown-linux-gnu
	```
	
## 在命令行中执行一个简单程序
既然安装完成了运行环境,那么接下来我们就试着编写一个最简单的程序.然后执行.

1. 创建一个`test.swift`文件.
2. 在其中写上

```swift
func test() {
	print("Hello Swift!")
}

test()
```

3. 保存文件后,在命令行中执行 `swift test.swift`.即可得到程序执行的结果.

```bash
root@Parallels-Virtual-Platform:~$ swift test.swift
Hello Swift!
```

4. 我们也可以把这个文件直接编译成可执行的程序:`swift -O test.swift`.执行了这个命令后,会在当前目录生成一个`test`可执行文件. 直接在命令行中执行`./test`也是可以得到结果的.

Swift的命令行程序与其他的语言不同,它不需要一个特殊约定的`main`函数. 只要你执行的命令的作用于是全局的,那么在就会在执行的时候按顺序的先执行这些全局作用于的语句.相当于它的全局作用域就是一个大的`main`函数.
又由于一个Swift文件中可以写很多的类或方法,不想`JAVA`一样,一个`java`文件只能有一个公开的类.因此,对于一个简单的小程序来说,我们完全可以把所有的代码都写在一个`swift`文件中,然后进行执行,不用考虑什么包依赖等等,非常的方便.

## 在命令行中执行一个多文件编译的程序
当然,除了最最简单的只有1个文件的程序外,更多的程序都是有代码结构的,都是由多个`swift`文件组成的.
在这种情况下,就需要使用`swift build`命令来 多文件协同编译了.

1. 这种情况下的swift程序源码需要按照一定的约定来创建.
	1. 项目的名称即为目录的名称. 比如我现在有个`TestProgram`的项目,那么就需要创建一个`TestProgram`的目录.
	2. 在这个目录下创建一个`Package.swift`文件,这个文件是必须的,它用于提供给包管理器进行包依赖的信息.这就类似于JAVA中的`package-info.java`
	3. 创建一个`Sources`文件夹,所有的源码都应该放在这里
	4. 在`Sources`文件夹下,创建一个`main.swift`文件,这个就是应用的入口文件.
	5. 最终,项目的结构就是这样的

	```bash
	/TestProgram
	/TestProgram/Package.swift
	/TestProgram/Sources/main.swift
	```
2. 为了体现多文件协同编译,现在再在`Sources`目录下新增加一个文件`Hello.swif`

```swift
func hello(a:(name:String)->Void) {
		let args = Process.arguments
		if args.count >= 2{
			a(name:args[1])
		}else{
			print("Hello Swift!")
		}       }
```

3. 然后在`main.swift`中编写:

	```swift
	hello({name in print("Hello \(name) on Linux!")})
	```

4. 回到项目的根目录`/TestProgram`.执行`swift build`编译即可.正确的话它会输出以下信息

	```bash
	root@Parallels-Virtual-Platform:~/TestProject$ swift build	Compiling Swift Module 'TestProject' (2 sources)	Linking Executable:  .build/debug/TestProject	root@Parallels-Virtual-Platform:~/TestProject$ .build/debug/TestProject
	```
	需要注意的是,`swift build`会在工程目录下生成一个`.build`文件夹,里面就是编译后的可执行的文件,默认是使用的`debug target`. 并且`Linux`下编译的可执行文件是不能直接在`OSX`上使用的,反之亦然.

5. 直接调用`.build/debug/TestProject` 便可执行程序.

6. 第二步的代码中有一句`let args = Process.arguments`.我们可以通过此函数获取命令行的输入,它肯定是一个大于等于1的数组,第一个元素就是程序自己的名字.后面是用户在命令行中输入的参数,并且不仅仅限于`main.swift`才能获取,任何的Swift文件中都可以取得这个值.因此,我们刚才的程序也可以这样输入:`.build/debug/TestProject SUN`.那么程序的`args[1]`即为`SUN`.
7. 由于Swift不需要像`JAVA`或者`OC`一样,如果源码在两个源文件中就需要编写一堆无用的`import`语句,只要在同一个项目中,`Swift`的不同源文件定义的类或函数都可以直接的调用,只有在跨工程或`Framework`的时候,才需要`import Package`.这大大的方便了我们编写项目.

