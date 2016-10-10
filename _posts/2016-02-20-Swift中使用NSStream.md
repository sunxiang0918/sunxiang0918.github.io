---
title: Swift中使用NSStream
date: 2016-02-20 15:11:41
tags:
- JAVA
---

#Swift中使用NSStream

因为我的开源项目[zkClient4Swift](https://github.com/sunxiang0918/zkClient4Swift)的需要,要在Swift连接Socket,因此涉及到了使用NSStream来进行网络的流交互.特此把使用的过程整理出来供大家参考.

##流交互
通常在跨语言的交互中,由于语言对于数据结构的存储是不一样的,因此,会使用流的方式来进行交互.流交互方式与语言和设备无关.流是在通信隧道中串行传输的连续的比特位序列.从编码的角度来说,流是单向同步的.因此,一般流都分为了输入流(InputStream)和输出流(OutputStream).这些流的数据通常只能使用一次,消耗完后,就无法从流对象中再次的获取或写入.

##Swift中的流
在Swift中,与流相关的主要是三个类:`NSStream`,`NSInputStream`,`NSOutputStream`.除开`NSStream`是一个抽象的基类外,剩下的两个类分别对应了输入流和输出流的所有属性和操作.

在`NSInputStream`或`NSOutputStream`中,可以对`文件`,`Socket`,`NSData`中获取数据.由于我这次是进行网络交互,因此,主要说下针对`Socket`进行的操作.

`NSStream`对象中有一个属性`delegate`用来指定流事件的代理对象,这个也是整个`NSStream`最总要的方法.我们可以自己实现`NSStreamDelegate`协议并复制给`NSInputStream`或`NSOutputStream`的`delegate`属性,每当有流事件的时候,就会调用`NSStreamDelegate`协议的`public func stream(aStream: NSStream, handleEvent eventCode: NSStreamEvent)`方法实现,从而处理流相关的所有处理.

而对于输入流来说,我们可以使用`public func read(buffer: UnsafeMutablePointer<UInt8>, maxLength len: Int) -> Int`方法来获取数据.
而对于输出流来说,我们可以使用`public func write(buffer: UnsafePointer<UInt8>, maxLength len: Int) -> Int`方法来写入数据.

对于`NSStreamDelegate`协议的`public func stream(aStream: NSStream, handleEvent eventCode: NSStreamEvent)`方法,主要是需要了解`NSStreamEvent`事件类型,它们主要有:

* **None** : 
* **OpenCompleted** : 当连接创建完毕时触发
* **HasBytesAvailable** : 当有数据可读取时触发
* **HasSpaceAvailable** : 当可以发送数据时触发
* **ErrorOccurred** : 当出现错误时触发
* **EndEncountered** : 当连接结束时触发

<!--more-->

##使用NSInputStream读取数据
在Swift中使用`NSInputStream`读取数据主要有两种方式,一种是使用`NSStreamDelegate`的事件代理机制,当接收到`NSStreamEvent.HasBytesAvailable`事件的时候,在事件的响应中使用`read`方法来获取. 另外一种是直接启动一个永远循环的线程,然后在线程中不断的调用`read`方法来获取数据即可.一般来说我们推荐使用第一种方式.

对于第一种方式读取数据主要有以下几个步骤:

1. 从数据源中创建和初始化一个NSInputStream
2. 将InputStream实例放入一个runloop中,并打开流
3. 处理流对象的事件代理
4. 当完成数据读取时,关闭并销毁流对象

下面我们分别来说:

###从数据源中创建和初始化一个NSInputStream
由于我们这篇文章主要讲的是在Socket中使用NSStream.那么我们的数据源就是Socket.

创建Socket的事件流的方式为:

```swift
private var inputStream:NSInputStream?
private var outputStream:NSOutputStream?

NSStream.getStreamsToHostWithName(addr, port: port, inputStream: &inputStream, outputStream: &outputStream)
```

因为`NSStream`其实还是Objective-C的方法签名.因此,后面的inputStream和outputStream的方法入参是类似于`C`语言方式的指针传入.
调用了这个方法后,如果inputStream就应该会有值了,如果没有值,那么就说明Socket没有打开:

```swift
guard let inputStream = inputStream,outputStream = outputStream else {
//这里说明没有开启流
return (false,"Can not open stream")
}
```

然后就是设置事件的代理:

```swift
inputStream.delegate = self
```

###将InputStream实例放入一个runloop中,并打开流
由于需要监听InputStream的事件,并异步的进行处理.因此,就需要在其他线程上异步的注册事件处理的操作.这个步骤主要有两种方式来实现:

####使用`scheduleInRunLoop:forMode:`
这种方式可能是最常用的方式了,就是直接调用`NSStream`的`public func scheduleInRunLoop(aRunLoop: NSRunLoop, forMode mode: String)`方法,把事件回调的代理绑定到某一个RunLoop上.通常的写法为:

```swift
let loop = NSRunLoop.currentRunLoop()
inputStream.scheduleInRunLoop(loop, forMode: NSDefaultRunLoopMode)
```

这样的话,他就会在当前线程的RunLoop来执行这个`schedule`.同时,由于一般来说,当前线程都是主线程,那么如果在主线程来进行监听的话.进行命令处理的时候会把主线程阻塞.因此,就需要在调用`scheduleInRunLoop`的时候启动子线程,让inputStream的`schedule`在子线程中进行监听:

```swift
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0)) { () -> Void in
let loop = NSRunLoop.currentRunLoop()
inputStream.scheduleInRunLoop(loop, forMode: NSDefaultRunLoopMode)
inputStream.open()
loop.run()
        }
```
需要注意的是最后一句的`loop.run()`,这个是必须要要的.否则不能阻塞新的异步队列.

采用这种方式存在一个问题,那就是如果要想关闭`NSStream`的时候,是没有办法关闭启动的异步线程的.由于我们在异步线程的内部没有进行循环的操作,而是使用`loop.run`的方式来阻塞的.所以,那怕是使用标志位的方式都是不能结束这个异步loop的.


####使用`CFReadStreamSetDispatchQueue`
针对上一种方式无法关闭的问题,我们可以直接使用`CFStream`中提供的`public func CFReadStreamSetDispatchQueue(stream: CFReadStream!, _ q: dispatch_queue_t!)`方法来给`NSStream`指定异步的队列:

```swift
let sharedWorkQueue = dispatch_queue_create("socketWork.queue", DISPATCH_QUEUE_CONCURRENT)

CFReadStreamSetDispatchQueue(inputStream, sharedWorkQueue)
```
它同样可以达到和`RunLoop`相同的效果.并且他的优势是由于没有显示的启动新的异步线程,使用了异步队列的方式,在我们关闭`NSStream`的时候就可以做相反的操作即可.

###处理流对象的事件代理
这个就是实现`func stream(aStream: NSStream, handleEvent eventCode: NSStreamEvent)`方法即可.我们可以通过`switch`语法来分别的处理`eventCode`.然后通过调用`NSStream`的`streamStatus`来获取流对象的状态,以及调用`streamError`来获取流对象可能存在的错误.

```swift
public func stream(aStream: NSStream, handleEvent eventCode: NSStreamEvent) {
        switch (eventCode) {
            case NSStreamEvent.HasSpaceAvailable:
                reconnectionLock.readLock()
                defer{reconnectionLock.readUnlock()}
                hasSpaceAvailableDelegate?(aStream)
                break
            case NSStreamEvent.HasBytesAvailable:
                reconnectionLock.readLock()
                defer{reconnectionLock.readUnlock()}
                hasBytesAvailableDelegate?(aStream)
                break
            case NSStreamEvent.EndEncountered:  
                connected = false
                endEncounteredDelegate?(aStream)
                break
            case NSStreamEvent.ErrorOccurred:
                errorOccurredDelegate?(aStream)
                break
            case NSStreamEvent.OpenCompleted:
                if aStream is NSInputStream {
                    inputLock.writeUnlock()
                }
                if aStream is NSOutputStream {
                    outputLock.writeUnlock()
                }
                break
        default:
            print("接收到事件:\(eventCode) :\(aStream)")
            break;
        }
    }
```
我这个例子里面使用了闭包来处理事件的响应,其实完全是可以把事件的响应直接写到这个函数中的.

###从NSInputStream中读取数据
从NSInputStream中读取数据需要使用`public func read(buffer: UnsafeMutablePointer<UInt8>, maxLength len: Int) -> Int`方法,他的入参是一个[UInt8]的指针以及期望读取的长度,返回值是实际读取的字节数.因此,我们通常需要预先的初始化一个`UInt8`的数组,并根据`maxLength`赋初始值,然后调用`read`方法来填充这个`UInt8`的数组,并进行返回:

```swift
var buff:[UInt8] = [UInt8](count:expectlen,repeatedValue:0x0)
        
let len = inputStream.read(&buff, maxLength: expectlen)
        
if len > 0 {
   let result = buff[0..<len]
   return Array(result)
}
        
print("读取出来的len为0,错误:\(inputStream.streamError)")
        
return nil
```

当然,在Swift中,`NSData`类型通常会比`[UInt8]`类型有更多的使用场景,那么就牵涉到`[UInt8]`和`NSData`类型之间的转换了.由于`[UInt8]`其实就是一个字节数组,因此完全可以使用`NSData(bytes:UnsafePointer<Void>,length:Int)`的方式进行初始化的,为了方便书写,我们可以写一个NSData的扩展:

```swift
public extension NSData {
	convenience init(var uints:[UInt8]) {
        self.init(bytes:&uints,length:uints.count)
    }
}
```
这样就可以把从`NSInputStream`中获取的`[UInt8]`转换成`NSData`了:

```swift
guard let uints = _connection.read(102400, timeout: _sessionTimeout) else {
      print("读取错误")
      continue
}
            
let data = NSData(uints: uints)
```

###当完成数据读取时,关闭并销毁流对象
当需要关闭流的时候,只需要反向进行操作即可. 但是对于使用`RunLoop`的方式进行监听事件的来说,是无法跳出`RunLoop`的. 

```swift
public func close()->(Bool,String){
        _closed = true
        
        outputStream?.delegate = nil
        inputStream?.delegate = nil
        if let stream = inputStream {
            CFReadStreamSetDispatchQueue(stream, nil)
            stream.close()
        }
        if let stream = outputStream {
            CFWriteStreamSetDispatchQueue(stream, nil)
            stream.close()
        }
        outputStream = nil
        inputStream = nil
        
        self.connected = false
        
        return (true,"")
}
```
如果使用的是`RunLoop`的方式,那么就把代码中的`CFReadStreamSetDispatchQueue(stream, nil)`修改为:`stream.removeFromRunLoop(NSRunLoop.currentRunLoop(),forMode:NSDefaultRunLoopMode)`即可

##使用NSOutputStream写入数据
NSOutputStream写入数据和使用NSInputStream读取数据其实在大体上是一模一样的,他们两个通常都会成对的出现,同样是有以下几个步骤:

1. 从数据源中创建和初始化一个NSOutputStream
2. 将NSOutputStream实例放入一个runloop中,并打开流
3. 处理流对象的事件代理
4. 当完成数据写入后,关闭并销毁流对象

具体的步骤细节这就不说了,我们主要介绍一下不同的地方:

1. 在`stream:handleEvent:`方法中,主要是监听的`HasSpaceAvailable`事件.
2. 写入数据的方法为:`public func write(buffer: UnsafePointer<UInt8>, maxLength len: Int) -> Int`,他的入参同样是`[UInt8]`的指针以及`maxLength`
3. 绑定异步队列时使用`CFWriteStreamSetDispatchQueue`方法

同样,在swift中我们更多的是使用`NSData`或`String`来表示数据,这同样会有`NSData`或`String`到`[UInt8]`的转换:

```swift
	public func send(var data d:[UInt8])->(Bool,String){
        outputLock.readLock()
        defer{outputLock.readUnlock()}
        
        guard let outputStream = outputStream else {
            return (false,"outputStream is closed")
        }
        let len = outputStream.write(&d, maxLength: d.count)
        return len==d.count ? (true,"") : (false,"send error")
    }
    
    
    public func send(str s:String)->(Bool,String){
        
        guard let data = s.dataUsingEncoding(NSUTF8StringEncoding) else {
            return (false,"string format error")
        }
        
        return send(data: data)
    }
    
    public func send(data d:NSData)->(Bool,String){
        var buff:[UInt8] = [UInt8](count:d.length,repeatedValue:0x0)
        d.getBytes(&buff, length: d.length)
        return send(data: buff)
    }
```

##流的错误处理
当流出现错误的时候,会停止对数据的处理.
在`NSStream`中可以由以下的几种方式来知道错误的发生:

1. 如果流被绑定到`RunLoop`或使用`CFReadStreamSetDispatchQueue`来打开.那么,可以在`stream:handleEvent:`方法中监听并处理`ErrorOccurred`事件.
2. 在任何时候,都可以调用`streamStatus`和`streamError`属性来获取错误.
3. 如果在调用`write:maxLength:`或者`read:maxLength:`方法写入或读取的实际数量为-1时,则表示发生了一个错误.

##流的其他设置
当需要在连接中使用SSL等的时候,需要对`NSInputStream`和`NSInputStream`的属性进行特别的设置.在`NSStream`中有`public func setProperty(property: AnyObject?, forKey key: String) -> Bool`方法来给`NSStream`的连接添加上特别的属性.它要求是在调用`open`方法前使用.

比如如果要创建SSL连接,就可以这样设置:

```swift
inputStream.setProperty(NSStreamSocketSecurityLevelNegotiatedSSL, forKey: NSStreamSocketSecurityLevelKey)
let settings: [NSObject: NSObject] = [kCFStreamSSLValidatesCertificateChain: NSNumber(bool:false), kCFStreamSSLPeerName: kCFNull]
inputStream.setProperty(settings, forKey: kCFStreamPropertySSLSettings as String)
```

同样,如果需要获取连接中的特殊属性,可以使用`public func propertyForKey(key: String) -> AnyObject?`方法

##总结
本文介绍了`NSStream`的常用方法.以及通过例子,讲解了具体如何的创建,使用,销毁`NSStream`.
如果有其他的使用细节,我们会在后面再详细讨论.


