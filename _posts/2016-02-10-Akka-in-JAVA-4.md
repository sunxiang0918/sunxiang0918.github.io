---
title: Akka in JAVA(四)
date: 2016-02-10 20:35:02
tags:
- JAVA
- Akka
---

# Akka in JAVA(四)
最后这个部分讲一讲AKKA中的事件消息类型. 在Akka中主要是有三种消息类型,每一种类型对应了不同的使用场景.他们分别是:`Fire and Forget模式`,`Send and Receive模式`和`Publisher-Subscriber模式`.

## Fire and Forget模式
这种发送消息的模式是Akka中所推荐的,也是我们前面一直在使用的方式.也就是单向消息模式,Actor在发送消息之后,并不需要获取响应.这种方式在JAVA中需要使用`ActorRef`或`ActorSelection`的`tell`方法.和消息队列类似,直接调用该方法即可,程序不会阻塞,会直接执行后面的操作,但是消息已经发送给目标的Actor了.这种方式的具体使用方法前面已经列举了很多了,这里就不再重复的举例了.

## Send and Receive模式
这种发送消息的模式是双向的,当Actor在发送消息之后,会接收到一个Future对象.和JAVA的Future一样,通过这个可以异步的接收到对方的结果消息.

在整个AKKA中,提供了一套完整的Future机制,不光是在Actor传递消息间可以使用,也可以在非Actor中直接使用.

<!--more-->

### 在代码中用来异步运算
在一般的代码中,我们除了可以直接使用JAVA提供了`Future`机制实现异步处理外,还可以使用`Akka`提供了`Future`机制来实现异步处理,这样的好处在于可以直接的与Akka做无缝的集成.

与`Future`相关的类和方法都在`akka.dispatch`和`akka.pattern.Patterns`中.

比如我们来看一个很简单的例子:

```java
public class FutureTest {

    public static void main(String[] args) throws Exception {

        final ActorSystem system = ActorSystem.create("helloakka");
        
        /*通过Futures的静态方法future创建一个Future类.入参就是一个异步的计算*/
        Future<String> f = Futures.future(() -> {
        	  Thread.sleep(10);
            if (new Random(System.currentTimeMillis()).nextBoolean()){
                return "Hello"+"World!";
            }else {
                throw new IllegalArgumentException("参数错误");       
            }
        },system.dispatcher());

        f.onSuccess(new PrintResult<String>(),system.dispatcher());
        f.onFailure(new FailureResult(),system.dispatcher());
        System.out.println("这个地方是外面");
    }

    public final static class PrintResult<T> extends OnSuccess<T> {
        @Override public final void onSuccess(T t) {
            System.out.println(t);
        }
    }
    
    public final static class FailureResult extends OnFailure {
        @Override
        public void onFailure(Throwable failure) throws Throwable {
            System.out.println("进入错误的处理");
            failure.printStackTrace();
        }
    }
}
```
这个例子使用`Futures.future`创建了一个异步的代码执行块.然后可以指定`Future`的`onSuccess`和`onFailure`等状态的响应.当Future执行到这些状态的时候,就会执行响应代码中的方法.
比如这个例子中,在`Futures.future`异步执行体中随机的返回正常结果或抛出异常.然后增加了`future`成功和失败的两个状态的处理,分别是打印正常的结果和打印失败的异常.当我们执行这段代码后,由于异步代码块中休眠了10毫秒,因此必然会先打印`这个地方是外面`这句话,这也证明了这些代码全是异步执行的.然后随机的会打印成功或失败的信息.

除了可以给`Future`设定`onSuccess` `onFailure`和`onComplete`外,还有一个很有用的功能就是可以设定`Future`的后续操作,也就是多个`Future`联合操作.比如接着上一个例子中的:

```java
f.andThen(new OnComplete<String>() {
            @Override
            public void onComplete(Throwable failure, String success) throws Throwable {
                System.out.println("这里是andThen");
            }
        },system.dispatcher()).andThen(new OnComplete<String>() {
            @Override
            public void onComplete(Throwable failure, String success) throws Throwable {
                System.out.println("这里是andThen2");
            }
        },system.dispatcher());
```
当执行完`f`的`onSuccess`方法后,接下来会执行第一个`andThen`中所指定的异步操作,而后继续执行第二个`andThen`中的操作.通过这样的操作,就可以形成一条异步调用链.

除此之外,`Future`还有很多有用的方法,比如`foreach` `transform` `map` `filter`等等.这些个方法和JDK1.8中的`StreamAPI`非常的相似.具体的可以参考[这里](http://doc.akka.io/docs/akka/2.4.1/java/futures.html).

### 在Actor中用来发送消息
除了直接在代码中使用`Future`功能来使用异步操作外,另外一个用法就是真正的在Actor中进行消息的传递了. 在[Akka in JAVA(一)](/2016/01/10/Akka-in-JAVA-1/)中我们已经演示过通过`MailInbox`的方式来接收消息的反馈,而使用Akka的`Future`方式进行`Send and Receive模式`的消息传递是第二种方式.

Akka提供了`ask`和`pipe`两个方法来实现`Send and Receive模式`的消息通信.

`ask`方法是`Patterns`类提供的.它最常用的方法签名是`Future<Object> ask(ActorRef actor, Object message, Timeout timeout)`.也就是像某一个或某一组Actor发送一个消息,并设定一个超时时间,它返回一个`Future`对象.这样就可以像上面的例子那样的设定响应的事件处理了. 但是需要注意的是,ask的被调用Actor必须在`onReceive`方法中显示的调用`getSender().tell(xxxx,getSelf())`发送Response为返回的Future填充数据.
同时,Akka还提供了一个`Await.result`方法来阻塞的获取`Future`的结果.比如:

```java
Timeout timeout = new Timeout(Duration.create(5, "seconds"));
Future<Object> future = Patterns.ask(actor, msg, timeout);
String result = (String) Await.result(future, timeout.duration());
```

而`pipe`方法同样是`Patterns`类提供的.它的最主要的作用是把一个`Future`的结果发送给某一个Actor.也就是指定当某一个`future`执行结束后,把结果发送给某个Actor.

在我们的[上一篇博客](/2016/01/18/Akka-in-JAVA-3/)的例子中`FactorialBackend`类就使用了这个语法:

```java
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
```

它使用`future`函数,异步的计算阶乘.并且把计算结果进行封装处理.然后通过`pipe`方法,把这个异步的结果尝试发送给`getSender()`,从而达到整个代码完全非阻塞的结果.

## Publisher-Subscriber模式
这种模式也就是消息的订阅和发布.用处非常的广泛,比如一个Publisher和多个Subscriber的组合应用,形成事件的广播,前者会将消息同时发送给所有的订阅者,实现分布式的并行处理.比如针对订单的处理,当用户下了订单后,既要生成订单数据,又要通知库存还要通知卖方和买房.于是,就可以将这些不同的任务交给不同的Subscriber,当接收到消息后,同时对订单进行处理.另外,这种模式还可以对Actor的生命周期进行完整的监听,当Actor的节点成员发生变化后,其他节点可以及时的进行各种处理.
具体的例子,可以参考上一篇博客中的`SimpleClusterListener`,它就是在`preStart`方法中调用了`cluster.subscribe`方法订阅了集群中的`MemberEvent.class` `UnreachableMember.class`两个事件.
除了监听集群中的状态外,还可以通过调用`system.eventStream()`来获取消息总线,从而调用`subscribe`和`publish`方法来发布和订阅消息.

## Demo

这里使用一个稍微复杂点的例子来对Akka做一个简单的总结.它涉及到了Akka中的远程调用,本地调用,集群调用.使用了`Future` `Publisher-Subscriber`等特性,算是一个比较全面的例子.

这个例子原载于[http://shiyanjun.cn/archives/1186.html](http://shiyanjun.cn/archives/1186.html)上,是使用`scala`写的,我用`JAVA`重写了一次,并增加了一些东西.这个例子主要的功能是实现一个简单的模拟日志实时处理的集群系统,类似于`Flume`,可以从某一个数据源中输入数据,然后程序收集数据,然后经过一个拦截器层,处理数据,并且转换为特定的格式,最后把数据写入`Kafka`中.具体的逻辑如下图:

![](/img/2016/02/10/1.png)

在上图中,将日志处理系统分为了3个子部分,通过Akka的Role来进行划分,3个角色分别为`collector`收集器,`interceptor`拦截器,`processor`处理器,3个子系统中的节点都是整个Akka集群中的成员.整个集群系统的数据流向是:`collector`接收数据,然后将数据发送到`interceptor`,`interceptor`接收到数据后,解析出真实IP地址,拦截存在于黑名单中的IP请求,如果IP地址不在黑名单,则发送给`processor`去转换为最终的模型,然后保存到Kafka中.

首先,我们需要定义的是几个子系统间传递的消息:

**EventMessages.java**

```java
public interface EventMessages {

    public static class EventMessage implements Serializable {

    }

    /**
     * 内存中的Nginx的日志
     */
    public static class RawNginxRecord extends EventMessage {

        private String sourceHost;

        private String line;

        public RawNginxRecord(String sourceHost, String line) {
            this.sourceHost = sourceHost;
            this.line = line;
        }

        public String getLine() {
            return line;
        }

        public String getSourceHost() {
            return sourceHost;
        }
    }

    /**
     * 解析出了事件内容的Nginx记录
     */
    public static class NginxRecord extends EventMessage {

        private String sourceHost;

        private String line;

        private String eventCode;

        public NginxRecord(String sourceHost, String line, String eventCode) {
            this.sourceHost = sourceHost;
            this.line = line;
            this.eventCode = eventCode;
        }

        public String getSourceHost() {
            return sourceHost;
        }

        public String getLine() {
            return line;
        }

        public String getEventCode() {
            return eventCode;
        }
    }

    /**
     * 通过了拦截器的日志记录
     */
    public static class FilteredRecord extends EventMessage {

        private String sourceHost;

        private String line;

        private String eventCode;

        private String logDate;

        private String realIp;

        public FilteredRecord(String sourceHost, String line, String eventCode, String logDate, String realIp) {
            this.sourceHost = sourceHost;
            this.line = line;
            this.eventCode = eventCode;
            this.logDate = logDate;
            this.realIp = realIp;
        }

        public String getSourceHost() {
            return sourceHost;
        }

        public String getLine() {
            return line;
        }

        public String getEventCode() {
            return eventCode;
        }

        public String getLogDate() {
            return logDate;
        }

        public String getRealIp() {
            return realIp;
        }
    }

    /**
     * 子系统注册的消息
     */
    public static final class Registration implements Serializable {
    }

}
```

然后就是抽象出来的订阅集群事件相关的逻辑.

**ClusterRoledWorker.java**

```java
public abstract class ClusterRoledWorker extends UntypedActor{

    /*记录日志*/
    protected LoggingAdapter log = Logging.getLogger(getContext().system(), this);

    /*集群系统*/
    protected Cluster cluster = Cluster.get(getContext().system());

    // 用来缓存下游注册过来的子系统ActorRef
    protected List<ActorRef> workers = new ArrayList<>();

    @Override
    public void preStart() throws Exception {
        // 订阅集群事件
        cluster.subscribe(getSelf(), ClusterEvent.initialStateAsEvents(), ClusterEvent.MemberUp.class,ClusterEvent.MemberEvent.class, ClusterEvent.UnreachableMember.class);
    }

    @Override
    public void postStop() throws Exception {
        // 取消事件监听
        cluster.unsubscribe(getSelf());
    }

    /**
     * 下游子系统节点发送注册消息
     */

    protected void register(Member member,String actorPath) {
        ActorSelection actorSelection = getContext().actorSelection(actorPath);
        
        /*发送注册消息*/
        actorSelection.tell(new EventMessages.Registration(),getSelf());
    }
    
}
```

然后,就是对整个Akka的集群进行配置: 

**demo8.conf**

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
      "akka.tcp://event-cluster-system@127.0.0.1:2751",
      "akka.tcp://event-cluster-system@127.0.0.1:2752",
      "akka.tcp://event-cluster-system@127.0.0.1:2753"]
    seed-node-timeout = 60s
    auto-down-unreachable-after = 10s
  }
}
```

接下来就是Collector的实现,它是一个Acotr,继承自ClusterRoledWorker抽象类:

**EventCollector.java**

```java
public class EventCollector extends ClusterRoledWorker {

    private AtomicInteger recordCounter = new AtomicInteger(0);


    @Override
    public void onReceive(Object message) throws Exception {
        if (message instanceof ClusterEvent.MemberUp) {
            ClusterEvent.MemberUp member = (ClusterEvent.MemberUp) message;
            log.info("Member is Up: {}", member.member().address());
        } else if (message instanceof ClusterEvent.UnreachableMember) {
            ClusterEvent.UnreachableMember mUnreachable = (ClusterEvent.UnreachableMember) message;
            log.info("Member detected as unreachable: {}", mUnreachable.member());
        } else if (message instanceof ClusterEvent.MemberRemoved) {
            ClusterEvent.MemberRemoved mRemoved = (ClusterEvent.MemberRemoved) message;
            log.info("Member is Removed: {}", mRemoved.member());
        } else if (message instanceof ClusterEvent.MemberEvent) {
            // ignore
            log.info("Member Event: {}", ((ClusterEvent.MemberEvent) message).member());
        } else if (message instanceof EventMessages.Registration) {
            // watch发送注册消息的interceptor，如果对应的Actor终止了，会发送一个Terminated消息
            getContext().watch(getSender());
            workers.add(getSender());
            log.info("Interceptor registered: " + getSender());
            log.info("Registered interceptors: " + workers.size());
        } else if (message instanceof Terminated) {
            // interceptor终止，更新缓存的ActorRef
            Terminated terminated = (Terminated) message;
            workers.remove(terminated.actor());
        } else if (message instanceof EventMessages.RawNginxRecord) {
            EventMessages.RawNginxRecord rawNginxRecord = (EventMessages.RawNginxRecord) message;
            String line = rawNginxRecord.getLine();
            String sourceHost = rawNginxRecord.getSourceHost();
            String eventCode = findEventCode(line);

            // 构造NginxRecord消息，发送到下游interceptor
            log.info("Raw message: eventCode=" + eventCode + ", sourceHost=" + sourceHost + ", line=" + line);
            int counter = recordCounter.incrementAndGet();
            if (workers.size() > 0) {
                // 模拟Roudrobin方式，将日志记录消息发送给下游一组interceptor中的一个
                int interceptorIndex = (counter < 0 ? 0 : counter) % workers.size();
                workers.get(interceptorIndex).tell(new EventMessages.NginxRecord(sourceHost, line, eventCode), getSelf());
                log.info("Details: interceptorIndex=" + interceptorIndex + ", interceptors=" + workers.size());
            }
        }
    }

    private String findEventCode(String line) {
        Pattern pattern = Pattern.compile("eventcode=(\\d+)");
        Matcher matcher = pattern.matcher(line);
        if (matcher.find()) {
            return matcher.group(1);
        }
        return null;
    }
}
```

然后就是Interceptor的实现,和Collector类似,同样是继承自`ClusterRoledWorker`抽象类:

**EventInterceptor.java**

```java
public class EventInterceptor extends ClusterRoledWorker {

    private AtomicInteger interceptedRecords = new AtomicInteger(0);

    /*IP地址的正则表达式*/
    private Pattern IP_PATTERN = Pattern.compile("[^\\s]+\\s+\\[([^\\]]+)\\].+\"(\\d+\\.\\d+\\.\\d+\\.\\d+)");

    /*黑名单*/
    private List<String> blackIpList = Arrays.asList("5.9.116.101", "103.42.176.138", "123.182.148.65", "5.45.64.205",
            "27.159.226.192", "76.164.228.218", "77.79.178.186", "104.200.31.117",
            "104.200.31.32", "104.200.31.238", "123.182.129.108", "220.161.98.39",
            "59.58.152.90", "117.26.221.236", "59.58.150.110", "123.180.229.156",
            "59.60.123.239", "117.26.222.6", "117.26.220.88", "59.60.124.227",
            "142.54.161.50", "59.58.148.52", "59.58.150.85", "202.105.90.142");
    
    @Override
    public void onReceive(Object message) throws Exception {
        if (message instanceof ClusterEvent.MemberUp){
            ClusterEvent.MemberUp member = (ClusterEvent.MemberUp) message;
            log.info("Member is Up: {}", member.member().address());
            
            register(member.member(), getCollectorPath(member.member()));
        }else if (message instanceof ClusterEvent.CurrentClusterState) {

            ClusterEvent.CurrentClusterState state = (ClusterEvent.CurrentClusterState) message;
            Iterable<Member> members = state.getMembers();
            
            // 如果加入Akka集群的成员节点是Up状态，并且是collector角色，则调用register向collector进行注册
            members.forEach(o -> {
                if (o.status() == MemberStatus.up()){
                    register(o,getCollectorPath(o));
                }
            });
        } else if (message instanceof ClusterEvent.UnreachableMember) {
            ClusterEvent.UnreachableMember mUnreachable = (ClusterEvent.UnreachableMember) message;
            log.info("Member detected as unreachable: {}", mUnreachable.member());
        } else if (message instanceof ClusterEvent.MemberRemoved) {
            ClusterEvent.MemberRemoved mRemoved = (ClusterEvent.MemberRemoved) message;
            log.info("Member is Removed: {}", mRemoved.member());
        } else if (message instanceof ClusterEvent.MemberEvent) {
            // ignore
            log.info("Member Event: {}", ((ClusterEvent.MemberEvent) message).member());
        } else if (message instanceof EventMessages.Registration) {
            // watch发送注册消息的interceptor，如果对应的Actor终止了，会发送一个Terminated消息
            getContext().watch(getSender());
            workers.add(getSender());
            log.info("Interceptor registered: " + getSender());
            log.info("Registered interceptors: " + workers.size());
        } else if (message instanceof Terminated) {
            // interceptor终止，更新缓存的ActorRef
            Terminated terminated = (Terminated) message;
            workers.remove(terminated.actor());
        }else if (message instanceof EventMessages.NginxRecord) {
            EventMessages.NginxRecord nginxRecord = (EventMessages.NginxRecord) message;

            CheckRecord checkRecord = checkRecord(nginxRecord.getEventCode(), nginxRecord.getLine());
            
            if (!checkRecord.isIpInBlackList){
                int records = interceptedRecords.incrementAndGet();
                
                if (workers.size()>0){
                    int processorIndex = (records<0?0:records) % workers.size();
                    workers.get(processorIndex).tell(new EventMessages.FilteredRecord(nginxRecord.getSourceHost(),nginxRecord.getLine(), nginxRecord.getEventCode() , checkRecord.data.get("eventdate"), checkRecord.data.get("realip")),getSelf());
                    log.info("Details: processorIndex=" + processorIndex + ", processors=" + workers.size());
                }
                log.info("Intercepted data: data=" + checkRecord.data);
            }else {
                log.info("Discarded: " + nginxRecord.getLine());
            }
        }
    }

    /**
     * 检查和解析每一行日志的记录
     * @param eventCode
     * @param line
     * @return
     */
    private CheckRecord checkRecord(String eventCode,String line){
        
        Map<String,String> data = new HashMap<>();
        boolean isIpInBlackList = false;

        Matcher matcher = IP_PATTERN.matcher(line);

        while (matcher.find()) {
            String rawDt = matcher.group(1);
            String realIp = matcher.group(2);

            data.put("eventdate", rawDt);
            data.put("realip", realIp);
            data.put("eventcode", eventCode);
            
            isIpInBlackList = blackIpList.contains(realIp);
        }
        
        return new CheckRecord(isIpInBlackList,data);
    }

    /**
     * 获取Collector的路径
     * @param member
     * @return
     */
    private String getCollectorPath(Member member){
        return member.address()+"/user/collectingActor";
    }
    
    private class CheckRecord {
        private boolean isIpInBlackList;
        
        private Map<String,String> data;

        public CheckRecord(boolean isIpInBlackList, Map<String, String> data) {
            this.isIpInBlackList = isIpInBlackList;
            this.data = data;
        }
    }
}
```

然后就是Interceptor这个子系统的启动器:

**EventInterceptorMain.java**

```java
public class EventInterceptorMain {

    public static void main(String[] args) throws Exception {
        
        final String port = args.length > 0 ? args[0] : "0";
        
        /*修改配置文件中的端口和角色*/
        final Config config = ConfigFactory.parseString("akka.remote.netty.tcp.port=" + port).
                withFallback(ConfigFactory.parseString("akka.cluster.roles = [interceptor]")).
                withFallback(ConfigFactory.load("demo8"));

        final ActorSystem system = ActorSystem.create("event-cluster-system", config);

        /*实例化EventInterceptor Actor*/
        ActorRef interceptingActor = system.actorOf(Props.create(EventInterceptor.class), "interceptingActor");

        system.log().info("Processing Actor: " + interceptingActor);
    }
}
```

在接着就是Processor的实现了:

**EventProcessor.java**

```java
public class EventProcessor extends ClusterRoledWorker {

    /*内容的正则表达式*/
    private Pattern PATTERN = Pattern.compile("[\\?|&]([^=]+)=([^&]+)&");

    /*kafka的连接工具*/
    private KafkaTemplate kafkaTemplate = new KafkaTemplate("127.0.0.1:8092");
    
    @Override
    public void onReceive(Object message) throws Exception {
        if (message instanceof ClusterEvent.MemberUp) {
            ClusterEvent.MemberUp member = (ClusterEvent.MemberUp) message;
            log.info("Member is Up: {}", member.member().address());

            register(member.member(), getProcessorPath(member.member()));
        } else if (message instanceof ClusterEvent.CurrentClusterState) {

            ClusterEvent.CurrentClusterState state = (ClusterEvent.CurrentClusterState) message;
            Iterable<Member> members = state.getMembers();

            // 如果加入Akka集群的成员节点是Up状态，并且是collector角色，则调用register向collector进行注册
            members.forEach(o -> {
                if (o.status() == MemberStatus.up()) {
                    register(o, getProcessorPath(o));
                }
            });
        } else if (message instanceof ClusterEvent.UnreachableMember) {
            ClusterEvent.UnreachableMember mUnreachable = (ClusterEvent.UnreachableMember) message;
            log.info("Member detected as unreachable: {}", mUnreachable.member());
        } else if (message instanceof ClusterEvent.MemberRemoved) {
            ClusterEvent.MemberRemoved mRemoved = (ClusterEvent.MemberRemoved) message;
            log.info("Member is Removed: {}", mRemoved.member());
        } else if (message instanceof ClusterEvent.MemberEvent) {
            // ignore
            log.info("Member Event: {}", ((ClusterEvent.MemberEvent) message).member());
        } else if (message instanceof EventMessages.FilteredRecord) {
            EventMessages.FilteredRecord filteredRecord = (EventMessages.FilteredRecord) message;
            /*处理每一行日志内容,转换成Map模型*/
            Map<String, String> data = process(filteredRecord.getEventCode(), filteredRecord.getLine(), filteredRecord.getLogDate(), filteredRecord.getRealIp());

            log.info("Processed: data=" + data);
            
            // 将解析后的消息一JSON字符串的格式，保存到Kafka中
            kafkaTemplate.convertAndSend("app_events", JSON.toJSONString(data));
        }
    }

    private Map<String, String> process(String eventCode, String line, String logDate, String realIp) {

        Map<String, String> data = new HashMap<>();
        
        Matcher matcher = PATTERN.matcher(line);

        while (matcher.find()) {
            String key = matcher.group(1);
            String value = matcher.group(2);
            data.put(key,value);
        }

        data.put("eventdate", logDate);
        data.put("realip", realIp);

        return data;
    }

    private String getProcessorPath(Member member) {
        return member.address()+"/user/interceptingActor";
    }
}
```

上面涉及到kafka消息的发送,我使用了一个工具类来进行消息的发送:

**KafkaTemplate.java**

```java
public class KafkaTemplate {

    private Producer<String, Serializable> producer;

    public KafkaTemplate(String urls) {
        Properties props = new Properties();
        props.put("metadata.broker.list", urls);
        props.put("serializer.class", "cn.sunxiang0918.akka.demo8.kafka.ObjectEncoder");
        props.put("request.required.acks", "1");

        ProducerConfig config = new ProducerConfig(props);

        producer = new Producer<>(config);
    }

    public void convertAndSend(String destinationName, Object message) {
        assert message instanceof Serializable;
        KeyedMessage<String, Serializable> data = new KeyedMessage<>(destinationName, destinationName, (Serializable) message);
        producer.send(data);
    }
}

public class ObjectDecoder implements Decoder<Serializable> {

    public ObjectDecoder(){}
    
    public ObjectDecoder(VerifiableProperties props) {
//        this.encoding = props == null?"UTF8":props.getString("serializer.encoding", "UTF8");
    }
    
    public Serializable fromBytes(byte[] bytes) {

//        return (Serializable) SerializationUtils.deserialize(bytes);

        if (bytes == null) {
            return null;
        }
        try {
            ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(bytes));
            return (Serializable) ois.readObject();
        }
        catch (IOException ex) {
            throw new IllegalArgumentException("Failed to deserialize object", ex);
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException("Failed to deserialize object type", ex);
        }
    }
}

public class ObjectEncoder implements Encoder<Serializable> {

    public ObjectEncoder(){}
    
    public ObjectEncoder(VerifiableProperties props) {
//        this.encoding = props == null?"UTF8":props.getString("serializer.encoding", "UTF8");
    }
    
    public byte[] toBytes(Serializable object) {
        if (object == null) {
            return null;
        }
        ByteArrayOutputStream baos = new ByteArrayOutputStream(1024);
        try {
            ObjectOutputStream oos = new ObjectOutputStream(baos);
            oos.writeObject(object);
            oos.flush();
        }
        catch (IOException ex) {
            throw new IllegalArgumentException("Failed to serialize object of type: " + object.getClass(), ex);
        }
        return baos.toByteArray();
    }
}
```

同样需要Processor子系统的启动器:

**EventProcessorMain.java**

```java
public class EventProcessorMain {

    public static void main(String[] args) throws Exception {
        final String port = args.length > 0 ? args[0] : "0";

        final Config config = ConfigFactory.parseString("akka.remote.netty.tcp.port=" + port).
                withFallback(ConfigFactory.parseString("akka.cluster.roles = [processor]")).
                withFallback(ConfigFactory.load("demo8"));

        final ActorSystem system = ActorSystem.create("event-cluster-system", config);

        ActorRef processingActor = system.actorOf(Props.create(EventProcessor.class), "processingActor");

        system.log().info("Processing Actor: " + processingActor);
    }
}
```

最后就是需要写一个模拟的发送日志的客户端了,并且包含了Collector子系统的启动器:

**EventClient.java**

```java
public class EventClient {

    /*假的Nginx的日志,Key是Nginx的端口,Value是日志内容,一行一个*/
    private static Map<Integer,List<String>> events=new HashMap<>();
    
    static {
        /*构造假的日志内容*/
        events.put(2751,new ArrayList<>());
        events.put(2752,new ArrayList<>());
        events.put(2753,new ArrayList<>());

        events.get(2751).add("\"\"10.10.2.72 [21/Aug/2015:18:29:19 +0800] \"GET /t.gif?installid=0000lAOX&udid=25371384b2eb1a5dc5643e14626ecbd4&sessionid=25371384b2eb1a5dc5643e14626ecbd41440152875362&imsi=460002830862833&operator=1&network=1&timestamp=1440152954&action=14&eventcode=300039&page=200002& HTTP/1.0\" \"-\" 204 0 \"-\" \"Dalvik/1.6.0 (Linux; U; Android 4.4.4; R8207 Build/KTU84P)\" \"121.25.190.146\"\"\"");
        events.get(2751).add("\"\"10.10.2.8 [21/Aug/2015:18:29:19 +0800] \"GET /t.gif?installid=0000VACO&udid=f6b0520cbc36fda6f63a72d91bf305c0&imsi=460012927613645&operator=2&network=1&timestamp=1440152956&action=1840&eventcode=100003&type=1&result=0& HTTP/1.0\" \"-\" 204 0 \"-\" \"Dalvik/1.6.0 (Linux; U; Android 4.4.2; GT-I9500 Build/KOT49H)\" \"61.175.219.69\"\"\"");

        events.get(2752).add("\"\"10.10.2.72 [21/Aug/2015:18:29:19 +0800] \"GET /t.gif?installid=0000gCo4&udid=636d127f4936109a22347b239a0ce73f&sessionid=636d127f4936109a22347b239a0ce73f1440150695096&imsi=460036010038180&operator=3&network=4&timestamp=1440152902&action=1566&eventcode=101010&playid=99d5a59f100cb778b64b5234a189e1f4&radioid=1100000048450&audioid=1000001535718&playtime=3& HTTP/1.0\" \"-\" 204 0 \"-\" \"Dalvik/1.6.0 (Linux; U; Android 4.4.4; R8205 Build/KTU84P)\" \"106.38.128.67\"\"\"");
        events.get(2752).add("\"\"10.10.2.72 [21/Aug/2015:18:29:19 +0800] \"GET /t.gif?installid=0000kPSC&udid=2ee585cde388ac57c0e81f9a76f5b797&operator=0&network=1&timestamp=1440152968&action=6423&eventcode=100003&type=1&result=0& HTTP/1.0\" \"-\" 204 0 \"-\" \"Dalvik/v3.3.85 (Linux; U; Android L; P8 Build/KOT49H)\" \"202.103.133.112\"\"\"");
        events.get(2752).add("\"\"10.10.2.72 [21/Aug/2015:18:29:19 +0800] \"GET /t.gif?installid=0000lABW&udid=face1161d739abacca913dcb82576e9d&sessionid=face1161d739abacca913dcb82576e9d1440151582673&operator=0&network=1&timestamp=1440152520&action=1911&eventcode=101010&playid=b07c241010f8691284c68186c42ab006&radioid=1100000000762&audioid=1000001751983&playtime=158& HTTP/1.0\" \"-\" 204 0 \"-\" \"Dalvik/1.6.0 (Linux; U; Android 4.1; H5 Build/JZO54K)\" \"221.232.36.250\"\"\"");
        
        
        events.get(2753).add("\"\"10.10.2.8 [21/Aug/2015:18:29:19 +0800] \"GET /t.gif?installid=0000krJw&udid=939488333889f18e2b406d2ece8f938a&sessionid=939488333889f18e2b406d2ece8f938a1440137301421&imsi=460028180045362&operator=1&network=1&timestamp=1440152947&action=1431&eventcode=300030&playid=e1fd5467085475dc4483d2795f112717&radioid=1100000001123&audioid=1000000094911&playtime=951992& HTTP/1.0\" \"-\" 204 0 \"-\" \"Dalvik/1.6.0 (Linux; U; Android 4.0.4; R813T Build/IMM76D)\" \"5.45.64.205\"\"\"");
        events.get(2753).add("\"\"10.10.2.72 [21/Aug/2015:18:29:19 +0800] \"GET /t.gif?installid=0000kcpz&udid=cbc7bbb560914c374cb7a29eef8c2144&sessionid=cbc7bbb560914c374cb7a29eef8c21441440152816008&imsi=460008782944219&operator=1&network=1&timestamp=1440152873&action=360&eventcode=200003&page=200003&radioid=1100000046018& HTTP/1.0\" \"-\" 204 0 \"-\" \"Dalvik/v3.3.85 (Linux; U; Android 4.4.2; MX4S Build/KOT49H)\" \"119.128.106.232\"\"\"");
        events.get(2753).add("\"\"10.10.2.8 [21/Aug/2015:18:29:19 +0800] \"GET /t.gif?installid=0000juRL&udid=3f9a5ffa69a5cd5f0754d2ba98c0aeb2&imsi=460023744091238&operator=1&network=1&timestamp=1440152957&action=78&eventcode=100003&type=1&result=0& HTTP/1.0\" \"-\" 204 0 \"-\" \"Dalvik/v3.3.85 (Linux; U; Android 4.4.3; S?MSUNG. Build/KOT49H)\" \"223.153.72.78\"\"\"");
    }
    
    private static List<Integer> ports = Arrays.asList(2751,2752, 2753);
    
    private static Map<Integer,ActorRef> actors = new HashMap<>();
    
    public static void main(String[] args) throws Exception {

        /*根据端口号的多少,启动多少个Collector在集群中*/
        ports.forEach(port -> {
            final Config config = ConfigFactory.parseString("akka.remote.netty.tcp.port=" + port).
                    withFallback(ConfigFactory.parseString("akka.cluster.roles = [collector]")).
                    withFallback(ConfigFactory.load("demo8"));
            final ActorSystem system = ActorSystem.create("event-cluster-system", config);

            ActorRef collectingActor = system.actorOf(Props.create(EventCollector.class), "collectingActor");

            actors.put(port,collectingActor);
        });

        Thread.sleep(10000);

        /*使用JDK中的scheduleAtFixedRate,每5秒发送一次日志给Collector*/
        ScheduledExecutorService service = Executors.newSingleThreadScheduledExecutor();

        service.scheduleAtFixedRate(() -> {
            ports.forEach(port -> {
                events.get(port).forEach(line ->{
                    System.out.println("RAW: port=" + port + ", line=" + line);
                    actors.get(port).tell(new EventMessages.RawNginxRecord("host.me:" + port, line),ActorRef.noSender());
                } );
            } );
        },0,5, TimeUnit.SECONDS);
    }
}
```

最后就是整个Demo的入口:

**Demo8App.java**

```java
public class Demo8App {

    public static void main(String[] args) throws Exception {

        // 启动一个Client
        EventClient.main(new String[0]);
        
        // 启动两个Interceptor
        EventInterceptorMain.main(new String[] { "2851" });
        EventInterceptorMain.main(new String[] { "2852" });
        
        // 启动两个Processor
        EventProcessorMain.main(new String[]{"2951"});
        EventProcessorMain.main(new String[]{"2952"});
        EventProcessorMain.main(new String[]{"2953"});
        EventProcessorMain.main(new String[]{"2954"});
        EventProcessorMain.main(new String[]{"2955"});
        
    }
}
```

当执行了这个Demo后,首先会看到的就是和上一篇博客中一样的集群启动和加入的消息.而后会看到每5秒接收到一批Nginx的日志,并发送给Kafka.这时可以通过Kafka的客户端看到这些处理后的日志:

```
{"installid":"0000VACO","imsi":"460012927613645","network":"1","action":"1840","type":"1","eventdate":"2015-08-21 18:29:19","realip":"61.175.219.69"}
{"installid":"0000kcpz","sessionid":"cbc7bbb560914c374cb7a29eef8c21441440152816008","operator":"1","timestamp":"1440152873","eventcode":"200003","radioid":"1100000046018","eventdate":"2015-08-21 18:29:19","realip":"119.128.106.232"}
{"installid":"0000lAOX","sessionid":"25371384b2eb1a5dc5643e14626ecbd41440152875362","operator":"1","timestamp":"1440152954","eventcode":"300039","eventdate":"2015-08-21 18:29:19","realip":"121.25.190.146"}
```

## 总结
经过这四篇的博客,简单的介绍了一下Akka是什么,大概的用处是什么,以及Akka在JAVA中该如何的使用.由于篇幅以及本人经验的原因,还有很多都没有讲到,以后如有涉及,再慢慢的补上.总之,Akka是一个非常强大的框架,在现在大数据,高性能,分布式的环境下可以发挥很多的作用,各位可以试一试.

PS:本系列文章中所有的Demo的源码已上传至[GitHub](https://github.com/sunxiang0918/AkkaDemo)中.

