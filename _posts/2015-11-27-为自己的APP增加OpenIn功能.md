title: 为自己的APP增加OpenIn功能
date: 2015-11-27 09:55:25
tags:
- Swift
toc: false
---

#为自己的APP增加OpenIn功能

在做App的时候,可能会遇到你的程序可以打开某种类型的文档.比如用户在safari中下载了一个torrent类型的文件,然后需要直接使用你写的App打开.那么这个时候就需要在编写App的时候进行文档类型的注册了.

![](/img/2015/11/27/1.png)

<!--more-->

##1.在Info.plist中注册文档类型

首先要做的就是在Info.plist中增加文档类型的注册,表示这个程序能处理哪些文档.

这里有两种方式,一种是通过项目的Info栏直接增加`Document Types`和`Exported UTIs`.
就像这样:

![](/img/2015/11/27/3.png)

还有一种就是直接修改info.plist文件:

![](/img/2015/11/27/2.png)

其xml为:

```xml
<key>CFBundleDocumentTypes</key>
	<array>
		<dict>
			<key>LSItemContentTypes</key>
			<array>
				<string>cn.sunxiang0918.transmission.torrent</string>
			</array>
			<key>CFBundleTypeRole</key>
			<string>Viewer</string>
			<key>CFBundleTypeName</key>
			<string>torrent file</string>
			<key>LSHandlerRank</key>
			<string>Owner</string>
		</dict>
	</array>
	<key>UTExportedTypeDeclarations</key>
	<array>
		<dict>
			<key>UTTypeConformsTo</key>
			<array>
				<string>public.data</string>
			</array>
			<key>UTTypeIdentifier</key>
			<string>cn.sunxiang0918.transmission.torrent</string>
			<key>UTTypeTagSpecification</key>
			<dict>
				<key>public.mime-type</key>
				<string>application/torrent</string>
				<key>public.filename-extension</key>
				<string>torrent</string>
			</dict>
		</dict>
	</array>
```

这里稍作解释.
* **CFBundleDocumentTypes**:为注册文档类型以及角色
* **UTExportedTypeDeclarations**:为注册处理类型与方式
* **LSItemContentTypes**:自己定义的一种文档的唯一串
* **CFBundleTypeName**:文档的类型名字
* **UTTypeTagSpecification**:指定文档的类型以及后缀名

通过这个的指定,现在程序就能关联到某种文档文件了.

##处理文档的打开操作

有了与文档的关联后,还需要进行的操作就是当用户指定APP打开文档后需要执行的操作了.

~~在`AppDelegate`这个程序的入口类中,有一个方法:`func application(application: UIApplication, openURL url: NSURL, sourceApplication: String?, annotation: AnyObject) -> Bool`.
我们只要重载这个方法就可以了.
这个方法中最总要的两个入参就是`openURL url: NSURL`以及`sourceApplication: String?`
它说明了是哪一个其他程序传过来的文档,以及文档目前的路径是什么.
当有了这个参数后,我们就可以直接读取文档的内容了.~~`IOS9废弃`

在`AppDelegate`这个程序的入口类中,有一个方法:`func application(application: UIApplication, openURL url: NSURL, options: [String : AnyObject]) -> Bool`.
我们只要重载这个方法就可以了.
这个方法中最重要的两个入参就是`openURL url: NSURL`以及`options: [String : AnyObject]`
它说明了是哪一个其他程序传过来的文档,以及文档目前的路径是什么.
当有了这个参数后,我们就可以直接读取文档的内容了.

```swift
func application(application: UIApplication, openURL url: NSURL,options: [String : AnyObject]) -> Bool {
        
        let encrypteddata = NSData(contentsOfURL: url)
        
        let base64 = encrypteddata!.base64EncodedStringWithOptions(NSDataBase64EncodingOptions(rawValue: 0))
        do {
            //尝试删除文件
            try NSFileManager.defaultManager().removeItemAtURL(url)
        } catch let e {
            print(e)
        }
                
        return true
    }
```

通过`NSData(contentsOfURL: url)`可以读取文档的二进制的内容.
文档读取后可以通过`NSFileManager.defaultManager().removeItemAtURL(url)`删除磁盘上的文档.
最后返回一个true,表示是这个程序是能处理这个文档的.

到此,我们的自己的程序就能打开任意的文档并进行相应的处理了.