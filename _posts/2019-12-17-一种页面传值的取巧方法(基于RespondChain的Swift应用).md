---
layout: post
title: 一种页面传值的取巧方法(基于RespondChain的Swift应用)
date:   2019-12-17 00:00:00 +0800
categories: iOS开发记录
tag: iOS, 传值技巧
comments: true
---

* content
{:toc}
### 一种页面传值的取巧方法(基于RespondChain的Swift应用)

![未命名store_1024pt](https://raw.githubusercontent.com/xuyunshi/EasyResponder/master/Example/EasyResponder/Images.xcassets/AppIcon.appiconset/%E6%9C%AA%E5%91%BD%E5%90%8Dstore_1024pt.png)

第一次见到这种传值方式是来自Casa这里.
[一种基于ResponderChain的对象交互方式](https://casatwy.com/responder_chain_communication.html)

现在假设有一个页面A, 他有一个子视图A1, A1里有一个子视图AA1.

就如同封面图上，三层的View嵌套。

我们需要获取这个AA1视图里的一个点击事件的回调，一般可以采取Block, Delegate的形式来处理，那么最后写出来的结果很有可能是这样的： AA1把值传给A1, A1再把值传给A, A暴露出一个可供外界获取的接口（delegate或者block)让外界去处理点击事件。

这中间会夹杂着许多很无聊的代码，逐层传值的方式让UI仔真的很痛苦。

后来有幸看到了上文Casa的那篇文章，发现界面传值居然还可以这么做。

关于OC的使用方式，Casa的文章里已经讲的比较明白了，方式也比较简单，就是给UIResponder增加一个分类, 然后沿着传递链把他的 



由于Swift是不支持给UIResponder加一个分类的，我在做Swift版本的传递链是用了另外一种方式。

首先是定义了一个Event事件

```swif
public protocol EasyEvent {

    var identifier: String { get }

    var sourceView: UIView { get }
    
    var info: [String: Any] { get }
    
    mutating func insertInfo(_ newInfo: [String: Any])
}
```

其中identifier用来标志事件，sourceView用来指代事件发生的初始点，info用来携带一些其他信息, insert方法允许在传递的过程中增加一些数据到这个点击事件中。

然后定义了一个EventHandler

```swift
public protocol EasyEventHandler: UIResponder {
    
    var stopTransferIfEventHandled: Bool { get }
    
    func handle(_ event: EasyEvent) -> Bool
}
```

参数stopTranferIfEventHandled是如果事件在当前Responder被处理之后，那么是否还要继续往上传。默认为否

方法handler(:event)是处理事件的方法，如果当前要处理，那么就返回true, 否则为false.

我们在事件需要被处理的地方（一般就是ViewController)，给他遵守EasyEventHandler就可以正确地处理事件了。

以上就事件模式和事件处理者的逻辑。接下来讲一讲如何发起我们的event事件呢？



首先给UIRespomder加一个extension

```swif
extension UIResponder {
    
    func transEasyEvent(_ event: EasyEvent) {
        
        if let handler = self as? EasyEventHandler {
            let processed = handler.handle(event)
            if processed {
                if handler.stopTransferIfEventHandled {
                    return
                }
            }
        }
        next?.transEasyEvent(event)
    }
}
```



额，细心的同学们肯定可以看到这个协议扩展被限制在模块内部了，也就是说不对外开放的。

为什么呢？ 因为这个EasyEvent对象还要让业务方去主动构造，我认为不太友好，所以用了另外一种方式去传递我们的Event事件。

请看如下拓展

```swift
extension UIControl {
    
    public func transferEasyEventWithIdentifier(_ identifier: String, for event: UIControl.Event) {
        
        es_identifier = identifier
        addTarget(self, action: #selector(es_callBack(_:)), for: event)
    }
    
    @objc func es_callBack(_ sender: UIControl) {
        
        guard let identifier = self.es_identifier else { return }
        let event = AnyEasyEvent(identifier: identifier, sourceView: self)
        transEasyEvent(event)
    }
}
```

上述方法通过给UIControl增加一个方法，使得UIControl可以在触发UIControl.Event时，直接在传递链上发起一个对应的事件。

所以业务方的调用就会变成，只需要提供一个identifier和对应的event事件，方便了许多。

所以只需要给需要的控件增加对应的扩展，就可以实现比较方便的传递事件调用了。

类似的，在EasyResponder还实现了UITextField和UITextView的便捷调用。不展开赘述了。

这套流程讲到这里也就差不多了。

讲一讲一些局限性吧

1. 必须要在UIResponder传递链上才可以响应事件， 如果是非响应链上的事件需要找到在链上的最近点去中转一下对应的事件。
2. 事件的标志被定义为字符串。目前对于此类的标识信息除了枚举就是字符串，在这个地方用枚举显然是非常不明智的，所以也是下策中的上策了。

我的Github地址在[这里](https://github.com/xuyunshi/EasyResponder)

[感谢Casa和他的小龙朋友](https://casatwy.com/responder_chain_communication.html)