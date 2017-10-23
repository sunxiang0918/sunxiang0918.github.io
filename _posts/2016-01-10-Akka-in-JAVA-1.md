---
title: Akka in JAVA(一)
date: 2016-01-10 16:46:28
tags:
- JAVA
- Akka
---

# Akka in JAVA(一)

## AKKA简介

### 什么是AKKA
Akka是一个由`Scala`编写的,能兼容`Sacala`和`JAVA`的,用于编写高可用和高伸缩性的`Actor模型`框架.它基于了事件驱动的并发处理模式,性能非常的高,并且有很高的可用性.大大的简化了我们在应用系统中开发并发处理的过程.它在各个领域都有很好的表现.

### 使用AKKA的好处
就如上面简介中所说的,AKKA把并发操作的各种复杂的东西都统一的做了封装.我们主要关心的是业务逻辑的实现,只需要少量的关心`Actor模型`的串联即可构建出高可用,高性能,高扩展的应用.

### Akka for JAVA
由于AKKA是使用`Scala`编写的,而`Scala`是一种基于JVM的语言.因此`JAVA`对AKKA的支持也是很不错的.Akka自身又是采用微内核的方式来实现的,这就意味着能很容易的在自己的项目中应用AKKA,只需要引入几个akka的Lib包即可.而官方直接就提供了`Maven`库供我们在JAVA中使用AKKA.
这些AKKA的依赖包主要有:

* **akka-actor**:最核心的依赖包,里面实现了Actor模型的大部分东西
* **akka-agent**:代理/整合了Scala中的一些STM特性
* **akka-camel**:整合了Apache的Camel
* **akka-cluster**:akka集群依赖,封装了集群成员的管理和路由
* **akka-kernel**:akka的一个极简化的应用服务器,可以脱离项目单独运行.
* **akka-osgi**:对OSGI容器的支持,有akka的最基本的Bundle
* **akka-remote**:akka远程调用
* **akka-slf4j**:Akka的日志事件监听
* **akka-testkit**:Akka的各种测试工具
* **akka-zeromq**:整合ZeroMQ
其中最总要的就是`akka-actor`,最简单的AKKA使用的话,只需要引入这个包就可以了.

<!--more-->

## Actor模型

### 什么是Actor
既然说AKKA是一个`Actor模型`框架,那么就需要搞清楚什么是`Actor模型`.`Actor模型`是由`Carl Hewitt`于上世纪70年代提出的,目的是为了解决分布式编程中的一系列问题而产生.
在`Actor模型`中,**一切都可以抽象为Actor**.
而Actor是封装了状态和行为的对象,他们的唯一通讯方式就是交换消息,交换的消息放在接收方的邮箱(Inbox)里.也就是说Actor之间并不直接通信,而是通过了消息来相互沟通,每一个Actor都把它要做的事情都封装在了它的内部.
每一个Actor是可以有状态也可以是无状态的,理论上来讲,每一个Actor都拥有属于自己的轻量级线程,保护它不会被系统中的其他部分影响.因此,我们在编写Actor时,就不用担心并发的问题.
通过Actor能够简化锁以及线程管理,Actor具有以下的特性:

* 提供了一种高级的抽象,能够封装状态和操作.简化并发应用的开发.
* 提供了异步的非阻塞的/高性能的事件驱动模型
* 超级轻量级的线程事件处理能力.

要在JAVA中实现一个`Actor`也非常的简单,直接继承`akka.actor.UntypedActor`类,然后实现`public void onReceive(Object message) throws Exception`方法即可.

### Actor系统
光有一个一个独立的Actor显然是不行的.Akka中还有一个`Actor System`.
`Actor System`统管了`Actor`,是Actor的系统工厂或管理者,掌控了Actor的生命周期.

![](/img/2016/01/10/1.png)
如上图所示,我们可以通过`ActorSystem.create`来创建一个ActorSystem的实例.然后通过`actorOf`等方法来获取`ActorRef`对象.`ActorRef`即为`Actor Reference`.它是Actor的一个引用,主要的作用是发送消息给它表示的Actor.而Actor可以通过访问`self()`或`sender()`方法来获取到自身或消息发送者的Actor引用.通过引用发送消息.在Akka中,Actor之间永远都不能直接的通信,必须通过他们的代理`ActorRef`建立通信.

### Actor路径
为了实现一切事物都是Actor,为了能把一个复杂的事物划分的更细致.Akka引入了父子Actor.也就是Actor是有树形结构的关系的.这样的父子结构就能递归的把任何复杂的事物原子化.这也是Actor模型的精髓所在.这样做不仅使任务本身被清晰地划分出结构,而且最终的Actor也能按照他们明确的消息类型以及处理流程来进行解析.这样的递归结构使得消息能够在正确的层次进行处理.

![](/img/2016/01/10/2.png)

为了能管理父子结构的Actor,Akka又引入了`Actor Path`,也就是Actor路径.
Actor路径使用类似于URL的方式来描述一个Actor,`Actor Path`在一个`Actor System`中是唯一的.通过路径,可以很明确的看出某个Actor的父级关系是怎样的.

```bash
//本地Actor
"akka://my-sys/user/service-a/worker1"

//远程Actor
"akka.tcp://my-sys@host.example.com:2552/user/service-b"

//集群Actor服务
"cluster://my-cluster/service-c"
```
以上三种就是Akka中支持的`Actor`路径. 每一个通过ActorSystem创建出来的Actor都会有一个这样的路径.也可以通过这个路径从ActorSystem中获取一个`Actor`.

当我们创建一个ActorSystem的时候,AKKA会为该System默认的创建三个Actor,并处于不同的层次:

![](/img/2016/01/10/3.png)
其中的`root guardian`是所有Actor的父.
而`User`Actor是所有用户创建的Actor的父.它的路径是`/user`,通过system.actorOf()创建出来的Actor都算是用户的Actor,也都是这个Actor的子.
`System`Actor是所有系统创建的Actor的父.它的路径是`/system`,主要的作用是提供了一系列的系统的功能.

当我们查找一个Actor的时候,可以使用ActorSystem.actorSelection()方法.并且可以使用绝对路径或者相对路径来获取.如果是相对路径,那么`..`表示的是父Actor.比如:

```java
ActorSelection selection = system.actorSelection("../brother");
ActorRef actor = selection.anchor();
selection.tell(xxx);
```
同时,也可以通过通配符来查询逻辑的Actor层级,比如:

```java
ActorSelection selection = system.actorSelection("../*");
selection.tell(xxx);
```
这个就表示把消息发送给当前Actor之外的所有同级的Actor.

## Hello AKKA Demo
原理讲了这么多,那么我们就来看一看一个最简单的Akka的例子吧.
这个是一个最简单的打招呼的例子,这个例子中,定义了招呼,打招呼的人两个对象或者说消息.然后定义了执行打招呼和打印招呼两个Actor.然后通过ActorSystem整合整个打招呼的过程.

**Greet.java**

```java
/**
 * 用于表示执行打招呼这个操作的消息
 * @author SUN
 * @version 1.0
 * @Date 16/1/6 21:43
 */
public class Greet implements Serializable {
}
```

**Greeting.java**

```java
/**
 * 招呼体,里面有打的什么招呼
 * @author SUN
 * @version 1.0
 * @Date 16/1/6 21:44
 */
public class Greeting implements Serializable {
    public final String message;
    public Greeting(String message) {
        this.message = message;
    }
}
```

**WhoToGreet.java**

```java
/**
 * 打招呼的人
 * @author SUN
 * @version 1.0
 * @Date 16/1/6 21:41
 */
public class WhoToGreet implements Serializable {
    public final String who;
    public WhoToGreet(String who) {
        this.who = who;
    }
}
```

**Greeter.java**

```java
/**
 * 打招呼的Actor
 * @author SUN
 * @version 1.0
 * @Date 16/1/6 21:40
 */
public class Greeter extends UntypedActor{

    String greeting = "";
    
    @Override
    public void onReceive(Object message) throws Exception {
        if (message instanceof WhoToGreet)
            greeting = "hello, " + ((WhoToGreet) message).who;
        else if (message instanceof Greet)
            // 发送招呼消息给发送消息给这个Actor的Actor
            getSender().tell(new Greeting(greeting), getSelf());

        else unhandled(message);
    }
}
```

**GreetPrinter.java**

```java
/**
 * 打印招呼
 * @author SUN
 * @version 1.0
 * @Date 16/1/6 21:45
 */
public class GreetPrinter extends UntypedActor{

    @Override
    public void onReceive(Object message) throws Exception {
        if (message instanceof Greeting)
            System.out.println(((Greeting) message).message);
    }
}
```

**DemoMain.java**

```java
/**
 * @author SUN
 * @version 1.0
 * @Date 16/1/6 21:39
 */
public class DemoMain {

    public static void main(String[] args) throws Exception {
        final ActorSystem system = ActorSystem.create("helloakka");

        // 创建一个到greeter Actor的管道
        final ActorRef greeter = system.actorOf(Props.create(Greeter.class), "greeter");

        // 创建邮箱
        final Inbox inbox = Inbox.create(system);

        // 先发第一个消息,消息类型为WhoToGreet
        greeter.tell(new WhoToGreet("akka"), ActorRef.noSender());

        // 真正的发送消息,消息体为Greet
        inbox.send(greeter, new Greet());

        // 等待5秒尝试接收Greeter返回的消息
        Greeting greeting1 = (Greeting) inbox.receive(Duration.create(5, TimeUnit.SECONDS));
        System.out.println("Greeting: " + greeting1.message);

        // 发送第三个消息,修改名字
        greeter.tell(new WhoToGreet("typesafe"), ActorRef.noSender());
        // 发送第四个消息
        inbox.send(greeter, new Greet());
        
        // 等待5秒尝试接收Greeter返回的消息
        Greeting greeting2 = (Greeting) inbox.receive(Duration.create(5, TimeUnit.SECONDS));
        System.out.println("Greeting: " + greeting2.message);

        // 新创建一个Actor的管道
        ActorRef greetPrinter = system.actorOf(Props.create(GreetPrinter.class));
        
        //使用schedule 每一秒发送一个Greet消息给 greeterActor,然后把greeterActor的消息返回给greetPrinterActor 
        system.scheduler().schedule(Duration.Zero(), Duration.create(1, TimeUnit.SECONDS), greeter, new Greet(), system.dispatcher(), greetPrinter);
        //system.shutdown();
    }
}
```
以上就是整个Demo的所有代码,并不多.接下来我们就分析一下这个程序.

首先是定义的几个消息.在Akka中传递的消息必须实现`Serializable`接口.`WhoToGreet`消息表示了打招呼的人,`Greeting`表示了招呼的内容,而`Greet`表示了打招呼这个动作.

接着就是两个最重要的Actor了.`GreetPrinter`非常简单,接收到消息后,判断消息的类型,如果是`Greeting`招呼内容,那么就直接打印消息到控制台.而`Greeter`这个Actor稍微复杂点,它消费两种不同的消息,如果是`WhoToGreet`,那么就把要打招呼的人记录到自己的上下文中,如果是`Greet`,那么就构造出招呼的内容,并把消息反馈回sender.

最后,再来分析下DemoMain.
	
1. 一来,先创建了一个`ActorSystem`,
2. 然后创建了一个`Greeter`Actor的实例,命名为`greeter`.
3. 接着,为这个Actor,显示的创建了一个`邮箱`.
4. 而后,调用`greeter.tell(new WhoToGreet("akka"), ActorRef.noSender());`,表示给greeter这个Actor发送一个消息,消息的内容是`WhoToGreet`,发送者是空.这就意味着在greeter这个Actor内部,调用sender是不能获取到发送者的.通过这个动作,就把消息限定为了单向的.
5. 再然后,通过`inbox.send(greeter, new Greet());`,使用邮箱显示的发送一个Greet消息给greeter.这是给Actor发送消息的另外一种方法,这种方法通常会有更高的自主性,能完成更多更复杂的操作.但是调用起来比直接使用`ActorRef`来的复杂.
6. `Greeting greeting1 = (Greeting) inbox.receive(Duration.create(5, TimeUnit.SECONDS));`表示的就是尝试在5秒钟内,从`Inbox`邮箱中获取到反馈消息.如果5秒内没有获取到,那么就抛出`TimeoutException`异常. 由于我们在greeter这个Actor中有处理,接收到`Greet`消息后,就构造一个`Greeting`消息给`sender`,因此这个地方是能够正确的获取到消息的反馈的.
7. 后面的操作都是一样的,就不再重复描述.
8. 只有最后一个代码稍微有点不一样`system.scheduler().schedule(Duration.Zero(), Duration.create(1, TimeUnit.SECONDS), greeter, new Greet(), system.dispatcher(), greetPrinter);`,这个使用了`ActorSystem`中的调度功能.每一秒钟给greeter这个Actor发送一个`Greet`消息,并指定消息的发送者是`greetPrinter`.这样每隔一秒钟,greeter就会收到`Greet`消息,然后构造成`Greeting`消息,又返回给`GreetPrinter`这个Actor,这个Actor接收到消息后,打印出来.形成一个环流.

