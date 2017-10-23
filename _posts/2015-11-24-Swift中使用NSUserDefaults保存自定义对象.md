---
title: Swift中使用NSUserDefaults保存自定义对象
date: 2015-11-24 11:32:07
tags:
- Swift
toc: false
---

# Swift中使用NSUserDefaults保存自定义对象

NSUserDefaults适合存储轻量级的本地客户端数据,比如保存一些系统的基本配置等东西,使用NSUserDefaults是首选,它非常的简单,不依赖其他的什么东西,屏蔽了Plist文件的读写等等.

但是NSUserDefaults支持的数据格式比较有限,只支持了`Int`、`Float`、`Double`，`String`，`NSDate`，`NSArray`，`NSDictionary`，`Bool`.不支持自定义对象的存取.

这个特性对于简单的数值来说还算比较容易.只需要简单的操作(一个Value 一个Key ),例如，想要保存一个NSString的对象，代码实现为：

```swift
//存
NSUserDefaults.standardUserDefaults().setInteger(10, forKey: "aaa")
//取
NSUserDefaults.standardUserDefaults().integerForKey("aaaa")
```

<!--more-->
但是涉及到复杂的对象的时候,就需要我们进行特殊的处理了.

其实这个特殊处理也很简单,既然`NSUserDefaults`支持`NSDate`类型的数据.那么我们在存取自定义对象的时候,就可以预先把我们的自定义对象转换成为`NSData`类型的即可.

在Swift中如果要把一个自定义对象能转换成为`NSData`.需要在自定义对象上实现`NSObject`和`NSCoding`协议,并且实现`func encodeWithCoder(aCoder: NSCoder)`和`init?(coder aDecoder: NSCoder)`方法,比如:

```swift
import Foundation

class SiteInfoVO : NSObject,NSCoding {
    
    let url:String
    
    var userName:String?
    
    var password:String?
    
    init(url:String){
        self.url = url
    }
    
    @objc internal func encodeWithCoder(aCoder: NSCoder) {
        aCoder.encodeObject(url, forKey: "url")
        aCoder.encodeObject(userName, forKey: "userName")
        aCoder.encodeObject(password, forKey: "password")
    }
    
    @objc internal required init?(coder aDecoder: NSCoder) {
        url = aDecoder.decodeObjectForKey("url") as! String
        
        userName = aDecoder.decodeObjectForKey("userName") as? String
        password = aDecoder.decodeObjectForKey("password") as? String
    }
    
    
}
```

这样就可以使用`NSKeyedUnarchiver`类来进行转换了.

那么保存一个自定义对象到`NSUserDefaults`就变为了:

```swift
let _value = SiteInfoVO(url:"123")
let modelData:NSData = NSKeyedArchiver.archivedDataWithRootObject(_value)
NSUserDefaults.standardUserDefaults().setObject(modelData, forKey: "defaultName")
```

而读取一个自定义对象就成为了:

```swift
let data = NSUserDefaults.standardUserDefaults().objectForKey("defaultName") as? NSData

if let _data = data {
   let model = NSKeyedUnarchiver.unarchiveObjectWithData(_data)
}
```

当然我们可以给`NSUserDefaults`增加一个扩展,把编解码自定义对象进行一次封装:


```swift
import Foundation

public extension NSUserDefaults {
    
    public func modelForKey(defaultName: String) -> AnyObject? {
        let obj = self.objectForKey(defaultName) as? NSData
        
        if let tmp = obj {
            return NSKeyedUnarchiver.unarchiveObjectWithData(tmp)
        }
        
        return nil
    }
    
    public func arrayModelForKey(defaultName: String) -> [AnyObject]? {
        let obj = self.objectForKey(defaultName) as? [NSData]
        
        var result:[AnyObject]?
        
        if let _obj = obj {
            result = []
            for tmp in _obj {
                let myModel = NSKeyedUnarchiver.unarchiveObjectWithData(tmp)
                result?.append(myModel!)
            }
            return result
        }
        return nil
    }
    
    public func setModel(value: AnyObject?, forKey defaultName: String){
        
        guard let _value = value else{
            self.setObject(nil, forKey: defaultName)
            return
        }
        
        let modelData:NSData = NSKeyedArchiver.archivedDataWithRootObject(_value)
        self.setObject(modelData, forKey: defaultName)
    }
    
    public func setArrayModels(value: [AnyObject]?, forKey defaultName: String) {
        guard let _value = value else{
            self.setObject(nil, forKey: defaultName)
            return
        }
        
        var data:[NSData] = []
        
        for v in _value {
            data.append(NSKeyedArchiver.archivedDataWithRootObject(v))
        }
        
        self.setObject(data, forKey: defaultName)
    }
}
```

PS:按照这个思路,其实把自定义对象转换成JSON字符串等都是可以的.

