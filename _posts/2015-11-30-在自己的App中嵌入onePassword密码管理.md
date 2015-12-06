---
title: 在自己的App中嵌入onePassword密码管理
date: 2015-11-30 19:36:31
tags:
- Swift
---

#在自己的App中嵌入onePassword密码管理

[1Password](https://agilebits.com/onepassword/)是一个密码管理软件,它可以方便和安全的管理你的密码.它提供反钓鱼保护功能和卓越的密码管理,并具有自动生成强密码功能.所有的机密资料,包括密码,身份卡和信用卡,都是保存在一个安全的地方.在OSX和IOS系统上非常的流行.在我的博文[MacOS下JAVA开发使用经验(一)](/2015/09/21/MacOS下JAVA开发使用经验(一) )也有介绍.  
自从我的账号被CSDN泄露的干干净净以后的一段时间里,我的其他账号是不是的就会收到更改密码的邮件.于是,一下狠心购买了`1Password`,并且把我所有的差不多200多个账号的密码通过`1Password`都给改成了14位的随机字符,以后就不用担心一个密码丢失导致其他的账号也丢失了(1Password自己本身在它的网上没有所谓的用户名密码,密码是通过加密文件整体在WIFI或iCloud中同步的.).但是这就带来了一个麻烦,就是我的密码都变成了类似于这样的`RxCa9vdBUB3fCU`的字符串.当使用safari这些的时候还好,它有浏览器的插件可以自动填充.遇到一些App,就只能手动的输入了(类似于光大银行和QQ不允许粘贴密码).这相当的麻烦,又容易出错.因此,自己开发APP的时候,就特别的注意了在输入密码的地方要与`1Password`的整合,这点网易系的APP就做的比较好,`网易云音乐`和`考拉海淘`就是支持了`1Password`的.  
要在自己的App中引入`1Password`其实也不麻烦,可以说是相当的简单.`1Password`在`GitHub`上开源了App与它的软件交互的扩展[1PasswordExtension](https://github.com/AgileBits/onepassword-app-extension).我们通过这个扩展就可以在自己的App中嵌入`1Password`的密码管理了.

<!--more-->

##导入1PasswordExtension扩展

这个可以使用`cocoaPods`. 在项目的`Pods`文件中增加`pod '1PasswordExtension'`,然后执行`pod install`即可.由于它已经兼容了`swift`了,所以不需要特别的再链接OC的头文件了.

##使用1Password进行登陆框的填充
这个过程也非常的简单.

1. 在登录的页面上增加一个按钮,`1PasswordExtension`内置好了它的图片,你可以直接使用.
2. 给这个按钮增加点击的事件.
3. 在Controller类中增加事件的实现,调用`public func findLoginForURLString(URLString: String, forViewController viewController: UIViewController, sender: AnyObject?, completion: (([NSObject : AnyObject]?, NSError?) -> Void)?)`方法:

	```swift
	OnePasswordExtension.sharedExtension().findLoginForURLString("", forViewController: self, sender: sender) { (_loginDictionary, error) -> Void in
				   //loginDictionary中即用调用1Password后,用户选择的用户名和密码
                guard let loginDictionary = _loginDictionary else {
                    return
                }
                if loginDictionary.count == 0 {
                		//如果用户点的是取消,这里就是0
                    if error!.code != Int(AppExtensionErrorCodeCancelledByUser) {
                        //TODO
                    }
                    return
                }
                
                //把用户名和密码赋值给输入框即可.更进一步的可以直接激活登录操作
                view?.userNameField.text = loginDictionary[AppExtensionUsernameKey] as? String
                view?.passwordField.text = loginDictionary[AppExtensionPasswordKey] as? String
                
            }
	```
	效果如图:
	
	![](/img/2015/11/30/1.PNG)
	![](/img/2015/11/30/2.PNG)
	
4. 如果更想进一步,只有当判断到安装了`1Password`程序,按钮才出现的话.可以调用`view?.onePasswordButton.hidden = !OnePasswordExtension.sharedExtension().isAppExtensionAvailable()`.这个方法会判断系统中是否有应用程序能处理`org-appextension-feature-password-management`这个特性.如果有程序可以执行,那么就返回`true`. 需要注意的是:在IOS9中你需要给`Info.plist`文件中增加一行记录:
	![](/img/2015/11/30/3.png)
	更多的信息可以参见[Privacy and Your Apps session](https://developer.apple.com/videos/wwdc/2015/?id=703)

##使用1Password进行新用户的注册
同使用1Password进行用户登陆差不多,新用户的注册也是很简单的.

1. 在登录的页面上增加一个按钮,`1PasswordExtension`内置好了它的图片,你可以直接使用.
2. 给这个按钮增加点击的事件.
3. 在Controller类中增加事件的实现,调用`public func storeLoginForURLString(URLString: String, loginDetails loginDetailsDictionary: [NSObject : AnyObject]?, passwordGenerationOptions: [NSObject : AnyObject]?, forViewController viewController: UIViewController, sender: AnyObject?, completion: (([NSObject : AnyObject]?, NSError?) -> Void)?)`方法

	```swift
	//构造1Password新注册用户页面的一些信息
	let newLoginDetails:[String: AnyObject] = [
			AppExtensionTitleKey: "ACME",
			AppExtensionUsernameKey: self.usernameTextField.text!,
			AppExtensionPasswordKey: self.passwordTextField.text!,
			AppExtensionNotesKey: "Saved with the ACME app",
			AppExtensionSectionTitleKey: "ACME Browser",
			AppExtensionFieldsKey: [
				"firstname" : self.firstnameTextField.text!,
				"lastname" : self.lastnameTextField.text!
				// Add as many string fields as you please.
			]
		]

		//设置密码生成规则
		let passwordGenerationOptions:[String: AnyObject] = [
			// 密码最小长度
			AppExtensionGeneratedPasswordMinLengthKey: (8),
			
			// 最大密码长度
			AppExtensionGeneratedPasswordMaxLengthKey: (30),
			
			// 是否必须包含数字
			AppExtensionGeneratedPasswordRequireDigitsKey: (true),
			
			// 字符必须包含符号
			AppExtensionGeneratedPasswordRequireSymbolsKey: (true),
			
			// Here are all the symbols available in the the 1Password Password Generator:
			// !@#$%^&*()_-+=|[]{}'\";.,>?/~`
			// The string for AppExtensionGeneratedPasswordForbiddenCharactersKey should contain the symbols and characters that you wish 1Password to exclude from the generated password.
			//生成密码的时候排除的字符
			AppExtensionGeneratedPasswordForbiddenCharactersKey: "!@#$%/0lIO"
		]
		
		//调用1Password,打开它的页面输入用户注册信息
		OnePasswordExtension.sharedExtension().storeLoginForURLString("https://www.acme.com", loginDetails: newLoginDetails, passwordGenerationOptions: passwordGenerationOptions, forViewController: self, sender: sender) { (loginDictionary, error) -> Void in
			if loginDictionary == nil {
				if error!.code != Int(AppExtensionErrorCodeCancelledByUser) {
					print("Error invoking 1Password App Extension for find login: \(error)")
				}
				return
			}
			//把用户输入的注册信息返回给界面组件
			self.usernameTextField.text = loginDictionary?[AppExtensionUsernameKey] as? String
			self.passwordTextField.text = loginDictionary?[AppExtensionPasswordKey] as? String
			self.firstnameTextField.text = loginDictionary?[AppExtensionReturnedFieldsKey]?["firstname"] as? String
			self.lastnameTextField.text = loginDictionary?[AppExtensionReturnedFieldsKey]?["lastname"] as? String
		}
	```
	效果如图:
	
	![](/img/2015/11/30/4.PNG)
	![](/img/2015/11/30/5.PNG)
	
##使用1Password进行密码的修改
既然有新用户的注册以及登陆,那么密码管理中还有一个就是密码的修改.

1. 在登录的页面上增加一个按钮,`1PasswordExtension`内置好了它的图片,你可以直接使用.
2. 给这个按钮增加点击的事件.
3. 在Controller类中增加事件的实现,调用`public func changePasswordForLoginForURLString(URLString: String, loginDetails loginDetailsDictionary: [NSObject : AnyObject]?, passwordGenerationOptions: [NSObject : AnyObject]?, forViewController viewController: UIViewController, sender: AnyObject?, completion: (([NSObject : AnyObject]?, NSError?) -> Void)?)`

	```swift
	//构造1Password修改密码页面的一些信息
	let newLoginDetails:[String: AnyObject] = [
			AppExtensionTitleKey: "ACME", // Optional, used for the third schenario only
			AppExtensionUsernameKey: "aUsername", // Optional, used for the third schenario only
			AppExtensionPasswordKey: changedPassword,
			AppExtensionOldPasswordKey: oldPassword,
			AppExtensionNotesKey: "Saved with the ACME app", // Optional, used for the third schenario only
		]
		
		//设置密码生成规则
		let passwordGenerationOptions:[String: AnyObject] = [
			// 密码最小长度
			AppExtensionGeneratedPasswordMinLengthKey: (8),
			
			// 最大密码长度
			AppExtensionGeneratedPasswordMaxLengthKey: (30),
			
			// 是否必须包含数字
			AppExtensionGeneratedPasswordRequireDigitsKey: (true),
			
			// 字符必须包含符号
			AppExtensionGeneratedPasswordRequireSymbolsKey: (true),
			
			// Here are all the symbols available in the the 1Password Password Generator:
			// !@#$%^&*()_-+=|[]{}'\";.,>?/~`
			// The string for AppExtensionGeneratedPasswordForbiddenCharactersKey should contain the symbols and characters that you wish 1Password to exclude from the generated password.
			//生成密码的时候排除的字符
			AppExtensionGeneratedPasswordForbiddenCharactersKey: "!@#$%/0lIO"
		]
		
		//调用1Password,打开它的页面输入密码修改信息OnePasswordExtension.sharedExtension().changePasswordForLoginForURLString("https://www.acme.com", loginDetails: newLoginDetails, passwordGenerationOptions: passwordGenerationOptions, forViewController: self, sender: sender) { (loginDictionary, error) -> Void in
			if loginDictionary == nil {
				if error!.code != Int(AppExtensionErrorCodeCancelledByUser) {
					print("Error invoking 1Password App Extension for find login: \(error)")
				}
				return
			}
			
			//把用户的修改返还给界面
			self.oldPasswordTextField.text = loginDictionary?[AppExtensionOldPasswordKey] as? String
			self.freshPasswordTextField.text = loginDictionary?[AppExtensionPasswordKey] as? String
			self.confirmPasswordTextField.text = loginDictionary?[AppExtensionPasswordKey] as? String
		}
	```
	效果如图:
	
	![](/img/2015/11/30/6.PNG)
	![](/img/2015/11/30/7.PNG)
	