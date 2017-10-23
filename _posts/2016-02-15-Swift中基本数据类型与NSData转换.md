---
title: Swift中基本数据类型与NSData转换
date: 2016-02-15 18:01:21
tags:
- Swift
---

# Swift中基本数据类型与NSData转换

最近由于程序的需要,要与JAVA的服务端进行Socket的交互,那么这就牵涉到了数据的交互.Socket的数据交互一般都是直接采用二进制Bytes的方式来传递,那么就需要把Swift中的各种基本数据转换成为JAVA服务器可以认可的Bytes字节数组,以及把JAVA的字节数组反序列化为Swift中的基本数据.

## big-endian and little-endian
要在不同程序中进行字节数组的数据交换,有个很重要的东西就是`字节序`.`字节序`顾名思义就是字节的顺序,也就是大于1个字节类型的数据在内存中存放的顺序.这个在跨平台以及网络程序交互中非常的重要.

常见的字节序主要有两类:`Big-Endian`和`Little-Endian`.它们的定义为:

* Big-Endian:高位字节排放在内存的低地址端,地位字节排放在内存的高地址端.
* Little-Endian:地位字节排放在内存的低地址端,高位字节排放在内存的高地址端.
这样说可能比较抽象,我们来举个例子就非常清楚了:
比如一个32位的Int类型数据: `let a = Int32(2)`分别采用`Big-Endian`和`Little-Endian`的情况如下:

|字节号|0|1|2|3|
|:--:|:--:|:--:|:--:|:--:|
|Big-endian|00|00|00|02|
|Little-Endian|02|00|00|00|

也就是他们两个是相反的.`Big-Endian`和`Little-Endian`跟CPU的指令有关,每一种CPU不是`Big-Endian`就是`Little-Endian`.常见的IA架构的CPU,比如Intel或AMD的都是使用的`Little-Endian`,而PowerPC活着SPARC的处理器则是`Big-Endian`的.而在互联网的网络交互以及TCP协议中使用的是`Big-Endian`,JAVA的虚拟机中的字节序是`Big-Endian`的.而Swift由于是运行在IA架构的CPU上的,因此,它的字节序是`Little-Endian`的.

正是由于运行在X86上的`Swift`和`JAVA`的字节序是相反的,因此,它们两个进行跨语言的网络数据交互的时候,就需要对数据进行字节序的转换.否则就会出现数据读取错误的情况,比如用`JAVA`采用`Big-Endian`序的Int32`02000000`,`Swift`采用`Little-Endian`序解析出来是`33554432`而不是期望的`2`.

<!--more-->

在Swift中,Apple在`CoreFoundation`中提供了一些列的函数来提供字节序的转换.它们都在`CFByteOrder`中有所定义:

```swift
public func CFSwapInt16BigToHost(arg: UInt16) -> UInt16

public func CFSwapInt32BigToHost(arg: UInt32) -> UInt32

public func CFSwapInt64BigToHost(arg: UInt64) -> UInt64

public func CFSwapInt16HostToBig(arg: UInt16) -> UInt16

public func CFSwapInt32HostToBig(arg: UInt32) -> UInt32

public func CFSwapInt64HostToBig(arg: UInt64) -> UInt64

public func CFSwapInt16LittleToHost(arg: UInt16) -> UInt16

public func CFSwapInt32LittleToHost(arg: UInt32) -> UInt32

public func CFSwapInt64LittleToHost(arg: UInt64) -> UInt64

public func CFSwapInt16HostToLittle(arg: UInt16) -> UInt16

public func CFSwapInt32HostToLittle(arg: UInt32) -> UInt32

public func CFSwapInt64HostToLittle(arg: UInt64) -> UInt64

public func CFConvertFloat32HostToSwapped(arg: Float32) -> CFSwappedFloat32

public func CFConvertFloat32SwappedToHost(arg: CFSwappedFloat32) -> Float32

public func CFConvertFloat64HostToSwapped(arg: Float64) -> CFSwappedFloat64

public func CFConvertFloat64SwappedToHost(arg: CFSwappedFloat64) -> Float64

public func CFConvertFloatHostToSwapped(arg: Float) -> CFSwappedFloat32

public func CFConvertFloatSwappedToHost(arg: CFSwappedFloat32) -> Float

public func CFConvertDoubleHostToSwapped(arg: Double) -> CFSwappedFloat64

public func CFConvertDoubleSwappedToHost(arg: CFSwappedFloat64) -> Double
```
使用这些函数就可以对字节序进行转换. 更多的可以参考[Apple的官方手册](https://developer.apple.com/library/prerelease/ios/documentation/CoreFoundation/Reference/CFByteOrderUtils/index.html)

**2016-02-16 Update**
最新的swift中,对`UInt64`和`UInt32`,已经自带了成员方法:`public var bigEndian: UInt32 { get }` `public var littleEndian: UInt32 { get }`等等. 也就是说,不需要使用`CFSwapInt32BigToHost(val)`这样转换了,直接`val.bigEndian`即可.

## 基础数据与NSData的转换
为了能让Swift和JAVA进行网络的交互,那么就必须把它们的基础数据转换成为Bytes字节数组.
在Swift中使用`NSData`或者`NSMutableData`来表示.因此,也就是需要把基础数据放入到NSData中.

### Int32与Int64
在Swift中,`Int`是一个特殊的整数类型,它的长度与当前平台的原生字长相同:

* 在32位平台上,`Int`与`Int32`长度相同
* 在64位平台上,`Int`与`Int64`长度相同.相当于C中的Long.

因此,在网络传输的时候,是需要区分的对待Int32和Int64的.需要把一个`Int`类型强转为需要的长度.
具体的代码为:

```swift
public extension NSMutableData {
    
    public func appendInt(value:Int){
        var networkOrderVal = CFSwapInt32HostToBig(UInt32(value))
        self.appendBytes(&networkOrderVal, length: sizeof(UInt32))
    }
    
    public func appendLong(value:Int) {
        var networkOrderVal = CFSwapInt64HostToBig(UInt64(value));
        self.appendBytes(&networkOrderVal, length: sizeof(UInt64))
    }
    
    public func getInt(range:NSRange = NSRange(location:0,length:sizeof(UInt32))) -> Int {
        var val: UInt32 = 0
        self.getBytes(&val, range: range)
        return Int(CFSwapInt32BigToHost(val))
    }
    
    public func getLong(range:NSRange = NSRange(location:0,length:sizeof(UInt64))) -> Int {
        var val: UInt64 = 0
        self.getBytes(&val, range: range)
        return Int(CFSwapInt64BigToHost(val))
    }
}
```
这里使用了扩展机制,直接在`NSMutableData`上增加扩展.

首先,使用`CFSwapInt32HostToBig`函数把字节序给改变了.然后调用`NSMutableData.appendBytes`方法,赋值给`NSData`.由于`NSMutableData`现在还是`Objective-C`的实现,因此,调用方式稍微有点奇怪,是使用指针的方式进行赋值的.更多的可以参见[Using Swift with Cocoa and Objective-C](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/BuildingCocoaApps/InteractingWithCAPIs.html).

反序列化也是相似的,首先定义了一个变量.然后使用`NSMutableData.getBytes`,同样传入一个指针以及数据的返回.然后最后在进行一次字节序的转换即可.

**2016-02-16 Update:**
对于需要在网络传输中传输负数的情况需要先把负数的`Int`转换为无符号的整数`UInt`.在计算机中,负数的表示方法是采用补码的形式.在swift中,可以使用`UInt32(bitPattern:Int32)`以及`Int32(bitPattern:UInt32)`方法来相互的转换.比如,`-5`转换为无符号的补码形式为:`fffffffb`. 因此我们的`appendInt`和`getInt`可以改成这样:

```swift
	public func appendInt(value:Int){
        var networkOrderVal = UInt32(bitPattern:Int32(value)).bigEndian
        self.appendBytes(&networkOrderVal, length: sizeof(UInt32))
    }
    
    public func appendLong(value:Int) {
        var networkOrderVal = UInt64(bitPattern:Int64(value)).bigEndian
        self.appendBytes(&networkOrderVal, length: sizeof(UInt64))
    }
    
    public func getInt(range:NSRange = NSRange(location:0,length:sizeof(UInt32))) -> Int {
        var val: UInt32 = 0
        self.getBytes(&val, range: range)
        return Int(Int32(bitPattern:val.bigEndian))
    }
    
    public func getLong(range:NSRange = NSRange(location:0,length:sizeof(UInt64))) -> Int {
        var val: UInt64 = 0
        self.getBytes(&val, range: range)
        return Int(Int64(bitPattern:val.bigEndian))
    }
```

### Float32与Float64
它的情况与`Int`的非常相似.同样需要经历字节序的转换,以及`NSMutableData.appendBytes`的调用.

```swift
public extension NSMutableData {
    
    // MARK: Float32与Float64
    
    public func appendFloat(value:Float) {
        var networkOrderVal = CFConvertFloat32HostToSwapped(Float32(value))
        self.appendBytes(&networkOrderVal, length: sizeof(Float32))
    }
    
    public func appendDouble(value:Double) {
        var networkOrderVal = CFConvertFloat64HostToSwapped(Float64(value))
        self.appendBytes(&networkOrderVal, length: sizeof(Float64))
    }
    
    
    public func getFloat(range:NSRange = NSRange(location:0,length:sizeof(Float32))) -> Float {
        
        var val: CFSwappedFloat32 = CFSwappedFloat32(v: 0)
        self.getBytes(&val, range: range)
        let result = CFConvertFloat32SwappedToHost(val)
        return result
    }
    
    public func getDouble(range:NSRange = NSRange(location:0,length:sizeof(Float64))) -> Double {
        var val: CFSwappedFloat64 = CFSwappedFloat64(v: 0)
        self.getBytes(&val, range: range)
        let result = CFConvertFloat64SwappedToHost(val)
        return result
    }
}
```

### Bool
`Bool`类型由于是单字节的数据,不存在字节序的问题.因此,它与NSData的转换最为简单.

```swift
public extension NSMutableData {
	// MARK: Bool
    public func appendBool(var val:Bool){
        self.appendBytes(&val, length: sizeof(Bool))
    }
    
    public func getBool(range:NSRange = NSRange(location:0,length:sizeof(Bool))) -> Bool {
        var val:Bool = false
        self.getBytes(&val, range: range)
        return val
    }
}
```

### String
字符串的处理又相对的要麻烦些了.因为字符串的长度是可变的.不像其他的数据类型是有固定的长度的.因此,一般在网络传输中,都会在字符串的bytes前接上一个`Int32`的字节数组来表示这个字符串的长度.

因此,我们在转换`String`到`NSData`的时候,实际上是两个步骤.首先计算出字符串的字节长度.然后把这个字节长度放入NSData中,接着,再把字符串的内容转换为字节数组放入NSData中:

```swift
public extension NSMutableData {
	// MARK: String
    public func appendString(val:String,encoding:NSStringEncoding = NSUTF8StringEncoding){
        
        //获取到字节的长度,使用某一种编码
        let pLength : Int = val.lengthOfBytesUsingEncoding(encoding)

        //放入字符串的长度
        self.appendInt(pLength)
        
        //把字符串按照某种编码转化为字节数组
        let data = val.dataUsingEncoding(encoding)
        
        //放入NSData中
        self.appendData(data!)
    }

    public func getString(location:Int = 0,encoding:NSStringEncoding = NSUTF8StringEncoding) throws -> (String,Int){
        
        //先获取到长度
        let len = self.getInt(NSRange(location:location,length:sizeof(UInt32)))
        
        //找到子字节数组
        let subData = self.subdataWithRange(NSRange(location: location+sizeof(UInt32), length: len))
        
        //直接使用String的构造函数,采用某种编码格式获取字符串
        let string = String(data: subData, encoding: encoding)
        
        //如果凑不起字符串,就表示数据不正确,那么就抛出异常
        guard let _string = string else {
            throw AppException.FormatCastException
        }
        
        //返回结果
        return (_string,len+sizeof(UInt32))
    }
}
```

## 总结
以上就是简单的介绍了在Swift中如何把几种常用的数据类型转换为网络交互格式的Bytes数组的.至于其他的数据类型,或者自定义的数据结构,无外乎都是从这几种基础数据类型上拼接出来的,稍微灵活修改下即可.


