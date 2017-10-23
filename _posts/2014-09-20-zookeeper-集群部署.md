---
title: zookeeper 集群部署
date: 2014-09-20 19:55:15
tags:
- Zookeeper
- 集群
- 大数据
comments: true
---

# zookeeper 集群部署

在现在的软件系统中,通常都会搭建集群或者分布式环境了.而在这种情况下,通常都需要一个分布式程序协调者的服务.这最常用的就是zookeeper了.
在我们的系统中,会使用zookeeper来做`统一的配置管理`,集群`节点状态管理`,以及`分布式锁`的功能.
为了保证zookeeper本身的高可用性,那么这就需要对ZK进行集群.

## 安装环境
操作系统:Ubuntu 12.04 64位
JDK:1.7.0_55 64位
机器: `192.168.1.100` `192.168.1.101`  `192.168.1.101`

## 安装步骤
<!--more-->
### 1. 下载Zookeeper
直接到他的官网上下载最新的zookeeper软件: [http://zookeeper.apache.org/releases.html#download](http://zookeeper.apache.org/releases.html#download)

![](/img/2014/09/20/1.png)

### 2. 上传Zookeeper安装包到服务器
这个随便使用什么东西上传都可以, `sft` `scp` 等等

### 3. 为Zookeeper创建目录.
在bash中创建文件夹. 我把目录建到 /usr/local/下的.

```bash
$ mkdir /usr/local/zookeeper
```

### 4. 解压安装包到zk的目录

```bash
$ tar –xzvf zookeeper-3.4.6.tar.gz /usr/local/zookeeper
```

### 5. 再创建几个ZK必要的文件夹
ZK还需要创建几个运行时必要的文件夹: 一个用来存放数据`data`,一个用来存放数据日志的`datalog`,以及一个存放ZK运行时日志的目录`logs`

```bash
$ mkdir /usr/local/zookeeper/data
$ mkdir /usr/local/zookeeper/datalog
$ mkdir /usr/local/zookeeper/logs
```

### 6. 在data目录中创建一个myid文件
这个文件主要是用于标示自己是哪一个服务器的.文件的内容很简单,里面就是一个数字.
比如自己的server1,那么里面就写一个`1`.如果是server2,那么里面就写一个`2`.

### 7. 配置ZK的配置文件zoo.cfg
接下来进入zk的conf目录.这个目录中应该有一个`zoo_sample.cfg`的文件.拷贝这个文件,并重命名为`zoo.cfg`

```bash
$ cp zoo_sample.cfg zoo.cfg
```

### 8. 修改zoo.cfg文件的内容
配置的内容如下:

```bash
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/usr/local/zookeeper/data
dataLogDir=/usr/local/zookeeper/datalog
# the port at which the clients will connect
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1

minSessionTimeout=1000
maxSessionTimeout=1500

server.1=192.168.1.100:2888:3888  
server.2=192.168.1.101:2888:3888  
server.3=192.168.1.102:2888:3888
```

**参数说明:**  
`tickTime`：zookeeper中使用的基本时间单位, 毫秒值.

`initLimit`: zookeeper集群中的包含多台server, 其中一台为leader, 集群中其余的server为follower。 initLimit参数配置初始化连接时, follower和leader之间的最长心跳时间. 此时该参数设置为5, 说明时间限制为5倍tickTime, 即5*2000=10000ms=10s.

`syncLimit`: 该参数配置leader和follower之间发送消息, 请求和应答的最大时间长度. 此时该参数设置为2, 说明时间限制为2倍tickTime, 即4000ms.

`dataDir`: 数据存放目录. 可以是任意目录.但是我喜欢这么干

`dataLogDir`: log目录, 同样可以是任意目录. 如果没有设置该参数, 将使用和dataDir相同的设置

`clientPort`: 监听client连接的端口号.

`server.X=A:B:C`: 其中X是一个数字, 表示这是第几号server. A是该server所在的IP地址. B配置该server和集群中的leader交换消息所使用的端口. C配置选举leader时所使用的端口.

`minSessionTimeout`: 最小的会话超时时间,这两个参数可以适当的调低一点,否则如果一个临时状态节点挂掉以后,会有很长时间才会在ZK中体现出来.

`maxSessionTimeout `: 最大的会话超时时间

### 9. 启动Zookeeper
配置到这里就完了.其实ZK的集群配置非常的简单.
启动也非常的简单.直接在命令行中调用:

```bash
$ ./zkServer.sh start
```

第一台机器启动的时候可能会报异常,这是正常的.他启动后会尝试连接其他的两台机器.但是由于其他两台还没启动起来,所以会报错. 启动了就好了.

PS: 需要注意的是,ZK集群提供了过半存活的能力.也就是说2n+1台机器环境下的ZK集群.最多允许n个节点挂掉.超过了就无法选举出leader了.


---

Update: 2015-12-05 10:25:04

## 在Docker1.9下部署跨物理机的ZK集群.
现在虚拟化非常的流行.Docker的使用也越来越多了.因此,在我们的系统中也使用Docker来封装了各种服务,ZK也不例外.

要在Docker中搭建ZK集群,主要有两种方法.第一种方法就是使用`Docker1.9`的`network`.另外一种就是使用`--net=host`模式来实现.

### network方式实现
首先搭建好`Docker1.9`,并创建一个网络`my-net`.这方面可以参考我的博文[Docker1.9新特性-跨物理机的多容器网络连接](/2015/11/09/Docker1-9新特性-跨物理机的多容器网络连接/).
然后在宿主机上创建两个目录,一个是 `conf` 一个是 `data`. 在`data`文件夹中照样的创建`myid`文件,并写上号码. 在`conf`文件夹中修改`zoo.cfg`配置文件:

```bash
...
server.1=zk1:2888:3888
server.2=zk2:2888:3888
server.3=zk3:2888:3888
```

然后就可以启动docker容器了:

```bash
docker run --privileged=true -p 3181:2181 -v /root/zookeeper/1/conf:/opt/zookeeper/conf -v /root/zookeeper/1/data:/tmp/zookeeper --name zk1 --hostname zk1 --net my-net jplock/zookeeper

docker run --privileged=true -p 3281:2181 -v /root/zookeeper/2/conf:/opt/zookeeper/conf -v /root/zookeeper/2/data:/tmp/zookeeper --name zk2 --hostname zk2 --net my-net jplock/zookeeper

docker run --privileged=true -p 3381:2181 -v /root/zookeeper/3/conf:/opt/zookeeper/conf -v /root/zookeeper/3/data:/tmp/zookeeper --name zk3 --hostname zk3 --net my-net jplock/zookeeper
```

这样就相当于启动了3个ZK,并且建立起集群了.

```bash
[root@CentOS7_with_kernel4 ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                        PORTS                                        NAMES
bd3e7befe015        jplock/zookeeper    "/opt/zookeeper/bin/z"   23 minutes ago      Up 3 seconds                  2888/tcp, 3888/tcp, 0.0.0.0:3181->2181/tcp   zk1
b5dfc1977efd        jplock/zookeeper    "/opt/zookeeper/bin/z"   24 minutes ago      Up 2 seconds                  2888/tcp, 3888/tcp, 0.0.0.0:3281->2181/tcp   zk2
1e605a32885d        jplock/zookeeper    "/opt/zookeeper/bin/z"   24 minutes ago      Up 2 seconds                  2888/tcp, 3888/tcp, 0.0.0.0:3381->2181/tcp   zk3
```

### host方式实现
host模式要求docker1.5以上. 这种方式就是使用宿主机的网络直接当成容器的网络.因此,配置ZK集群也很简单.相当于是在多台真实的物理机上进行部署. 这种方式的缺点就是 一个宿主机上只能部署一个ZK.否则会端口冲突.

在宿主机上创建两个目录,一个是 `conf` 一个是 `data`. 在`data`文件夹中照样的创建`myid`文件,并写上号码. 在`conf`文件夹中修改`zoo.cfg`配置文件:

```bash
...
server.1=10.211.55.13:2888:3888
server.2=10.211.55.14:2888:3888
server.3=10.211.55.15:2888:3888
```

然后就可以启动docker容器了:

```bash
宿主机1:
docker run --privileged=true -v /root/zk/1/conf:/opt/zookeeper/conf -v /root/zk/1/data:/tmp/zookeeper --name z_k1 --net=host jplock/zookeeper

宿主机2:
docker run --privileged=true -v /root/zk/2/conf:/opt/zookeeper/conf -v /root/zk/2/data:/tmp/zookeeper --name z_k2 --net=host jplock/zookeeper

宿主机3:
docker run --privileged=true -v /root/zk/3/conf:/opt/zookeeper/conf -v /root/zk/3/data:/tmp/zookeeper --name z_k3 --net=host jplock/zookeeper
```

这样就相当于启动了3个ZK,并且建立起集群了.

```bash
[root@CentOS7_with_kernel4 ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                        PORTS                                        NAMES
11cb1b859fb7        jplock/zookeeper    "/opt/zookeeper/bin/z"   10 minutes ago      Up 10 minutes      							                               z_k1

[root@CentOS7_with_kernel4_2 ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                        PORTS                                        NAMES
1vxsf1859fb7        jplock/zookeeper    "/opt/zookeeper/bin/z"   10 minutes ago      Up 10 minutes  								                               z_k2

[root@CentOS7_with_kernel4_3 ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                        PORTS                                        NAMES
11cbbd9s7fb7        jplock/zookeeper    "/opt/zookeeper/bin/z"   10 minutes ago      Up 10 minutes                                                          z_k3
```


