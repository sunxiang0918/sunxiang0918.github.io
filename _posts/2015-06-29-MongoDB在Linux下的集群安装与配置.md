---
title: MongoDB在Linux下的集群安装与配置
date: 2015-06-29 21:24:24
tags:
- Linux
- MongoDB
comments: true
---

#MongoDB在Linux下的集群安装与配置

##一.MongoDB集群方式
**Mongodb**是时下流行的NoSql数据库.它拥有三种集群的搭建方式:`Replica Set` / `Sharding` / `Master-Slaver`.   分别为 副本集方式集群, 分片集群, 主从集群.  
三种集群搭建方式首选`Replica Set`，只有真的是大数据，Sharding才能显现威力，毕竟备节点同步数据是需要时间的。Sharding可以将多片数据集中到路由节点上进行一些对比，然后将数据返回给客户端，但是效率还是比较低的说。  

这里用最常用的 Replica的方式集群来做介绍

##二.Replica Set 

`Replica Set`集群当中包含了多份数据，保证主节点挂掉了，备节点能继续提供数据服务，提供的前提就是数据需要和主节点一致。如下图：
![](/img/2015/06/29/1.png)

`Mongodb(M)`表示主节点，`Mongodb(S)`表示备节点，`Mongodb(A)`表示仲裁节点。主备节点存储数据，仲裁节点不存储数据。客户端同时连接主节点与备节点，不连接仲裁节点。

使用的时候 可以通过配置 让M负责写,S负责读.从而达到读写分离的效果

仲裁节点是一种特殊的节点，它本身并`不存储数据`，主要的作用是决定哪一个备节点在主节点挂掉之后提升为主节点，所以客户端不需要连接此节点。这里虽然只有一个备节点，但是仍然需要一个仲裁节点来提升备节点级别。

<!--more-->
##三.集群搭建
我们在这里使用了两台机器来安装.  
![](/img/2015/06/29/2.png)

147和148两个作为两个Replica节点,负责数据的存储.同时这两个机器又同时作为仲裁节点负责节点的提升

* 下载mongodb
在官网上直接下载:[https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-ubuntu1204-3.0.4.tgz](https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-ubuntu1204-3.0.4.tgz)

* 上传到两台机器上去:  

	```bash
scp /Users/SUN/Downloads/mongodb-linux-x86_64-ubuntu1204-3.0.4.tgz root@172.16.128.147:~/
```

* 解压到目的地:
	
	```bash
tar zxvf mongodb-linux-x86_64-ubuntu1204-3.0.4.tgz -C /usr/local/
```

* 修改目录的名字:

	```bash
mv mongodb-linux-x86_64-ubuntu1204-3.0.4/ mongo
cp -r mongo/ mongo2
```

* 建立数据文件夹:

	```bash
	mkdir data
	mkdir data/db
	mkdir log
	```
	
* 建立mongodb的配置文件:

* Replica Node1:
	
```bash
		#master.conf
		dbpath=/mongo/data/db
		logpath=/mongo/log/db.log
		pidfilepath=/mongo/db.pid
		directoryperdb=true
		logappend=true
		replSet=testrs
		bind_ip=172.16.128.147
		port=27017
		oplogSize=1000
		fork=true
		noprealloc=true
```
	
* Replica Node2:
	
```bash
		#master.conf
		dbpath=/mongo/data/db
		logpath=/mongo/log/db.log
		pidfilepath=/mongo/db.pid
		directoryperdb=true
		logappend=true
		replSet=testrs
		bind_ip=172.16.128.148
		port=27017
		oplogSize=1000
		fork=true
		noprealloc=true
```
	
* Replica arbiterNode1:
	
```bash
		#arbiter.conf 
		dbpath=/mongo2/data/db
		logpath=/mongo2/log/db.log
		pidfilepath=/mongo2/db.pid
		directoryperdb=true
		logappend=true
		replSet=testrs
		bind_ip=172.16.128.147
		port=27018
		oplogSize=1000
		fork=true
		noprealloc=true
```
	
* Replica arbiterNode2:

```bash
		#arbiter.conf 
		dbpath=/mongo2/data/db
		logpath=/mongo2/log/db.log
		pidfilepath=/mongo2/db.pid
		directoryperdb=true
		logappend=true
		replSet=testrs
		bind_ip=172.16.128.148
		port=27018
		oplogSize=1000
		fork=true
		noprealloc=true
```

**参数解释：**
`dbpath`：数据存放目录
`logpath`：日志存放路径
`pidfilepath`：进程文件，方便停止mongodb
`directoryperdb`：为每一个数据库按照数据库名建立文件夹存放
`logappend`：以追加的方式记录日志
`replSet`：replica set的名字
`bind_ip`：mongodb所绑定的ip地址
`port`：mongodb进程所使用的端口号，默认为27017
`oplogSize`：mongodb操作日志文件的最大大小。单位为Mb，默认为硬盘剩余空间的5%
`fork`：以后台方式运行进程
`noprealloc`：不预先分配存储

* 设置全局的Local环境变量:
	
	```bash
	nano /etc/default/locale
	```
	在locale文件中 加入:  `LC_ALL="zh_CN"`
	然后再执行:  
	
	```bash
	export LC_ALL=zh_CN
	```
	
* 启动mongodb:
	分别在三台机器上执行:
	
	```bash
	./mongod -f ../dbconfig.conf
	```
	![](/img/2015/06/29/3.png)

* 完成启动:
	这个时候可以输入:`pgrep mongo -l`  来判断mongo服务是否启动

* 配置Replica Set:

	随便登陆任意一个mongodb:
	
	```bash
	./mongo 172.16.128.147:27017
	```

	```bash
	>use admin

	>cfg={ _id:"testrs", members:[ {_id:0,host:'172.16.128.147:27017',priority:2}, {_id:1,host:'172.16.128.148:27017',priority:1},   
{_id:2,host:'172.16.128.147:27018',arbiterOnly:true},{_id:3,host:'172.16.128.148:27018',arbiterOnly:true}] };

	>rs.initiate(cfg)
	```
	
到此,就给mongodb 指定了 `replica`模式的 集群了.

这个时候可以通过: `>rs.status()` 命令来查看集群的状态
	
* 配置成为ubuntu的服务:
	在`/etc/init.d/`目录下新建脚本文件mongodb
	
	```bash
	#!/bin/sh

	### BEGIN INIT INFO
	# Provides:     mongodb
	# Required-Start:
	# Required-Stop:
	# Default-Start:        2 3 4 5
	# Default-Stop:         0 1 6
	# Short-Description: mongodb
	# Description: mongo db server
	### END INIT INFO

	. /lib/lsb/init-functions

	PROGRAM=/usr/local/mongo/bin/mongod
	MONGOPID=`ps -ef | grep 'mongod' | grep -v grep | awk '{print $2}'`

	test -x $PROGRAM || exit 0

	case "$1" in
	  start)
	     ulimit -n 3000
	     log_begin_msg "Starting MongoDB server”
	     export LC_ALL=zh_CN
	     $PROGRAM -f /usr/local/mongo/dbconfig.conf
	     log_end_msg 0
	     ;;
	  stop)
	     log_begin_msg "Stopping MongoDB server”
	     export LC_ALL=zh_CN
	     $PROGRAM --dbpath /usr/local/mongo/data/db --shutdown
	     log_end_msg 0
	     ;;
	  status)
	     ;;
	  *)
	     log_success_msg "Usage: /etc/init.d/mongodb {start|stop|status}"
	     exit 1
	esac

	exit 0
```
然后给这个文件 增加运行权限 `chmod +x /etc/init.d/mongodb`

这样就可以使用:

```bash
sudo service mongodb stop
sudo service mongodb start
```
启动或停止服务了

如果再加一句: `update-rc.d mongodb defaults`
那么就会 开机自启动

##四.集群测试
正常运行的时候 应该是有 两个数据节点 和两个决策节点. 其中147为主  148为辅

使用工具连接到mongo中去.这时显示的是:
![](/img/2015/06/29/4.png)

然后直接kill mongo 进程:   sudo kill 1004

然后再连接数据库,这个时候会发现 连接变了.
![](/img/2015/06/29/5.png)

这个时候继续正常的操作.  插入新的数据等等.
而后,重新启动147节点. 会发现 一切数据恢复正常. 

集群成功
