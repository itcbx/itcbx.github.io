---
title:  iOS逆向学习
author: itcbx
tags:
    - iOS
---

以往在iOS wx中阅读公众号文章时，当有信息过来的时候，如果想及时查阅并回复的话，就需要把整个文章界面关掉，并返回到最近聊天列表中，等回复完毕后再进入公众号中相应的文章进行阅读，非常的不方便。在最近几版的wx中，添加了一个`在聊天中置顶`的功能，可以将一个网页置顶在最近聊天列表中，以方便随时继续阅读。而在这个功能出现之前，已有插件实现了这个功能，甚至比目前wx自带的这个聊天置顶功能更加方便，具体可以看这个作者写的这篇文章[《我是如何利用 Xcode 调试开发微信消息预览插件的》](https://mp.weixin.qq.com/s?__biz=MjM5NTIyNTUyMQ==&mid=456678545&idx=1&sn=70f5784541fc15d5af252a15a400900d)。其实现方法，简单来说，就是把包括当前正在阅读的网页的所有`viewControllers`保存起来，快速返回到聊天页面，等需要的时候再把保存的`viewControllers`覆盖掉现有的聊天页面就行，当然，实际操作中还要考虑到许多情况。

由于插件所实现的`快速返回`方法，不只可以保存当前阅读的网页，可以保存包括公众号文章列表的所有页面，还是比内置的`在聊天中置顶`功能方便的，然而在最近的一次wx更新之后，该`快速返回`方法失效了，具体表现为返回到网页页面后，页面内容为空白。

网上的逆向教学，往往是直接给我们一个结果，告诉我们要去`hook`哪个类的哪个方法，然而逆向最关键的，恰恰是如何找到具体要修改哪个方法，这个却鲜有人写出来。今天写这篇文章，以具体问题具体分析的方式，解决这个网页页面内容空白的bug，同时让大家了解一下如何去定位到关键的类和方法。

逆向前的工具准备：

* 一台macOS电脑
* 一台越狱过的iPhone手机
* class-dump，可以dump出binary的头文件，对辅助逆向有极大的帮助
* Hopper Disassembler v4， 在反编译Objective-C上不逊于IDA，价格又比较亲民
* cycript， 使用js的语法写Objective-C，可以修改运行时的程序，在一些简单的调试测试上非常方便
* Reveal，可以查看iOS app的层级结构，获取相应控件的内存地址

我假设各位已经会安装并使用以上工具，如果不清楚的可以自行google，或者可以看看狗神写的神书《iOS应用逆向工程》，本文的目的是教授如何去逆向app寻找关键的节点，基础的工具使用不在讨论之列。

`快速返回`功能失效，可以想到应该是wx在更新了`在聊天中置顶`后做了一些手脚，那么我们就先来分析一下，`在聊天中置顶`这个功能是怎么实现的。首先我们来看看在点击了`在聊天中置顶`后，调用了什么类什么方法。随便在wx中打开一篇文章，点击右上角的`···`弹出菜单，随后在Reveal里查看该页面，定位到`在聊天中置顶`这个按钮，在Reveal右侧的信息框中，可以看到该按钮是一个`UIButton`类，并给出了内存地址，记录下来，我这边是`0x13f148c10`，注意该值是不固定的。

![reveal](https://blog.itcbx.com/uploads/images/2017-09-03-reveal.png)

随后在电脑上ssh到手机，输入`cycript -p WeChat`启用cycript，输入以下语句：

```
cy# [#0x13f148c10 allTargets ]
[NSSet setWithArray:@[#"<MMScrollActionSheetIconView: 0x13f152f40; frame = (0 0; 60 98); layer = <CALayer: 0x13f15de50>>"]]]
cy# [#0x13f148c10 allControlEvents ]
64
cy# [#0x13f148c10 actionsForTarget: #0x13f152f40 forControlEvent: 64]
@["onTaped"]
```

注意`cy#`是命令提示符，其后面的语句才是我们输入的，没有`cy#`的行是cycript的反馈输出，从上面的语句可以看出，点击`在聊天中置顶`这个按钮后，执行的是-[MMScrollActionSheetIconView onTaped]这个方法，接下来就是要祭上Hopper神器查看下该方法的具体实现了，将wx的二进制包扔进Hopper中并选择分析其中的64位架构的程序，等分析结束后，定位到`-[MMScrollActionSheetIconView onTaped]`查看其反汇编代码。简单分析后，可知该方法的大概流程为：

```Objective-C
-(void)onTaped {
    if(self.delegate != nil && [self.delegate  respondsToSelector:@selector(onActionSheetIconView:didTapedWithItem:)]) {
        [self.delegate onActionSheetIconView:self didTapedWithItem:self.item];
    }
}
```

在cycript输入以下语句：

```
cy# [#0x13f152f40 delegate]
#"<MMScrollActionSheet: 0x13f28e0e0; frame = (0 0; 0 0); layer = <CALayer: 0x13f28e2a0>>"
cy# [#0x13f152f40 item]
#"<MMScrollActionSheetItem: 0x13f2787a0>"
```

可以看出`self.delegate`和`self.item`的类型分别为`MMScrollActionSheet`和`MMScrollActionSheetItem`，在Hopper中定位到`-[MMScrollActionSheet onActionSheetIconView:didTapedWithItem:]`，分析之后，大概流程为：

```Objective-C
- (void)onActionSheetIconView:(MMScrollActionSheetIconView *)arg1 didTapedWithItem:(MMScrollActionSheetItem *)arg2 {

    // ...
    // 省略一些关闭菜单栏的代码
    // ...

    if(self.delegate != nil && [self.delegate respondsToSelector:@selector(scrollActionSheet:didSelecteItem:)]) {
        [self.delegate scrollActionSheet:self didSelecteItem:arg2];
    }
}
```

继续在cycript输入以下语句查看`MMScrollActionSheet`中的delegate类型：

```
cy# [#0x13f28e0e0 delegate]
#"<MMWebViewController: 0x13e919200>"
```

可知`MMScrollActionSheet`中的`delegate`类型为`MMWebViewController`，在Hopper中定位到`-[MMWebViewController scrollActionSheet:didSelecteItem:]`，内容很简单，就是继续调用了`-[MMWebViewController didSelecteMenuItem:]`，传入的值为`-[MMWebViewController scrollActionSheet:didSelecteItem:]`方法第二个参数，类型为`MMScrollActionSheetItem`，继续在Hopper中跟踪`-[MMWebViewController didSelecteMenuItem:]`方法，这个方法内容就比较多了，没法一眼看出关键点出来。

为了避免看花眼，我们要用个其他技巧，该方法中多次出现了类似`[r0 getStringForCurLanguage:r2 defaultTo:@"Webview_Open_With_Safari"]`的语句，这个很明显是通过关键字获取相应语言环境下的对应句子，而`Webview_Open_With_Safari`疑似对应菜单中的`在Safari中打开`。解开wx的资源包，找到`zh_CN.lproj`文件夹，里面有个mm.strings，在文件中搜索`在聊天中置顶`，找到`"Webview_returnBackToSession"="在聊天中置顶"`这个语句，回到Hopper中，在`-[MMWebViewController didSelecteMenuItem:]`方法中搜索`Webview_returnBackToSession`，找到之后，在该语句的下方不远处出现一个名为`MMWebViewKeepHolderMgr`的类及其方法调用`-[MMWebViewKeepHolderMgr keepHoldWebViewVCForNewMainFrameBanner:]`看名字，八九不离十是我们要找的目标方法了。

尝试在cycript中调用该方法：

```
cy# mgr = choose(MMWebViewKeepHolderMgr)[0]
#"<MMWebViewKeepHolderMgr: 0x161755eb0>"
cy# [mgr keepHoldWebViewVCForNewMainFrameBanner: #0x13e919200]
```

返回到最近聊天列表，发现已在聊天中置顶了一个网页，然而点击进去后是空白页面，尝试使用`在Safari中打开`，可以正常显示网页，说明`-[MMWebViewKeepHolderMgr keepHoldWebViewVCForNewMainFrameBanner:]`这个方法确实是将网页在聊天中置顶的方法，但是wx做了手脚，缺少了一些关键步骤，导致页面内容被置为空，但网页的网址信息等仍然有保留着。从调用`-[MMWebViewKeepHolderMgr keepHoldWebViewVCForNewMainFrameBanner:]`这个方法的地方继续往下看，发现几个可疑的语句：


```
0000000100ca64d4         ldrsw      x8, [x8, #0x2e8]                            ; objc_ivar_offset_MMWebViewController_m_bShowOnNewMainFrameBanner
0000000100ca64d8         orr        w9, wzr, #0x1
0000000100ca64dc         strb       w9, [x21, x8]
```

可以看出，这里是把`MMWebViewController`的成员变量`m_bShowOnNewMainFrameBanner`的值置为1，通过class-dump导出wx的所有头文件，查看`MMWebViewController`类的结构，发现`m_bShowOnNewMainFrameBanner`是该类的私有成员变量，类型为`_Bool`，重新在wx里打开一个网页，并在cycript中输入如下语句（注意`#0x160d0a600`是新打开的网页对象`MMWebViewController`的内存地址，按照上面写的通过`Reveal`获取）：

```
cy# mgr = choose(MMWebViewKeepHolderMgr)[0]
#"<MMWebViewKeepHolderMgr: 0x161755eb0>"
cy# web = #0x160d0a600
#"<MMWebViewController: 0x160d0a600>"
cy# [mgr keepHoldWebViewVCForNewMainFrameBanner: web]
cy# [web setValue:[NSNumber numberWithBool:YES] forKey:"m_bShowOnNewMainFrameBanner"]
```

输完之后在wx里返回最近聊天列表，发现网页已置顶，且点击之后能正常显示。至此，我们已经找到了解决插件`快速返回`功能失效，出现网页空白的方法，就是保存页面之后，要将`m_bShowOnNewMainFrameBanner`的值置为`YES`。

我们的目的已经达到，接下来是拓展思考时间，为什么没有设置`m_bShowOnNewMainFrameBanner`的值为`YES`会导致空白网页的出现呢。这里就要涉及到一些基础的iOS正向开发的知识了，逆向要玩的好，必须要有扎实的基础功。回到刚才的话题，为什么没有设置`m_bShowOnNewMainFrameBanner`的值为`YES`会导致空白网页的出现。我们知道，当一个视图隐藏或者关闭的时候，会自动调用一系列的方法，这些方法一般以`viewWill`和`viewDid`做为前辍命名，查看`MMWebViewController`所实现的方法，发现与视图隐藏关闭相关的方法有这么几个`viewWillDisappear:`，`viewDidDisappear:`，`viewWillBePoped:`，`viewDidBePoped:`，`viewWillBeDismissed:`，`viewWillPop:`，在hopper里查看这些方法的实现，发现`-[MMWebViewController viewDidBePoped:]`的大概流程为：

```Objective-C
- (void)viewWillBePoped:(_Bool)arg1 {

    // ···
    // 省略一些无关代码
    // ···

    if(m_bShowOnNewMainFrameBanner) {

        // ···
        // 省略一些无关代码
        // ···

    } else {
        if([webView isKindOfClass:[WKWebView class]]) {
            [webView loadHTMLString:@"<html></html>" baseURL:[NSURL URLWithString:[self getCurrentUrl]]]
        }
    }
}
```

真相大白！如果m_bShowOnNewMainFrameBanner的值不为真，则网页在被隐藏或关闭的时候，内容将被置为`<html></html>`，导致页面显示为空白！
