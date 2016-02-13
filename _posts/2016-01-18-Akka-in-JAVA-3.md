---
title: Akka in JAVA(三)
date: 2016-01-18 21:23:13
tags:
- JAVA
- Akka
---

#Akka in JAVA(三)

上两个部分讲了`Akka`的基本知识和常见的用法.接下来讲一讲`Akka`的远程调用以及集群的使用.因为在现在的项目中,基本上都是分布式的,单个的应用程序都快成为"熊猫"了.因此`Akka`的远程以及集群调用就是非常有必要的了.

##Remote调用

`Akka-Remoting`是采用了P2P(peer-to-peer)的通信方式设计的,也就是端对端的方式.特别是Akka-Remoting不能与网络地址转换和负载均衡一起的工作.
但是,由于`Akka`在设计的时候就考虑了远程调用以及分布式的情况.因此,`Akka-Remoting`在使用上就非常的简单,几乎等于是透明的,和本地调用几乎相同.除了传递的消息需要可序列化以及创建和查找Actor的时候路径稍有不同外,没有其他的区别了.

###远程调用的准备
要在项目中使用`Akka-Remoting`非常的简单,只需要引入Maven中的`akka-remote`就可以了.

```xml
		<dependency>
            <groupId>com.typesafe.akka</groupId>
            <artifactId>akka-remote_2.11</artifactId>
            <version>2.4.1</version>
        </dependency>
```

<!--more-->
###配置
由于Akka几乎没有特别的为`Remoting`提供专门的API,区别仅仅在于配置.因此,接下来就是要修改项目中的akka的配置了:

```
WCMapReduceApp {

  akka {
    actor {
      provider = "akka.remote.RemoteActorRefProvider"
    }
    remote {
      enabled-transports = ["akka.remote.netty.tcp"]
      netty.tcp {
        hostname = "127.0.0.1"
        port = 2552		# 0表示自动选择一个可用的
      }
    }
  }
  
}
```
我们来看看这个配置和本地的有什么不同.
这个配置文件在`akka`的配置项中添加了一个`actor`配置项,并指定`provider`也就是Actor提供者为`akka.remote.RemoteActorRefProvider`,即远程Actor提供者.
然后定义了`remote`远程传输方式,使用`akka.remote.netty.tcp`即`netty`的方式提供服务,服务的IP和端口分别是`127.0.0.1`和`2552`,就这么简单.而由于是端对端的通信,因此客户端的配置和服务器端的是一样的.

以上只是远程调用的最小的配置,完整的可选配置如下:

```
akka {
 
  actor {
 
    # 序列化方式
    serializers {
      proto = "akka.serialization.ProtobufSerializer"
    }
 
    # 发布远程Akka
    deployment {
 
      default {
 
        # 手动指定Actor的远程地址
        # if this is set to a valid remote address, the named actor will be deployed
        # at that node e.g. "akka://sys@host:port"
        remote = ""
 
        # 目标
        target {
 
          # A list of hostnames and ports for instantiating the children of a
          # non-direct router
          #   The format should be on "akka://sys@host:port", where:
          #    - sys is the remote actor system name
          #    - hostname can be either hostname or IP address the remote actor
          #      should connect to
          #    - port should be the port for the remote server on the other node
          # The number of actor instances to be spawned is still taken from the
          # nr-of-instances setting as for local routers; the instances will be
          # distributed round-robin among the given nodes.
          nodes = []
 
        }
      }
    }
  }
 
  remote {
 
    # 传输方式
    # Which implementation of akka.remote.RemoteTransport to use
    # default is a TCP-based remote transport based on Netty
    transport = "akka.remote.netty.NettyRemoteTransport"
 
    # 受信模式
    # Enable untrusted mode for full security of server managed actors, allows
    # untrusted clients to connect.
    untrusted-mode = off
 
    # 集群检测超时时间
    # Timeout for ACK of cluster operations, like checking actor out etc.
    remote-daemon-ack-timeout = 30s
 
    # 是否记录接收的消息
    # If this is "on", Akka will log all inbound messages at DEBUG level, if off then they are not logged
    log-received-messages = off
 
    # 是否记录发送的消息
    # If this is "on", Akka will log all outbound messages at DEBUG level, if off then they are not logged
    log-sent-messages = off
 
    # Each property is annotated with (I) or (O) or (I&O), where I stands for “inbound” and O for “outbound” connections.
    # The NettyRemoteTransport always starts the server role to allow inbound connections, and it starts
    # active client connections whenever sending to a destination which is not yet connected; if configured
    # it reuses inbound connections for replies, which is called a passive client connection (i.e. from server
    # to client).
    netty {
 
      # 延迟阻塞超时时间
      # (O) In case of increased latency / overflow how long should we wait (blocking the sender)
      # until we deem the send to be cancelled?
      # 0 means "never backoff", any positive number will indicate time to block at most.
      backoff-timeout = 0ms
 
      # 加密cookie
      # (I&O) Generate your own with '$AKKA_HOME/scripts/generate_config_with_secure_cookie.sh'
      # or using 'akka.util.Crypt.generateSecureCookie'
      secure-cookie = ""
 
      # 是否需要cookie
      # (I) Should the remote server require that its peers share the same secure-cookie
      # (defined in the 'remote' section)?
      require-cookie = off
 
      # 使用被动连接
      # (I) Reuse inbound connections for outbound messages
      use-passive-connections = on
 
      # 域名,连接地址
      # (I) The hostname or ip to bind the remoting to,
      # InetAddress.getLocalHost.getHostAddress is used if empty
      hostname = ""
 
      # 端口
      # (I) The default remote server port clients should connect to.
      # Default is 2552 (AKKA), use 0 if you want a random available port
      port = 2552
 
      # 
      # (O) The address of a local network interface (IP Address) to bind to when creating
      # outbound connections. Set to "" or "auto" for automatic selection of local address.
      outbound-local-address = "auto"
 
      # 消息帧大小
      # (I&O) Increase this if you want to be able to send messages with large payloads
      message-frame-size = 1 MiB
 
      # 超时时间
      # (O) Timeout duration
      connection-timeout = 120s
 
      #连接备份日志大小
      # (I) Sets the size of the connection backlog
      backlog = 4096
 
      # 执行线程池存活时间
      # (I) Length in akka.time-unit how long core threads will be kept alive if idling
      execution-pool-keepalive = 60s
 
      # 执行线程池大小
      # (I) Size of the core pool of the remote execution unit
      execution-pool-size = 4
 
      #最大通道内存大小
      # (I) Maximum channel size, 0 for off
      max-channel-memory-size = 0b
 
      #总计通道内存大小
      # (I) Maximum total size of all channels, 0 for off
      max-total-memory-size = 0b
 
      #重试时间间隔
      # (O) Time between reconnect attempts for active clients
      reconnect-delay = 5s
 
      # 读取超时时间
      # (O) Read inactivity period (lowest resolution is seconds)
      # after which active client connection is shutdown;
      # will be re-established in case of new communication requests.
      # A value of 0 will turn this feature off
      read-timeout = 0s
 
      # 写入超时时间
      # (O) Write inactivity period (lowest resolution is seconds)
      # after which a heartbeat is sent across the wire.
      # A value of 0 will turn this feature off
      write-timeout = 10s
 
      # 所有超时时间
      # (O) Inactivity period of both reads and writes (lowest resolution is seconds)
      # after which active client connection is shutdown;
      # will be re-established in case of new communication requests
      # A value of 0 will turn this feature off
      all-timeout = 0s
 
      # 重连窗口时间
      # (O) Maximum time window that a client should try to reconnect for
      reconnection-time-window = 600s
    }
 
    # The dispatcher used for the system actor "network-event-sender"
    network-event-sender-dispatcher {
      executor = thread-pool-executor
      type = PinnedDispatcher
    }
  }
}
```

###创建远程Actor
通过上面的配置后,在程序里面创建远程的Actor就非常的简单了,基本上感觉不到是创建的远程Actor.

只需要在创建`ActorSystem`的时候使用上面所说的配置文件即可,接下来的就和本地的Actor没有任何的区别:

```java
ActorSystem system = ActorSystem.create("WCMapReduceApp", ConfigFactory.load("application")
                .getConfig("WCMapReduceApp"));
```
通过这个`actorSystem`创建出来的Actor的路径会是这样的:`akka.tcp://WCMapReduceApp@127.0.0.1:2552/user/remoteActor`,即是一个远程的Actor.

当然,除了直接在服务器端创建服务外,还能在客户端远程的要求服务器端创建一个Actor,并保持引用.

要实现这样的功能,同样需要修改配置文件:

```
akka {
  actor {
    deployment {
      /sampleActor {
        remote = "akka.tcp://sampleActorSystem@127.0.0.1:2553"
      }
    }
  }
}
```
这个配置文件告诉akka,在创建一个`/sampleActor`的时候,即`system.actorOf()`.指定的actor并不会在本地创建,而是会请求远程的actorSystem创建这个actor.


###查找远程Actor
当客户端发布了一个远程的Actor后,客户端就需要调用它.而向它发送消息的先决条件就是要找到这个Actor.

查询远程的Actor也非常的简单.每一个远程的Actor都会有一个它自己的Path.其格式是:`akka://<actorsystemname>@<hostname>:<port>/<actor path>`,比如上面所说的`akka.tcp://WCMapReduceApp@127.0.0.1:2552/user/remoteActor`.那么获取这个Actor的`ActorRef`,就是通过`actorFor`或`actorSelection`方法传入这个`ActorPath`即可.

```java
final ActorRef remoteActor = system.actorFor("akka.tcp://WCMapReduceApp@127.0.0.1:2552/user/WCMapReduceActor");
```
接下来的操作就和本地的Actor一模一样了.

###序列化
既然是远程调用,那么就涉及到消息的序列化.Akka内置了集中序列化的方式,也提供了序列化的扩展,你可以使用内置的序列化方式,也可以自己实现一个.

要选择使用何种序列化,需要修改配置文件.在`akka`一节中配置`serializers`选项,指定序列化的实现类:

```
akka {
  actor {
    serializers {
      java = "akka.serialization.JavaSerializer" #本地调用的默认方式
      proto = "akka.remote.serialization.ProtobufSerializer" #远程调用的默认方式
      myown = "cn.sunxiang0918.akka.demo5.CustomSerializer"
    }
  }
}
```
配置好这个后,还可以绑定数据类型与序列化方式之间的映射:

```
akka {
    actor {
      serialization-bindings {
        "java.lang.String" = java
        "akka.docs.serialization.Customer" = java
        "com.google.protobuf.Message" = proto
        "akka.docs.serialization.MyOwnSerializable" = myown
        "java.lang.Boolean" = myown
      }
    }
  }
```
上面这段代码就指定了各种数据类型分别采用不同的序列化方式.

如果觉得Akka内置的序列化方式不满足你的要求,可以自定义一个序列化类(通常并不需要).
这个也比较的简单,需要自定义的序列化类继承自`JSerializer`类.然后实现其中的方法:

```java
import akka.actor.*;
import akka.serialization.*;
import com.typesafe.config.*;
 
    public static class MyOwnSerializer extends JSerializer {
 
      // This is whether "fromBinary" requires a "clazz" or not
      @Override public boolean includeManifest() {
          return false;
      }
 
      // Pick a unique identifier for your Serializer,
      // you've got a couple of billions to choose from,
      // 0 - 16 is reserved by Akka itself
      @Override public int identifier() {
          return 1234567;
      }
 
      // "toBinary" serializes the given object to an Array of Bytes
      @Override public byte[] toBinary(Object obj) {
        //序列化方法
      }
 
      // "fromBinary" deserializes the given array,
      // using the type hint (if any, see "includeManifest" above)
      @Override public Object fromBinaryJava(byte[] bytes,
                     Class<?> clazz) {
        //反序列化方法
      }
    }
```

###远程Actor的路由
由于`Akka-remoting`是基于点对点的.因此,并不能很好的使用网络提供的负载均衡等功能.其实,要解决这个问题,我们可以使用`Akka`的路由功能.即在配置远程Actor的时候,增加`router`的参数.

```java
akka.actor.deployment {
  /parent/remotePool {
    router = round-robin-pool
    nr-of-instances = 10
    target.nodes = ["akka.tcp://app@10.0.0.2:2552", "akka://app@10.0.0.3:2552"]
  }
}
```
那么当向这个`/parent/remotePool`发送消息的时候,会轮询的把消息分发到不同的远程服务上,从而实现了高可用和负载均衡.

###远程事件
同Akka的本地Actor的生命周期Hook相同,Akka为远程的Actor申明了很多的事件.我们可以监听这些远程调用中发生的事件,也可以订阅这些事件.只需要在`ActorSystem.eventStream`中为下面的事件增加注册监听器即可. 需要注意的是如果要订阅任意的远程事件,是订阅`RemotingLifecycleEvent`,如果只订阅涉及链接的生命周期,需要订阅`akka.remote.AssociationEvent`.

* DisassociatedEvent : 链接结束事件,这个事件包含了链接方向以及参与方的地址.
* AssociatedEvent : 链接成功建立事件,这个事件包含了链接方向以及参与方的地址.
* AssociationErrorEvent : 链接相关错误事件,这个事件包含了链接方向以及参与方的地址以及错误的原因.
* RemotingListenEvent : 远程子系统准备好接受链接时的事件,这个事件包含了链接方向以及参与方的地址.
* RemotingShutdownEvent :	远程子系统被关闭的事件
* RemotingErrorEvent : 	远程相关的所有错误

###Demo

这里举一个稍微复杂点的例子----单词计数.这个是Hadoop的入门的例子,我们使用Akka配合远程调用来实现一次. 功能是这样的,服务端提供了map/reduce方式的单词数量的计算功能,而服务端提供了文本内容的.
它大概的运行流程是这样的:
![](/img/2016/01/18/1.png)

1. 首先客户端的FileReadActor从文本文件中读取文件.
2. 然后通过ClientActor发送给远端的服务端.
3. 服务端通过WCMapReduceActor接受客户端发送的消息,并发消息放入优先级MailBox
4. WCMapReduceActor把接收到的文本内容分发给MapActor做Map计算
5. Map计算把结果都发送给ReduceActor,做汇总reduce计算.
6. 最后AggregateActor把计算的结果显示出来.

我们先来看客户端:

**FileReadActor.java**

```java
public class FileReadActor extends UntypedActor {

    @Override
    public void onReceive(Object message) throws Exception {
        if (message instanceof String) {
            /*如果消息是String类型的*/
            String fileName = (String) message;
            try {
                BufferedReader reader = new BufferedReader(
                        new InputStreamReader(Thread.currentThread().getContextClassLoader().getResource(fileName).openStream()));
                String line;
                while ((line = reader.readLine()) != null) {
                    /*遍历,一行一个消息反馈给消息发送方*/
                    getSender().tell(line,null);
                }
                System.out.println("All lines send !");
                /*发送一个结束标识*/
                getSender().tell(String.valueOf("EOF"),null);
            } catch (IOException x) {
                System.err.format("IOException: %s%n", x);
            }
        } else {
            throw new IllegalArgumentException("Unknown message [" + message + "]");
        }
    }
}
```
通过InputStreamReader把文本内容一行一个的发送给Sender.

**ClientActor.java**

```java
public class ClientActor extends UntypedActor {

    private ActorRef remoteServer = null;
    private long start;

    /**
     * @param inRemoteServer
     */
    public ClientActor(ActorRef inRemoteServer) {
        remoteServer = inRemoteServer;
    }

    @Override
    public void onReceive(Object message) throws Exception {
        /*如果接收到的任务是String的,那么就直接发送给remoteServer这个Actor*/
        if (message instanceof String) {
            String msg = (String) message;
            if (message.equals("EOF")){
                //这个的Sender设置为自己是为了接收聚合完成的消息
                remoteServer.tell(msg, getSelf());
            }else{
                remoteServer.tell(msg, null);
            }
        }else if (message instanceof Boolean) {
            System.out.println("聚合完成");
            //聚合完成后发送显示结果的消息
            remoteServer.tell("DISPLAY_LIST",null);

            //执行完毕,关机
            getContext().stop(self());
        }
    }

    @Override
    public void preStart() {
        /*记录开始时间*/
        start = System.currentTimeMillis();
    }

    @Override
    public void postStop() {
        /*计算用时*/
        // tell the world that the calculation is complete
        long timeSpent = (System.currentTimeMillis() - start);
        System.out
                .println(String
                        .format("\n\tClientActor estimate: \t\t\n\tCalculation time: \t%s MS",
                                timeSpent));
    }
}
```
这个Actor重载了`preStart()`和`postStop()`方法用以记录性能.

**ClientMain.java**

```java
public class ClientMain {

    public static void main(String[] args) throws Exception {

        //文件名
        final String fileName = "Othello.txt";

        /*根据配置,找到System*/
        ActorSystem system = ActorSystem.create("ClientApplication", ConfigFactory.load("client").getConfig("WCMapReduceClientApp"));

        /*实例化远程Actor*/
        final ActorRef remoteActor = system.actorFor("akka.tcp://WCMapReduceApp@127.0.0.1:2552/user/WCMapReduceActor");

        /*实例化Actor的管道*/
        final ActorRef fileReadActor = system.actorOf(Props.create(FileReadActor.class));
        
        /*实例化Client的Actor管道*/
        final ActorRef clientActor = system.actorOf(Props.create(ClientActor.class,remoteActor));
        
        /*发送文件名给fileReadActor.设置sender或者说回调的Actor为clientActor*/
        fileReadActor.tell(fileName,clientActor);

    }
}
```
这个类里面就使用了远程Actor的查找方法,通过`system.actorFor("akka.tcp://WCMapReduceApp@127.0.0.1:2552/user/WCMapReduceActor");`获取到了远程的Actor

**client.conf**

```java
WCMapReduceClientApp {
  include "common"
  akka {
    actor {
      provider = "akka.remote.RemoteActorRefProvider"
    }
    remote {
      enabled-transports = ["akka.remote.netty.tcp"]
      netty.tcp {
        hostname = "127.0.0.1"
        port = 2553   //0表示自动选择一个可用的
      }
    }
  }
}
```

接下来我们来看看服务器端:

**MyPriorityMailBox.java**

```java
public class MyPriorityMailBox extends UnboundedPriorityMailbox {

    /**
     * 创建一个自定义优先级的无边界的邮箱. 用来规定命令的优先级. 这个就保证了DISPLAY_LIST 这个事件是最后再来处理.
     */
    public MyPriorityMailBox(ActorSystem.Settings settings, Config config) {

        // Creating a new PriorityGenerator,
        super(new PriorityGenerator() {
            @Override
            public int gen(Object message) {
                if (message.equals("DISPLAY_LIST"))
                    return 2; // 'DisplayList messages should be treated
                    // last if possible
                else if (message.equals(PoisonPill.getInstance()))
                    return 3; // PoisonPill when no other left
                else
                    return 0; // By default they go with high priority
            }
        });
    }

}
```
通过这个类,自定了一个无边界的优先级邮箱,这样做的目的是保证`DISPLAY_LIST`命令最后的响应.否则会出现文本内容还没有发送完成的情况下,就进行了结果的统计显示了.要使用这个自定义的优先级邮箱,需要在配置文件中进行配置:
**server.conf**

```java
WCMapReduceApp {
  include "common"
  akka {
    actor {
      provider = "akka.remote.RemoteActorRefProvider"
    }
    remote {
      enabled-transports = ["akka.remote.netty.tcp"]
      netty.tcp {
        hostname = "127.0.0.1"
        port = 2552
      }
    }
  }
  
  priorityMailBox-dispatcher {
    mailbox-type = "cn.sunxiang0918.akka.demo2.server.MyPriorityMailBox"
  }
}
```

**MapActor.java**

```java
public class MapActor extends UntypedActor {

    //停用词
    String[] STOP_WORDS = {"a", "about", "above", "above", "across", "after",
            "afterwards", "again", "against", "all", "almost", "alone",
            "along", "already", "also", "although", "always", "am", "among",
            "amongst", "amoungst", "amount", "an", "and", "another", "any",
            "anyhow", "anyone", "anything", "anyway", "anywhere", "are",
            "around", "as", "at", "back", "be", "became", "because", "become",
            "becomes", "becoming", "been", "before", "beforehand", "behind",
            "being", "below", "beside", "besides", "between", "beyond", "bill",
            "both", "bottom", "but", "by", "call", "can", "cannot", "cant",
            "co", "con", "could", "couldnt", "cry", "de", "describe", "detail",
            "do", "done", "down", "due", "during", "each", "eg", "eight",
            "either", "eleven", "else", "elsewhere", "empty", "enough", "etc",
            "even", "ever", "every", "everyone", "everything", "everywhere",
            "except", "few", "fifteen", "fify", "fill", "find", "fire",
            "first", "five", "for", "former", "formerly", "forty", "found",
            "four", "from", "front", "full", "further", "get", "give", "go",
            "had", "has", "hasnt", "have", "he", "hence", "her", "here",
            "hereafter", "hereby", "herein", "hereupon", "hers", "herself",
            "him", "himself", "his", "how", "however", "hundred", "ie", "if",
            "in", "inc", "indeed", "interest", "into", "is", "it", "its",
            "itself", "keep", "last", "latter", "latterly", "least", "less",
            "ltd", "made", "many", "may", "me", "meanwhile", "might", "mill",
            "mine", "more", "moreover", "most", "mostly", "move", "much",
            "must", "my", "myself", "name", "namely", "neither", "never",
            "nevertheless", "next", "nine", "no", "nobody", "none", "noone",
            "nor", "not", "nothing", "now", "nowhere", "of", "off", "often",
            "on", "once", "one", "only", "onto", "or", "other", "others",
            "otherwise", "our", "ours", "ourselves", "out", "over", "own",
            "part", "per", "perhaps", "please", "put", "rather", "re", "same",
            "see", "seem", "seemed", "seeming", "seems", "serious", "several",
            "she", "should", "show", "side", "since", "sincere", "six",
            "sixty", "so", "some", "somehow", "someone", "something",
            "sometime", "sometimes", "somewhere", "still", "such", "system",
            "take", "ten", "than", "that", "the", "their", "them",
            "themselves", "then", "thence", "there", "thereafter", "thereby",
            "therefore", "therein", "thereupon", "these", "they", "thickv",
            "thin", "third", "this", "those", "though", "three", "through",
            "throughout", "thru", "thus", "to", "together", "too", "top",
            "toward", "towards", "twelve", "twenty", "two", "un", "under",
            "until", "up", "upon", "us", "very", "via", "was", "we", "well",
            "were", "what", "whatever", "when", "whence", "whenever", "where",
            "whereafter", "whereas", "whereby", "wherein", "whereupon",
            "wherever", "whether", "which", "while", "whither", "who",
            "whoever", "whole", "whom", "whose", "why", "will", "with",
            "within", "without", "would", "yet", "you", "your", "yours",
            "yourself", "yourselves", "the"};

    List<String> STOP_WORDS_LIST = Arrays.asList(STOP_WORDS);

    /*reduce聚合的Actor*/
    private ActorRef actor = null;

    public MapActor(ActorRef inReduceActor) {
        actor = inReduceActor;
    }

    @Override
    public void preStart() throws Exception {
        System.out.println("启动MapActor:"+Thread.currentThread().getName());
    }
    
    /**
     * 用于分词 计算单词的数量的
     * @param line
     * @return
     */
    private List<Result> evaluateExpression(String line) {
        List<Result> list = new ArrayList<>();
        /*字符串分词器*/
        StringTokenizer parser = new StringTokenizer(line);
        while (parser.hasMoreTokens()) {
            /*如果是,那么就判断是否是字母.然后把结果记录下来*/
            String word = parser.nextToken().toLowerCase();
            if (isAlpha(word) && !STOP_WORDS_LIST.contains(word)) {
                list.add(new Result(word, 1));
            }
        }
        return list;
    }

    /**
     * 判断是否是字母
     * @param s
     * @return
     */
    private boolean isAlpha(String s) {
        s = s.toUpperCase();
        for (int i = 0; i < s.length(); i++) {
            int c = (int) s.charAt(i);
            if (c < 65 || c > 90)
                return false;
        }
        return true;
    }

    @Override
    public void onReceive(Object message) throws Exception {
        if (message instanceof String) {
            String work = (String) message;
            
            if (work.equals("EOF")){
                /*表示已经结束了*/
                actor.tell(true,null);
                return;
            }
            
            // 计算这一行的单词情况
            List<Result> list = evaluateExpression(work);

            // 把这一行的单词情况发送给汇总的ReduceActor
            actor.tell(list, null);
        } else
            throw new IllegalArgumentException("Unknown message [" + message + "]");
    }
}
```
当这个actor接收到消息后,判断是否是结束标识,如果是就发送消息给reduceActor表示已经结束了.否则就计算这一行中的单词的个数,并把这个个数发送给reduceActor.

**ReduceActor.java**

```java
public class ReduceActor extends UntypedActor {

    /*管道Actor*/
    private ActorRef actor = null;

    public ReduceActor(ActorRef inAggregateActor) {
        actor = inAggregateActor;
    }

    @Override
    public void preStart() throws Exception {
        System.out.println("启动ReduceActor:"+Thread.currentThread().getName());
    }
    
    @Override
    public void onReceive(Object message) throws Exception {
        if (message instanceof List) {

            /*强制转换结果*/
            List<Result> work = (List<Result>) message;

            // 第一次汇总单词表结果.
            NavigableMap<String, Integer> reducedList = reduce(work);

            // 把这次汇总的结果发送给最终的结果聚合Actor
            actor.tell(reducedList, null);

        }else if (message instanceof Boolean) {
            //表示已经计算结束了
            // 把这次汇总的结果发送给最终的结果聚合Actor
            actor.tell(message, null);
        } else
            throw new IllegalArgumentException("Unknown message [" + message + "]");
    }

    /**
     * 聚合计算本次结果中各个单词的出现次数
     * @param list
     * @return
     */
    private NavigableMap<String, Integer> reduce(List<Result> list) {

        NavigableMap<String, Integer> reducedMap = new ConcurrentSkipListMap<>();

        for (Result result : list) {
            /*遍历结果,如果在这个小的结果中已经存在相同的单词了,那么数量+1,否则新建*/
            if (reducedMap.containsKey(result.getWord())) {
                Integer value = reducedMap.get(result.getWord());
                value++;
                reducedMap.put(result.getWord(), value);
            } else {
                reducedMap.put(result.getWord(), 1);
            }
        }
        return reducedMap;
    }
}
```
这个Actor接收到消息后,判断是什么消息,如果是结果消息,那么就对结果进行整理,得出某个单词出现的次数,否则就是结束标记,告诉管道Actor统计已经结束.

**AggregateActor.java**

```java
public class AggregateActor extends UntypedActor {

    /*最终的结果*/
    private Map<String, Integer> finalReducedMap = new HashMap<>();

    @Override
    public void preStart() throws Exception {
        System.out.println("启动AggregateActor:"+Thread.currentThread().getName());
    }

    @Override
    public void onReceive(Object message) throws Exception {
        /*如果是Map,那么就进行reduce操作*/
        if (message instanceof Map) {
            Map<String, Integer> reducedList = (Map<String, Integer>) message;
            aggregateInMemoryReduce(reducedList);
        } else if (message instanceof String) {
            /*如果是String,那么就是打印结果*/
            if (((String) message).compareTo("DISPLAY_LIST") == 0) {
                //getSender().tell(finalReducedMap.toString());
                System.out.println(finalReducedMap.toString());
               
            }
        }else if (message instanceof Boolean) {
            /*向客户端发送已经reduce完成的信息*/
            getSender().tell(true,null);
        }
    }

    private void aggregateInMemoryReduce(Map<String, Integer> reducedList) {

        for (String key : reducedList.keySet()) {
            /*最终的数量的累加*/
            if (finalReducedMap.containsKey(key)) {
                Integer count = reducedList.get(key) + finalReducedMap.get(key);
                finalReducedMap.put(key, count);
            } else {
                finalReducedMap.put(key, reducedList.get(key));
            }

        }
    }

}
```
这个actor接收到消息后,判断消息的类型,如果是`DISPLAY_LIST`标识,那么就打印结果.如果是`Boolean`就表示统计完成了,那么就发送消息给客户端.如果是`Map`,那么这个就是某一个map/reduce的结果,那么就把这个结果聚合到最终的结果中去.

**WCMapReduceActor.java**

```java
public class WCMapReduceActor extends UntypedActor{

    private ActorRef mapRouter;
    private ActorRef aggregateActor;

    @Override
    public void preStart() throws Exception {
        System.out.println("启动WCMapReduceActor:"+Thread.currentThread().getName());
    }
    
    @Override
    public void onReceive(Object message) throws Exception {
        if (message instanceof String) {
            /*如果接收到的是显示结果的请求,那么就调用reduce的Actor*/
            if (((String) message).compareTo("DISPLAY_LIST") == 0) {
                System.out.println("Got Display Message");
                aggregateActor.tell(message, getSender());
            }if (message.equals("EOF")){
                //表示发送完毕
                aggregateActor.tell(true, getSender());
            }else {
                /*否则给map的Actor进行计算*/
                mapRouter.tell(message,null);
            }
        }
    }

    public WCMapReduceActor(ActorRef inAggregateActor, ActorRef inMapRouter) {
        mapRouter = inMapRouter;
        aggregateActor = inAggregateActor;
    }
}
```
消息中转和统管的Actor,它统管了其他的几个Actor,是消息的入口.

**WCMapReduceServer.java**

```java
public class WCMapReduceServer{

    private ActorRef mapRouter;
    private ActorRef reduceRouter;
    private ActorRef aggregateActor;
    private ActorRef wcMapReduceActor;

    public WCMapReduceServer(int no_of_reduce_workers, int no_of_map_workers) {
        /*创建了Actor系统*/
        ActorSystem system = ActorSystem.create("WCMapReduceApp", ConfigFactory.load("application")
                .getConfig("WCMapReduceApp"));

        // 创建聚合Actor
        aggregateActor = system.actorOf(Props.create(AggregateActor.class));

        // 创建多个聚合的Actor
        reduceRouter = system.actorOf(Props.create(ReduceActor.class,aggregateActor).withRouter(new RoundRobinPool(no_of_reduce_workers)));

        // 创建多个Map的Actor
        mapRouter = system.actorOf(Props.create(MapActor.class,reduceRouter).withRouter(new RoundRobinPool(no_of_map_workers)));

        // create the overall WCMapReduce Actor that acts as the remote actor
        // for clients
        Props props = Props.create(WCMapReduceActor.class,aggregateActor,mapRouter).withDispatcher("priorityMailBox-dispatcher");
        wcMapReduceActor = system.actorOf(props, "WCMapReduceActor");
    }

    /**
     * @param args
     */
    public static void main(String[] args) {
        new WCMapReduceServer(50, 50);
    }

}
```
服务端的入口程序,定义了一个50个actor的单词统计服务.并使用轮询模式来分发客服端接收到的统计任务.

以上就是整个DEMO的所有的代码.当执行这个程序后,会在控制台打印:

服务器端:

```bash
Got Display Message
{minerals=5, half=35, exceeding=5, spoke=25, mince=5, hall=5, disproportion=5, youth=20, guards=5, wreck=5, begins=10, approved=20, imperfect=5, drunk=15, framed=10, pick=5,......}
```

客户端:

```bash
All lines send !
聚合完成
	ClientActor estimate: 		
	Calculation time: 	467 MS
```
到此,我们就实现了通过`AKKA-remoting` 来进行`map/reduce`的简单计算.

##Cluster调用

###原理

Akka除了remoting远程调用外,还提供了支持去中心化的基于P2P的集群服务,并且不会出现单点故障.Akka的集群是基于Gossip协议实现的,支持服务自动失效检测,能够自动发现出现问题而离开集群的成员节点,通过事件驱动的方式,将状态传播到整个集群的其他成员节点中去.Gossip协议是点对点通信协议的一种,它受社交网络中的流言传播的特点所启发,解决了在超大规模集群下其他方式无法解决的单点等问题.

一个Akka集群是由一组成员节点组成的,每一个成员节点都是通过`hostname:port:uid`来唯一标识,并且每一个成员节点间是完全解耦合的.

####节点状态

Akka集群内部为集群中的成员定义了6种状态,并提供了状态转换矩阵,这6种状态分别是:

* Joining : 正在加入集群的状态
* Up :	正常提供服务的状态
* Leaving :	正在离开集群的状态
* Down :	节点服务下线的状态
* Exiting :	节点离开状态
* Removed :	节点被删除状态

![](/img/2016/01/18/2.png)

在Akka集群中的每一个成员节点,都只可能处在这6种状态中的一种中.当节点状态发生变化的时候,会发出节点状态事件.需要注意的是,除了Down和Removed状态外,其他状态是有可能随时变为Down状态的,即节点故障而无法提供服务.处于Down状态的节点如果想要再次加入Akka集群中,需要重新启动,并加入Joining状态,然后才能进行后续状态的变化,加入集群.

####故障监控

在Akka集群中,集群的每一个成员节点,都会被其他另外一组节点(默认是5个)所监控,这一组节点会通过心跳来检测被监控的节点是否处于Unreachable状态,如果不可达则这一组节点会将被监控的节点的Unreachable状态向集群中的其他所有节点传播,最终使集群中的每个成员节点都知道被监控的节点已经故障.

Akka集群中任一一个成员节点都有可能成为集群的Leader,这是基于Gossip协议收敛过程得到的确定性结果,并不是通过选举产生,从而避免了单点故障.在Akka集群中,Leader只是一种角色,在各轮Gossip收敛过程中Leader可能是不断变化的.Leader的职责就是让成员节点加入和离开集群.一个成员节点最开始是处于Joining状态,一旦所有其他节点都看到了新加入的该节点,则Leader会设置这个节点的状态为up.如果一个节点安全离开Akka集群,那么这个节点的状态会变为Leaving状态,当Leader看到该节点为Leaving状态,会将其状态修改为Exiting,然后通知所有其他节点,当所有节点看到该节点状态为exiting后,Leader将该节点移除,状态修改为removed状态.

###配置

要在项目中使用Akka集群,首先需要的就是在项目的Maven中引入akka-Cluster:

```xml
		<dependency>
            <groupId>com.typesafe.akka</groupId>
            <artifactId>akka-cluster_2.11</artifactId>
            <version>2.4.1</version>
        </dependency>
```

然后在`application.conf`中配置必要的参数:

```
akka {
  actor {
   #表示Actor的提供者是Cluster集群
    provider = "akka.cluster.ClusterActorRefProvider"
  }
  #同akka-remoting,表示远程协议是什么
  remote {
    log-remote-lifecycle-events = off
    netty.tcp {
      hostname = "127.0.0.1"
      port = 0
    }
  }

  #集群特有的配置
  cluster {
    #种子节点
    seed-nodes = [
      "akka.tcp://ClusterSystem@127.0.0.1:2551",
      "akka.tcp://ClusterSystem@127.0.0.1:2552"]
    #自动down掉不可达的成员节点
    auto-down-unreachable-after = 10s
  }
}
```

上面的配置需要特别注意的就是`种子节点`,种子节点是akka集群中的一种特殊的节点角色.
种子节点最主要的用处是提供Cluster的初始化和加入点,同时也为其他节点作为中间联系人.要启动Akka-Cluster就必须配置一些列的种子节点.这些种子节点就是一开始就能够预料到的节点,有节点加入的时候,会等种子节点的返回确认才算是加入成功.

更多的集群配置请参见:[config-akka-cluster](http://doc.akka.io/docs/akka/snapshot/general/configuration.html#config-akka-cluster)

###集群事件

正如上文所说,当节点发生变化的时候,Leader会发送状态的事件给集群中的所有成员节点.因此,接收和处理这些事件也是非常重要的.

* MemberUp : 成员节点上线
* MemberExited : 成员节点下线
* MemberRemoved : 成员节点被剔除
* UnreachableMember : 成员节点无法到达
* ReachableMember : 成员节点可到达
* LeaderChanged : Leader变化
* RoleLeaderChanged : 角色Leader变化
* ClusterShuttingDown : 集群关闭
* ClusterMetricsChanged : 

要说明这些节点的变化可以参考官方给出的最最简单的Akka集群的Demo,它不仅列出了一个最简单的Akka的集群要如何构建,也说明了这几个事件状态的变化.

####集群事件Demo
在官方的文档中,编写了一个最简单的Akka-Cluster的例子,这个例子就是启动三个Akka的节点,并且监听了节点的所有事件,接收到事件后,打印出来.

**demo6.conf**

```
akka {
  actor {
    provider = "akka.cluster.ClusterActorRefProvider"
  }
  remote {
    log-remote-lifecycle-events = off
    netty.tcp {
      hostname = "127.0.0.1"
      port = 0
    }
  }

  cluster {
    seed-nodes = [
      #先启动了两个种子节点,这样当有新的节点加入时,会把事件通知给这两个节点.
      "akka.tcp://ClusterSystem@127.0.0.1:2551",
      "akka.tcp://ClusterSystem@127.0.0.1:2552"]

    auto-down-unreachable-after = 10s
  }
}
```

**SimpleClusterListener.java**

```java
public class SimpleClusterListener extends UntypedActor {
    /*记录日志*/
    LoggingAdapter log = Logging.getLogger(getContext().system(), this);
    
    /*创建,获取集群*/
    Cluster cluster = Cluster.get(getContext().system());

    //订阅集群中的事件
    @Override
    public void preStart() {
        //region subscribe
        cluster.subscribe(getSelf(), ClusterEvent.initialStateAsEvents(),
                MemberEvent.class, UnreachableMember.class);
        //endregion
    }

    //re-subscribe when restart
    @Override
    public void postStop() {
        cluster.unsubscribe(getSelf());
    }

    @Override
    public void onReceive(Object message) {
        /*当接收到不同的事件的时候,打印出不同的信息*/
        if (message instanceof MemberUp) {
            MemberUp mUp = (MemberUp) message;
            log.info("Member is Up: {}", mUp.member());

        } else if (message instanceof UnreachableMember) {
            UnreachableMember mUnreachable = (UnreachableMember) message;
            log.info("Member detected as unreachable: {}", mUnreachable.member());

        } else if (message instanceof MemberRemoved) {
            MemberRemoved mRemoved = (MemberRemoved) message;
            log.info("Member is Removed: {}", mRemoved.member());

        } else if (message instanceof MemberEvent) {
            // ignore
            log.info("Member Event: {}", ((MemberEvent) message).member());
        } else {
            unhandled(message);
        }

    }
}
```

**SimpleClusterApp.java**

```java
public class SimpleClusterApp {

    public static void main(String[] args) {
        if (args.length == 0)
            /*启动三个节点*/
            startup(new String[]{"2551", "2552", "0"});
        else
            startup(args);
    }

    public static void startup(String[] ports) {
        for (String port : ports) {
            // 重写配置中的远程端口
            Config config = ConfigFactory.parseString(
                    "akka.remote.netty.tcp.port=" + port).withFallback(
                    ConfigFactory.load("demo6"));

            // 创建ActorSystem,名称需要和conf配置文件中的相同
            ActorSystem system = ActorSystem.create("ClusterSystem", config);

            // 创建集群中的Actor,并监听事件
            system.actorOf(Props.create(SimpleClusterListener.class),
                    "clusterListener");

        }
    }
}
```

这个简单的例子启动了三个节点,并创建了一个Actor来监听集群中的各种事件,执行这个Demo,会在控制台打印:

```bash
[INFO] [02/11/2016 22:14:42.051] [main] [akka.remote.Remoting] Starting remoting
[INFO] [02/11/2016 22:14:42.347] [main] [akka.remote.Remoting] Remoting started; listening on addresses :[akka.tcp://ClusterSystem@127.0.0.1:2551]
[INFO] [02/11/2016 22:14:42.356] [main] [akka.cluster.Cluster(akka://ClusterSystem)] Cluster Node [akka.tcp://ClusterSystem@127.0.0.1:2551] - Starting up...
[INFO] [02/11/2016 22:14:42.425] [main] [akka.cluster.Cluster(akka://ClusterSystem)] Cluster Node [akka.tcp://ClusterSystem@127.0.0.1:2551] - Registered cluster JMX MBean [akka:type=Cluster]
[INFO] [02/11/2016 22:14:42.425] [main] [akka.cluster.Cluster(akka://ClusterSystem)] Cluster Node [akka.tcp://ClusterSystem@127.0.0.1:2551] - Started up successfully
[INFO] [02/11/2016 22:14:42.429] [ClusterSystem-akka.actor.default-dispatcher-14] [akka.cluster.Cluster(akka://ClusterSystem)] Cluster Node [akka.tcp://ClusterSystem@127.0.0.1:2551] - Metrics will be retreived from MBeans, and may be incorrect on some platforms. To increase metric accuracy add the 'sigar.jar' to the classpath and the appropriate platform-specific native libary to 'java.library.path'. Reason: java.lang.ClassNotFoundException: org.hyperic.sigar.Sigar
[INFO] [02/11/2016 22:14:42.432] [ClusterSystem-akka.actor.default-dispatcher-14] [akka.cluster.Cluster(akka://ClusterSystem)] Cluster Node [akka.tcp://ClusterSystem@127.0.0.1:2551] - Metrics collection has started successfully
[INFO] [02/11/2016 22:14:42.454] [main] [akka.remote.Remoting] Starting remoting
[INFO] [02/11/2016 22:14:42.460] [main] [akka.remote.Remoting] Remoting started; listening on addresses :[akka.tcp://ClusterSystem@127.0.0.1:2552]
[INFO] [02/11/2016 22:14:42.461] [main] [akka.cluster.Cluster(akka://ClusterSystem)] Cluster Node [akka.tcp://ClusterSystem@127.0.0.1:2552] - Starting up...
[INFO] [02/11/2016 22:14:42.464] [main] [akka.cluster.Cluster(akka://ClusterSystem)] Cluster Node [akka.tcp://ClusterSystem@127.0.0.1:2552] - Started up successfully
[INFO] [02/11/2016 22:14:42.464] [ClusterSystem-akka.actor.default-dispatcher-4] [akka.cluster.Cluster(akka://ClusterSystem)] Cluster Node [akka.tcp://ClusterSystem@127.0.0.1:2552] - Metrics will be retreived from MBeans, and may be incorrect on some platforms. To increase metric accuracy add the 'sigar.jar' to the classpath and the appropriate platform-specific native libary to 'java.library.path'. Reason: java.lang.ClassNotFoundException: org.hyperic.sigar.Sigar
[INFO] [02/11/2016 22:14:42.464] [ClusterSystem-akka.actor.default-dispatcher-4] [akka.cluster.Cluster(akka://ClusterSystem)] Cluster Node [akka.tcp://ClusterSystem@127.0.0.1:2552] - Metrics collection has started successfully
[INFO] [02/11/2016 22:14:42.486] [main] [akka.remote.Remoting] Starting remoting
[INFO] [02/11/2016 22:14:42.491] [main] [akka.remote.Remoting] Remoting started; listening on addresses :[akka.tcp://ClusterSystem@127.0.0.1:57726]
[INFO] [02/11/2016 22:14:42.492] [main] [akka.cluster.Cluster(akka://ClusterSystem)] Cluster Node [akka.tcp://ClusterSystem@127.0.0.1:57726] - Starting up...
[INFO] [02/11/2016 22:14:42.494] [main] [akka.cluster.Cluster(akka://ClusterSystem)] Cluster Node [akka.tcp://ClusterSystem@127.0.0.1:57726] - Started up successfully
[INFO] [02/11/2016 22:14:42.494] [ClusterSystem-akka.actor.default-dispatcher-3] [akka.cluster.Cluster(akka://ClusterSystem)] Cluster Node [akka.tcp://ClusterSystem@127.0.0.1:57726] - Metrics will be retreived from MBeans, and may be incorrect on some platforms. To increase metric accuracy add the 'sigar.jar' to the classpath and the appropriate platform-specific native libary to 'java.library.path'. Reason: java.lang.ClassNotFoundException: org.hyperic.sigar.Sigar
[INFO] [02/11/2016 22:14:42.494] [ClusterSystem-akka.actor.default-dispatcher-3] [akka.cluster.Cluster(akka://ClusterSystem)] Cluster Node [akka.tcp://ClusterSystem@127.0.0.1:57726] - Metrics collection has started successfully
[INFO] [02/11/2016 22:14:42.645] [ClusterSystem-akka.actor.default-dispatcher-4] [akka.cluster.Cluster(akka://ClusterSystem)] Cluster Node [akka.tcp://ClusterSystem@127.0.0.1:2551] - Node [akka.tcp://ClusterSystem@127.0.0.1:2551] is JOINING, roles []
[INFO] [02/11/2016 22:14:42.665] [ClusterSystem-akka.actor.default-dispatcher-4] [akka.cluster.Cluster(akka://ClusterSystem)] Cluster Node [akka.tcp://ClusterSystem@127.0.0.1:2551] - Leader is moving node [akka.tcp://ClusterSystem@127.0.0.1:2551] to [Up]
[INFO] [02/11/2016 22:14:42.672] [ClusterSystem-akka.actor.default-dispatcher-16] [akka://ClusterSystem/user/clusterListener] Member is Up: Member(address = akka.tcp://ClusterSystem@127.0.0.1:2551, status = Up)
[INFO] [02/11/2016 22:14:47.662] [ClusterSystem-akka.actor.default-dispatcher-3] [akka.cluster.Cluster(akka://ClusterSystem)] Cluster Node [akka.tcp://ClusterSystem@127.0.0.1:2551] - Node [akka.tcp://ClusterSystem@127.0.0.1:2552] is JOINING, roles []
[INFO] [02/11/2016 22:14:47.662] [ClusterSystem-akka.actor.default-dispatcher-3] [akka.cluster.Cluster(akka://ClusterSystem)] Cluster Node [akka.tcp://ClusterSystem@127.0.0.1:2551] - Node [akka.tcp://ClusterSystem@127.0.0.1:57726] is JOINING, roles []
[INFO] [02/11/2016 22:14:47.665] [ClusterSystem-akka.actor.default-dispatcher-18] [akka://ClusterSystem/user/clusterListener] Member Event: Member(address = akka.tcp://ClusterSystem@127.0.0.1:2552, status = Joining)
[INFO] [02/11/2016 22:14:47.666] [ClusterSystem-akka.actor.default-dispatcher-4] [akka://ClusterSystem/user/clusterListener] Member Event: Member(address = akka.tcp://ClusterSystem@127.0.0.1:57726, status = Joining)
[INFO] [02/11/2016 22:14:47.731] [ClusterSystem-akka.actor.default-dispatcher-21] [akka.cluster.Cluster(akka://ClusterSystem)] Cluster Node [akka.tcp://ClusterSystem@127.0.0.1:57726] - Welcome from [akka.tcp://ClusterSystem@127.0.0.1:2551]
[INFO] [02/11/2016 22:14:47.731] [ClusterSystem-akka.actor.default-dispatcher-17] [akka.cluster.Cluster(akka://ClusterSystem)] Cluster Node [akka.tcp://ClusterSystem@127.0.0.1:2552] - Welcome from [akka.tcp://ClusterSystem@127.0.0.1:2551]
[INFO] [02/11/2016 22:14:47.731] [ClusterSystem-akka.actor.default-dispatcher-2] [akka://ClusterSystem/user/clusterListener] Member is Up: Member(address = akka.tcp://ClusterSystem@127.0.0.1:2551, status = Up)
[INFO] [02/11/2016 22:14:47.732] [ClusterSystem-akka.actor.default-dispatcher-2] [akka://ClusterSystem/user/clusterListener] Member Event: Member(address = akka.tcp://ClusterSystem@127.0.0.1:2552, status = Joining)
[INFO] [02/11/2016 22:14:47.732] [ClusterSystem-akka.actor.default-dispatcher-2] [akka://ClusterSystem/user/clusterListener] Member Event: Member(address = akka.tcp://ClusterSystem@127.0.0.1:57726, status = Joining)
[INFO] [02/11/2016 22:14:47.733] [ClusterSystem-akka.actor.default-dispatcher-5] [akka://ClusterSystem/user/clusterListener] Member is Up: Member(address = akka.tcp://ClusterSystem@127.0.0.1:2551, status = Up)
[INFO] [02/11/2016 22:14:47.733] [ClusterSystem-akka.actor.default-dispatcher-5] [akka://ClusterSystem/user/clusterListener] Member Event: Member(address = akka.tcp://ClusterSystem@127.0.0.1:2552, status = Joining)
[INFO] [02/11/2016 22:14:47.745] [ClusterSystem-akka.actor.default-dispatcher-18] [akka://ClusterSystem/user/clusterListener] Member Event: Member(address = akka.tcp://ClusterSystem@127.0.0.1:57726, status = Joining)
[INFO] [02/11/2016 22:14:48.464] [ClusterSystem-akka.actor.default-dispatcher-15] [akka.cluster.Cluster(akka://ClusterSystem)] Cluster Node [akka.tcp://ClusterSystem@127.0.0.1:2551] - Leader is moving node [akka.tcp://ClusterSystem@127.0.0.1:2552] to [Up]
[INFO] [02/11/2016 22:14:48.464] [ClusterSystem-akka.actor.default-dispatcher-15] [akka.cluster.Cluster(akka://ClusterSystem)] Cluster Node [akka.tcp://ClusterSystem@127.0.0.1:2551] - Leader is moving node [akka.tcp://ClusterSystem@127.0.0.1:57726] to [Up]
[INFO] [02/11/2016 22:14:48.465] [ClusterSystem-akka.actor.default-dispatcher-15] [akka://ClusterSystem/user/clusterListener] Member is Up: Member(address = akka.tcp://ClusterSystem@127.0.0.1:2552, status = Up)
[INFO] [02/11/2016 22:14:48.465] [ClusterSystem-akka.actor.default-dispatcher-15] [akka://ClusterSystem/user/clusterListener] Member is Up: Member(address = akka.tcp://ClusterSystem@127.0.0.1:57726, status = Up)
[INFO] [02/11/2016 22:14:49.463] [ClusterSystem-akka.actor.default-dispatcher-16] [akka://ClusterSystem/user/clusterListener] Member is Up: Member(address = akka.tcp://ClusterSystem@127.0.0.1:2552, status = Up)
[INFO] [02/11/2016 22:14:49.463] [ClusterSystem-akka.actor.default-dispatcher-16] [akka://ClusterSystem/user/clusterListener] Member is Up: Member(address = akka.tcp://ClusterSystem@127.0.0.1:57726, status = Up)
[INFO] [02/11/2016 22:14:49.481] [ClusterSystem-akka.actor.default-dispatcher-21] [akka://ClusterSystem/user/clusterListener] Member is Up: Member(address = akka.tcp://ClusterSystem@127.0.0.1:2552, status = Up)
[INFO] [02/11/2016 22:14:49.481] [ClusterSystem-akka.actor.default-dispatcher-21] [akka://ClusterSystem/user/clusterListener] Member is Up: Member(address = akka.tcp://ClusterSystem@127.0.0.1:57726, status = Up)
```
从日志中就可以看出各种状态的变化

###集群实践

####阶乘服务Demo
这里通过一个简单的阶乘计算,来展示Akka-Cluster的使用.
这个Demo分为了前台和后台两个部分,前台只用来输入阶乘的大小以及打印计算的结果,后台节点负责真正的阶乘的计算.

**demo7.conf**

```
include "demo6"

# //#min-nr-of-members
akka.cluster.min-nr-of-members = 3
# //#min-nr-of-members

# //#role-min-nr-of-members
akka.cluster.role {
  frontend.min-nr-of-members = 1
  backend.min-nr-of-members = 2
}
# //#role-min-nr-of-members

# //#adaptive-router
akka.actor.deployment {
  /factorialFrontend/factorialBackendRouter = {
    router = adaptive-group
    # metrics-selector = heap
    # metrics-selector = load
    # metrics-selector = cpu
    metrics-selector = mix
    nr-of-instances = 100
    routees.paths = ["/user/factorialBackend"]
    cluster {
      enabled = on
      use-role = backend
      allow-local-routees = off
    }
  }
}
# //#adaptive-router
```

**FactorialResult.java**

```java
public class FactorialResult implements Serializable {
    public final int n;
    public final BigInteger factorial;

    FactorialResult(int n, BigInteger factorial) {
        this.n = n;
        this.factorial = factorial;
    }

    @Override
    public String toString() {
        return "FactorialResult{" +
                "n=" + n +
                ", factorial=" + factorial +
                '}';
    }
}
```

**FactorialBackend.java**

```java
public class FactorialBackend extends UntypedActor {

    @Override
    public void onReceive(Object message) {
        //如果是数字
        if (message instanceof Integer) {
            final Integer n = (Integer) message;
            /*使用akka的future功能,异步的计算阶乘*/
            Future<BigInteger> f = future(() -> factorial(n), getContext().dispatcher());

            /*合并计算的结果*/
            Future<FactorialResult> result = f.map(
                    new Mapper<BigInteger, FactorialResult>() {
                        public FactorialResult apply(BigInteger factorial) {
                            return new FactorialResult(n, factorial);
                        }
                    }, getContext().dispatcher());

            /*把结果返回Sender*/
            pipe(result, getContext().dispatcher()).to(getSender());

        } else {
            unhandled(message);
        }
    }

    /**
     * 进行阶乘计算
     * @param n
     * @return
     */
    BigInteger factorial(int n) {
        BigInteger acc = BigInteger.ONE;
        for (int i = 1; i <= n; ++i) {
            acc = acc.multiply(BigInteger.valueOf(i));
        }
        return acc;
    }
}
```

**FactorialBackendMain.java**

```java
public class FactorialBackendMain {

  public static void main(String[] args) {
    // 重写配置文件中的集群角色和端口
    final String port = args.length > 0 ? args[0] : "0";
    final Config config = ConfigFactory.parseString("akka.remote.netty.tcp.port=" + port).
      withFallback(ConfigFactory.parseString("akka.cluster.roles = [backend]")).
      withFallback(ConfigFactory.load("demo7"));

    ActorSystem system = ActorSystem.create("ClusterSystem", config);

    system.actorOf(Props.create(FactorialBackend.class), "factorialBackend");

  }

}
```

**FactorialFrontend.java**

```java
public class FactorialFrontend extends UntypedActor {
    final int upToN;        //计算到多少
    final boolean repeat;       //是否重复计算

    LoggingAdapter log = Logging.getLogger(getContext().system(), this);

    /*获取到Backend的Router*/
    ActorRef backend = getContext().actorOf(FromConfig.getInstance().props(),
            "factorialBackendRouter");

    public FactorialFrontend(int upToN, boolean repeat) {
        this.upToN = upToN;
        this.repeat = repeat;
    }

    @Override
    public void preStart() {
        //因为是在Start前就发送消息,所以必定超时.
        sendJobs();
        getContext().setReceiveTimeout(Duration.create(10, TimeUnit.SECONDS));
    }

    @Override
    public void onReceive(Object message) {
        if (message instanceof FactorialResult) {
            FactorialResult result = (FactorialResult) message;
            if (result.n == upToN) {
                System.out.println("计算的结果:" + result);
                if (repeat)
                    sendJobs();
                else
                    getContext().stop(getSelf());
            }

        } else if (message instanceof ReceiveTimeout) {
            log.info("Timeout");
            sendJobs();

        } else {
            unhandled(message);
        }
    }

    void sendJobs() {
        log.info("Starting batch of factorials up to [{}]", upToN);
        for (int n = 1; n <= upToN; n++) {
            backend.tell(n, getSelf());
        }
    }

}
```

**FactorialFrontendMain.java**

```java
public class FactorialFrontendMain {

  public static void main(String[] args) {
    final int upToN = 10;

    final Config config = ConfigFactory.parseString(
        "akka.cluster.roles = [frontend]").withFallback(
        ConfigFactory.load("demo7"));

    final ActorSystem system = ActorSystem.create("ClusterSystem", config);
    system.log().info(
        "Factorials will start when 2 backend members in the cluster.");
    //#registerOnUp
    Cluster.get(system).registerOnMemberUp((Runnable) () -> system.actorOf(Props.create(FactorialFrontend.class, upToN, false),
        "factorialFrontend"));
    //#registerOnUp
  }

}
```

**FactorialApp.java**

```java
public class FactorialApp {

  public static void main(String[] args) {
    // starting 3 backend nodes and 1 frontend node
    FactorialBackendMain.main(new String[] { "2551" });
    FactorialBackendMain.main(new String[] { "2552" });
    FactorialBackendMain.main(new String[0]);
    FactorialFrontendMain.main(new String[0]);
  }
}
```

通过执行`FactorialApp.java`可以启动整个演示. 它创建了3个后台节点和一个前台节点.
前台节点启动后,创建`FactorialFrontend`Actor,这个Actor负责发送计算数给后台节点,以及接受计算的结果并打印出来.后台节点的Actor`FactorialBackend`负责计算阶乘,并返回结果.

执行这个DEMO后,控制台会打印:

```bash
...
...
[INFO] [02/11/2016 22:45:04.580] [ClusterSystem-akka.actor.default-dispatcher-17] [akka://ClusterSystem/user/factorialFrontend] Starting batch of factorials up to [10]
[INFO] [02/11/2016 22:45:04.583] [ClusterSystem-akka.actor.default-dispatcher-2] [akka://ClusterSystem/deadLetters] Message [java.lang.Integer] from Actor[akka://ClusterSystem/user/factorialFrontend#1317512767] to Actor[akka://ClusterSystem/deadLetters] was not delivered. [1] dead letters encountered. This logging can be turned off or adjusted with configuration settings 'akka.log-dead-letters' and 'akka.log-dead-letters-during-shutdown'.
[INFO] [02/11/2016 22:45:04.583] [ClusterSystem-akka.actor.default-dispatcher-2] [akka://ClusterSystem/deadLetters] Message [java.lang.Integer] from Actor[akka://ClusterSystem/user/factorialFrontend#1317512767] to Actor[akka://ClusterSystem/deadLetters] was not delivered. [2] dead letters encountered. This logging can be turned off or adjusted with configuration settings 'akka.log-dead-letters' and 'akka.log-dead-letters-during-shutdown'.
[INFO] [02/11/2016 22:45:04.583] [ClusterSystem-akka.actor.default-dispatcher-2] [akka://ClusterSystem/deadLetters] Message [java.lang.Integer] from Actor[akka://ClusterSystem/user/factorialFrontend#1317512767] to Actor[akka://ClusterSystem/deadLetters] was not delivered. [3] dead letters encountered. This logging can be turned off or adjusted with configuration settings 'akka.log-dead-letters' and 'akka.log-dead-letters-during-shutdown'.
[INFO] [02/11/2016 22:45:04.583] [ClusterSystem-akka.actor.default-dispatcher-2] [akka://ClusterSystem/deadLetters] Message [java.lang.Integer] from Actor[akka://ClusterSystem/user/factorialFrontend#1317512767] to Actor[akka://ClusterSystem/deadLetters] was not delivered. [4] dead letters encountered. This logging can be turned off or adjusted with configuration settings 'akka.log-dead-letters' and 'akka.log-dead-letters-during-shutdown'.
[INFO] [02/11/2016 22:45:04.583] [ClusterSystem-akka.actor.default-dispatcher-2] [akka://ClusterSystem/deadLetters] Message [java.lang.Integer] from Actor[akka://ClusterSystem/user/factorialFrontend#1317512767] to Actor[akka://ClusterSystem/deadLetters] was not delivered. [5] dead letters encountered. This logging can be turned off or adjusted with configuration settings 'akka.log-dead-letters' and 'akka.log-dead-letters-during-shutdown'.
[INFO] [02/11/2016 22:45:04.583] [ClusterSystem-akka.actor.default-dispatcher-2] [akka://ClusterSystem/deadLetters] Message [java.lang.Integer] from Actor[akka://ClusterSystem/user/factorialFrontend#1317512767] to Actor[akka://ClusterSystem/deadLetters] was not delivered. [6] dead letters encountered. This logging can be turned off or adjusted with configuration settings 'akka.log-dead-letters' and 'akka.log-dead-letters-during-shutdown'.
[INFO] [02/11/2016 22:45:04.584] [ClusterSystem-akka.actor.default-dispatcher-2] [akka://ClusterSystem/deadLetters] Message [java.lang.Integer] from Actor[akka://ClusterSystem/user/factorialFrontend#1317512767] to Actor[akka://ClusterSystem/deadLetters] was not delivered. [7] dead letters encountered. This logging can be turned off or adjusted with configuration settings 'akka.log-dead-letters' and 'akka.log-dead-letters-during-shutdown'.
[INFO] [02/11/2016 22:45:04.584] [ClusterSystem-akka.actor.default-dispatcher-2] [akka://ClusterSystem/deadLetters] Message [java.lang.Integer] from Actor[akka://ClusterSystem/user/factorialFrontend#1317512767] to Actor[akka://ClusterSystem/deadLetters] was not delivered. [8] dead letters encountered. This logging can be turned off or adjusted with configuration settings 'akka.log-dead-letters' and 'akka.log-dead-letters-during-shutdown'.
[INFO] [02/11/2016 22:45:04.584] [ClusterSystem-akka.actor.default-dispatcher-2] [akka://ClusterSystem/deadLetters] Message [java.lang.Integer] from Actor[akka://ClusterSystem/user/factorialFrontend#1317512767] to Actor[akka://ClusterSystem/deadLetters] was not delivered. [9] dead letters encountered. This logging can be turned off or adjusted with configuration settings 'akka.log-dead-letters' and 'akka.log-dead-letters-during-shutdown'.
[INFO] [02/11/2016 22:45:04.584] [ClusterSystem-akka.actor.default-dispatcher-2] [akka://ClusterSystem/deadLetters] Message [java.lang.Integer] from Actor[akka://ClusterSystem/user/factorialFrontend#1317512767] to Actor[akka://ClusterSystem/deadLetters] was not delivered. [10] dead letters encountered, no more dead letters will be logged. This logging can be turned off or adjusted with configuration settings 'akka.log-dead-letters' and 'akka.log-dead-letters-during-shutdown'.
[INFO] [02/11/2016 22:45:14.600] [ClusterSystem-akka.actor.default-dispatcher-7] [akka://ClusterSystem/user/factorialFrontend] Timeout
[INFO] [02/11/2016 22:45:14.601] [ClusterSystem-akka.actor.default-dispatcher-7] [akka://ClusterSystem/user/factorialFrontend] Starting batch of factorials up to [10]
[WARN] [02/11/2016 22:45:14.607] [ClusterSystem-akka.remote.default-remote-dispatcher-8] [akka.serialization.Serialization(akka://ClusterSystem)] Using the default Java serializer for class [java.lang.Integer] which is not recommended because of performance implications. Use another serializer or disable this warning using the setting 'akka.actor.warn-about-java-serializer-usage'
[WARN] [02/11/2016 22:45:14.622] [ClusterSystem-akka.remote.default-remote-dispatcher-5] [akka.serialization.Serialization(akka://ClusterSystem)] Using the default Java serializer for class [cn.sunxiang0918.akka.demo7.FactorialResult] which is not recommended because of performance implications. Use another serializer or disable this warning using the setting 'akka.actor.warn-about-java-serializer-usage'
[WARN] [02/11/2016 22:45:14.622] [ClusterSystem-akka.remote.default-remote-dispatcher-6] [akka.serialization.Serialization(akka://ClusterSystem)] Using the default Java serializer for class [cn.sunxiang0918.akka.demo7.FactorialResult] which is not recommended because of performance implications. Use another serializer or disable this warning using the setting 'akka.actor.warn-about-java-serializer-usage'
[WARN] [02/11/2016 22:45:14.622] [ClusterSystem-akka.remote.default-remote-dispatcher-5] [akka.serialization.Serialization(akka://ClusterSystem)] Using the default Java serializer for class [cn.sunxiang0918.akka.demo7.FactorialResult] which is not recommended because of performance implications. Use another serializer or disable this warning using the setting 'akka.actor.warn-about-java-serializer-usage'
计算的结果:FactorialResult{n=10, factorial=3628800}
```

