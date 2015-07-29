title: <转>Swift局部SCOPE
date: 2015-07-29 10:06:40
tags:
- Swift
toc: false
comments: true
---

#局部SCOPE

C 系语言中在方法内部我们是可以任意添加成对的大括号 {} 来限定代码的作用范围的。这么做一般来说有两个好处，首先是超过作用域后里面的临时变量就将失效，这不仅可以使方法内的命名更加容易，也使得那些不被需要的引用的回收提前进行了，可以稍微提高一些代码的效率；另外，在合适的位置插入括号也利于方法的梳理，对于那些不太方便提取为一个单独方法，但是又应该和当前方法内的其他部分进行一些区分的代码，使用大括号可以将这样的结构进行一个相对自然的划分。

举一个不失一般性的例子，虽然我个人不太喜欢使用代码手写 UI，但钟情于这么做的人还是不在少数。如果我们要在 Objective-C 中用代码构建 UI 的话，我们一般会选择在 -loadView 里写一些类似这样的代码：

```objc
-(void)loadView {
    UIView *view = [[UIView alloc] initWithFrame:CGRectMake(0, 0, 320, 480)];

    UILabel *titleLabel = [[UILabel alloc] 
            initWithFrame:CGRectMake(150, 30, 20, 40)];
    titleLabel.textColor = [UIColor redColor];
    titleLabel.text = @"Title";
    [view addSubview:titleLabel];

    UILabel *textLabel = [[UILabel alloc] 
            initWithFrame:CGRectMake(150, 80, 20, 40)];
    textLabel.textColor = [UIColor redColor];
    textLabel.text = @"Text";
    [view addSubview:textLabel];

    self.view = view;
}
```
<!--more-->
在这里只添加了两个 view，就已经够让人心烦的了。真实的界面当然会比这个复杂很多，想想看如果有十来个 view 的话，这段代码会变成什么样子吧。我们需要考虑对各个子 view 的命名，以确保它们的意义明确。如果我们在上面的代码中把某个配置 textLabel 的代码写错成了 titleLabel 的话，编译器也不会给我们任何警告。这种 bug 是非常难以发现的，因此在类似这种一大堆代码但是又不太可能进行重用的时候，我更推荐使用局部 scope 将它们分隔开来。比如上面的代码建议加上括号重写为以下形式，这样至少编译器会提醒我们一些低级错误，我们也可能更专注于每个代码块：

```objc
-(void)loadView {
    UIView *view = [[UIView alloc] initWithFrame:CGRectMake(0, 0, 320, 480)];

    {
        UILabel *titleLabel = [[UILabel alloc] 
                initWithFrame:CGRectMake(150, 30, 20, 40)];
        titleLabel.textColor = [UIColor redColor];
        titleLabel.text = @"Title";
        [view addSubview:titleLabel];    
    }   

    {
        UILabel *textLabel = [[UILabel alloc] 
                initWithFrame:CGRectMake(150, 80, 20, 40)];
        textLabel.textColor = [UIColor redColor];
        textLabel.text = @"Text";
        [view addSubview:textLabel];
    }

    self.view = view;
}
```

在 Swift 中，直接使用大括号的写法是不支持的，因为这和闭包的定义产生了冲突。如果我们想类似地使用局部 scope 来分隔代码的话，一个不错的选择是定义一个接受 ()->() 作为函数的全局方法，然后执行它：

```swift
func local(closure: ()->()) {
    closure()
}
```

在使用时，可以利用尾随闭包的特性模拟局部 scope：

```swift
override func loadView() {
    let view = UIView(frame: CGRectMake(0, 0, 320, 480))

    local {
        let titleLabel = UILabel(frame: CGRectMake(150, 30, 20, 40))
        titleLabel.textColor = UIColor.redColor()
        titleLabel.text = "Title"
        view.addSubview(titleLabel)
    }

    local {
        let textLabel = UILabel(frame: CGRectMake(150, 80, 20, 40))
        textLabel.textColor = UIColor.redColor()
        textLabel.text = "Text"
        view.addSubview(textLabel)
    }

    self.view = view
}
```

在 Objective-C 中还有一个很棒的技巧是使用 GNU C 的[声明扩展](https://gcc.gnu.org/onlinedocs/gcc/Statement-Exprs.html#Statement-Exprs)来在限制局部作用域的时候同时进行赋值，运用得当的话，可以使代码更加紧凑和整洁。比如上面的 titleLabel 如果我们需要保留一个引用的话，在 Objective-C 中可以写为：

```objc
self.titleLabel = ({
    UILabel *label = [[UILabel alloc] 
            initWithFrame:CGRectMake(150, 30, 20, 40)];
    label.textColor = [UIColor redColor];
    label.text = @"Title";
    [view addSubview:label];
    label;
});
```

Swift 里当然没有 GNU C 的扩展，但是使用匿名的闭包的话，写出类似的代码也不是难事：

```swift
titleLabel = {
    let label = UILabel(frame: CGRectMake(150, 30, 20, 40))
    label.textColor = UIColor.redColor()
    label.text = "Title"
    self.view.addSubview(label)
    return label
}()
```

这也是一种隔离代码的很好的方式。

___
原文链接:[http://swifter.tips/local-scope/](http://swifter.tips/local-scope/)
