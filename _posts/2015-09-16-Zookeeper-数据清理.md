title: Zookeeper 数据清理
date: 2015-09-16 23:31:16
tags:
- Zookeeper
- 集群
- 大数据
comments: true
toc: false
---

#Zookeeper 数据清理

##问题现象:
情况是这样的,我在自己的笔记本上安装了`Zookeeper`,并使用了默认的配置作为开发的时候使用.
因为平时会关机等等,所以重启`zookeeper`的次数也是比较频繁的.突然最近发现系统磁盘空间就不够用了.
这不查不知道,一查吓一跳,`zookeeper`的data数据文件夹占了大概**5个G**的空间,而我zookeeper里面其实也没有什么东西,怎么会这么大呢?于是有了这篇文章的诞生.

##问题分析:
首先我就在网上找了一圈,都说使用`zk/bin`里面的`zkClean.sh`就可以了.
其调用语法是:`./zkClean.sh -n 3` 这样就可以保留最近的三个Log文件.
但是我执行了后,发现然并卵.点反应都没有.一个文件都没有删除掉..
然后又找了一篇文章:[ZooKeepr日志清理](http://nileader.blog.51cto.com/1381108/932156).按照里面说的第一种方法,直接删除data文件夹里面的数据.只保留最近的3个.结果删除以后,数据都没了...说明这个方法里面肯定有什么猫腻是没有发现的.

<!--more-->

于是,仔细的分析`zkClean.sh`这个脚本. 发现它其实是调用的`org.apache.zookeeper.server.PurgeTxnLog`这个类中的方法来执行清理工作的.于是找到这个类的源文件.分析其中的代码:

```java
public static void purge(File dataDir, File snapDir, int num) throws IOException {
        if (num < 3) {
            throw new IllegalArgumentException("count should be greater than 3");
        }

        FileTxnSnapLog txnLog = new FileTxnSnapLog(dataDir, snapDir);
        
        // found any valid recent snapshots?
        
        // files to exclude from deletion
        Set<File> exc=new HashSet<File>();
        List<File> snaps = txnLog.findNRecentSnapshots(num);
        if (snaps.size() == 0) 
            return;
        File snapShot = snaps.get(snaps.size() -1);
        for (File f: snaps) {
            exc.add(f);
        }
        long zxid = Util.getZxidFromName(snapShot.getName(),"snapshot");
        exc.addAll(Arrays.asList(txnLog.getSnapshotLogs(zxid)));

        final Set<File> exclude=exc;
        class MyFileFilter implements FileFilter{
            private final String prefix;
            MyFileFilter(String prefix){
                this.prefix=prefix;
            }
            public boolean accept(File f){
                if(!f.getName().startsWith(prefix) || exclude.contains(f))
                    return false;
                return true;
            }
        }
        // add all non-excluded log files
        List<File> files=new ArrayList<File>(
                Arrays.asList(txnLog.getDataDir().listFiles(new MyFileFilter("log."))));
        // add all non-excluded snapshot files to the deletion list
        files.addAll(Arrays.asList(txnLog.getSnapDir().listFiles(new MyFileFilter("snapshot."))));
        // remove the old files
        for(File f: files)
        {
            System.out.println("Removing file: "+
                DateFormat.getDateTimeInstance().format(f.lastModified())+
                "\t"+f.getPath());
            if(!f.delete()){
                System.err.println("Failed to remove "+f.getPath());
            }
        }

    }
```
通过断点发现,在我这的环境中执行到`Util.getZxidFromName(snapShot.getName(),"snapshot");`这句的时候,返回的`zxid`为`-1`.也就是没有找到`snapshot`的文件.所以剩下的所有文件都被加入到了`exclude`这个排除删除的变量中.因此,最后需要删除的队列`files`里面的东西其实是空的,也就是什么文件都不删除.

为什么没有找到`snapshot`的文件呢?仔细看了下我这的`zookeeper/data`文件夹.其中确实没有名为`snapshot.*`的文件,全是`log.*`的文件.

翻看`zookeeper`的官方文档.里面其实写的很清楚了.
在`zookeeper`中, `dataDir`里面存储的是`snapshot`文件,也就是快照文件,作用就类似于我们数据库的`dump`文件.里面存放了所有的`zk`数据.
而`dataLogDir`文件夹里面存储的是`log`文件.也就是事务日志文件.每对zk中的数据做一次修改并保存.都会往这个文件中记录一条数据.
也就是说,当`zookeeper`启动的时候,它其实是通过一个`snapshot`快照文件加上一堆的`log`事务日志文件的叠加来还原zk中的数据的.

那么,`snapshot`文件是从哪儿来得呢?为什么我的机器上没有`snapshot`文件呢?
`zookeeper`的[官方文档](http://zookeeper.apache.org/doc/r3.3.3/zookeeperAdmin.html)中说的比较清楚,它有一个配置项叫做`snapCount`这个是用来控制生成快照文件的,默认值是`100000`.它的意思是当有`snapCount/2+rand.nextInt(snapCount/2)`条事务日志记入到`log`文件中的时候,那么就做一次快照.也就是在默认情况下五万至十万条事务的时候会做一次快照,之所以用随机数是为了避免在集群中,所有服务器在同一时间做快照的操作.
分析到这,原因就显而易见了.由于我是自己机器上做测试用的,量都比较小.半年了都还没有提交到五万次事务.所以它就一直不会生成快照文件,所以也就在清理日志文件的时候不会清理任何的`log`了.

除了这个问题,还有一个问题就是为什么会生成这么多的`log.*`文件.而且每一个文件都是64MB大小.
其实这个是因为`zookeeper`在开始写事务日志的时候,server每次都会新生成一个指定大小的`log`文件,本意是预分配空间,使磁盘空间更连续,最小化磁盘寻道的次数.
默认配置大小就是`64M`.因此,当系统每次重启的时候都会新生成一个`64M`大小的`log.*`文件了.完全没有写满就重启`zookeeper`.大大的浪费了空间.因此,我们需要调整这个大小.
控制这个大小的配置是`preAllocSize`,单位是`KB`.我们可以粗略的计算一下所需要的空间.比如我们五万次事务提交一次镜像.而每一次的事务大概是150个字节.那么总大小就应该是:`150*50000/1024/1024=7`,也就是7MB左右.

这样修改了这两个参数后,重新启动`zookeeper`.写一个简易的程序不断的对`zk`做写入的测试.结果证明了上面的分析.`data`文件夹中开始出现`snapshot`文件了.而且在不重启的情况下,`log`文件的数量也是在可控的范围内.这个时候再通过`zkClean.sh`就可以清除掉多余的日志文件了.

到此,问题解决.


##其他补充:
除了使用`zkClean.sh`文件手工的删除外.还可以在`zoo.cfg`文件中配置`autopurge.snapRetainCount`和`autopurge.purgeInterval`两个参数.
第一个参数指明了需要保留的快照数量,默认值和最小值是`3`.
第二个参数是两次清楚数据的时间间隔,单位是小时.如果设置为`0`,那么就不开启自动清理.

除此之外,建议把`data`和`dataLog`两个文件夹分开.不要存放到一起,因为`data`文件存放的其实是快照,对磁盘的性能要求不高.而`dataLog`文件夹里面存放的是事务日志文件,每执行一次事务,都会写数据到这个文件夹中去.因此对磁盘的要求是非常高的. 再者,他`org.apache.zookeeper.server.PurgeTxnLog`这个类的处理逻辑其实有一定的问题.首先它会把MacOS系统里面的`.DS_Store`文件计算进去,这个获取最新的文件的时候有可能是会出问题的.其二就是他在查找最新的几个文件的时候是没有单独的过滤`snapshot`文件的.如果没有区分`data`和`dataLog`文件夹,`snapshot`和`log`文件都混在一起,那么最后获取的处理就会找不到`snapshot`文件.那么又会照成什么都不能删除的情况.


##问题总结:
1. 修改配置中的`snapCount`,根据自己业务的需要,适当的修改这个数值.比如我们系统修改成`5000`比较合适.
2. 修改配置中的`preAllocSize`,根据自己业务的需要,适当的修改这个数值.比如我们系统修改成`1000`比较合适.
3. 修改配置中的`dataDir`和`dataLogDir`.分开存放事务日志和快照文件
4. 修改配置中的`autopurge.snapRetainCount`和`autopurge.purgeInterval`.增加自动删除多余日志的检测.
