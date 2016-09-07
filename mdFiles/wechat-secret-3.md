---
title: （非越狱版）iOS微信聊天加密插件 （三）
toc: true
date: 2016-09-06 17:01:02
categories:
tags:
---

我们已经可以获取到联系人列表了，就可以以此建立一个加密名单，在加密名单里面的联系人聊天界面显示所有的聊天信息都是秘密，录音也不能播放，只用输入正确的界面密令之后，聊天界面才会显示明文，录音也可以正常的播放了。
	
# 基本思路
要实现聊天界面信息加密，我们现在来分析一下基本的思路。当用户在首页点击某个联系人进入聊天界面之后，应该检索改用户是否在“加密联系人”列表之中，如果在，文本信息需要显示密文，录音信息不能点击播放。
	
## 文本信息如何显示密文
### 方案1. 
首先未删除的聊天记录在本地应该有备份，我们是否可以在加载聊天记录的时候讲文本信息处理成密文，但是这样的话问题来了。新收到的消息和刚刚发送的信息理论上应该是先显示再保存到本地数据库，该如何处理这些消息尼？这样就变得非常复杂啦！这种方案不可行，需要处理很多种消息类型情况
	
### 方案2. 
最终的明文是需要显示在`cell` 控件上，每一个`cell` 肯定对应一个消息`messageModel`，不管什么样的消息类型，当需要显示一个cell的时候肯定需要到当前的`messageModel`中去获取数据，我们是否可以拦截这个过程。以就是说，对于一个文本消息，`cell`会从对应的`messageModel`中去取这段文本内容然后显示在界面。我们是否可以在`messageModel`给`cell`返回对应文本内容的时候将明文改成密文返回给`cell`。这样`cell`拿到的就是一个密文显示出来。当下一次需要解密的时候，只要将这个拦截过程取消，并命令`tableView` 重新`reloadData`就可以显示明文消息啦。
	
是的，就是方案2，如果我们能找到`messageModel`返回给`cell` 文本消息的方法，然后将其拦截判断如果该联系在“加密列表”就返回一段密文。这样就能一劳永逸，不管以什么形式出现的文本消息，都能将其加密后显示。
	
	
`Cycript`定位`cell`获取消息文本内容
通过数据线连接到手机

<img src="https://github.com/wmf00032/imagesRecource/blob/master/wechat3_images/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%87.png?raw=true" width = "320" height = "568" alt="图片名称" align=center />
	
`Cycript` 命令如下：

```
localhost:python-client Mango$ ssh root@localhost -p 2222
root@localhost's password: 
wutde-iPhone:~ root# cycript -p WeChat

```
然后进入到聊天界面，通过`UIApp.keyWindow.recursiveDescription().toString()`命令查看道歉界面元素结构

```

cy# UIApp.keyWindow.recursiveDescription().toString()
`<iConsoleWindow: 0x14d41990; baseClass = UIWindow; frame = (0 0; 320 568); autoresize = W+H; gestureRecognizers = <NSArray: 0x14d409e0>; layer = <UIWindowLayer: 0x14d41670>>
   | <UILayoutContainerView: 0x161aff80; frame = (0 0; 320 568); autoresize = W+H; layer = <CALayer: 0x163be1a0>>
   |    | <UITransitionView: 0x16768460; frame = (0 0; 320 568); clipsToBounds = YES; autoresize = W+H; layer = <CALayer: 0x163f69c0>>
   |    |    | <UIViewControllerWrapperView: 0x163c4780; frame = (0 0; 320 568); autoresize = W+H; layer = <CALayer: 0x16776550>>
   |    |    |    | <UILayoutContainerView: 0x1620c680; frame = (0 0; 320 568); autoresize = W+H; gestureRecognizers = <NSArray: 0x160c2fe0>; layer = <CALayer: 0x166664e0>>
   |    |    |    |    | <UINavigationTransitionView: 0x162950a0; frame = (0 0; 320 568); clipsToBounds = YES; autoresize = W+H; layer = <CALayer: 0x162c0c10>>
   |    |    |    |    |    | <UIViewControllerWrapperView: 0x1617a840; frame = (0 0; 320 568); layer = <CALayer: 0x16161a20>>
   |    |    |    |    |    |    | <UIView: 0x163edfa0; frame = (0 0; 320 568); autoresize = W+H; layer = <CALayer: 0x167ba0b0>>
   |    |    |    |    |    |    |    | <MMMultiSelectToolView: 0x16228660; frame = (0 568; 320 50); hidden = YES; layer = <CALayer: 0x162bb220>>
   |    |    |    |    |    |    |    |    | <UIImageView: 0x16690f20; frame = (0 0; 320 50); opaque = NO; layer = <CALayer: 0x162d6c20>>
   |    |    |    |    |    |    |    |    | <UIButton: 0x16646da0; frame = (24 7.5; 35 35); opaque = NO; tag = 10002; layer = <CALayer: 0x1627a500>>
   |    |    |    |    |    |    |    |    |    | <UIImageView: 0x160782c0; frame = (0 0; 35 35); clipsToBounds = YES; opaque = NO; userInteractionEnabled = NO; layer = <CALayer: 0x16078340>>
   |    |    |    |    |    |    |    |    | <UIButton: 0x166e2120; frame = (103 7.5; 35 35); opaque = NO; tag = 10001; layer = <CALayer: 0x166eec20>>
   |    |    |    |    |    |    |    |    |    | <UIImageView: 0x16891d00; frame = (0 0; 35 35); clipsToBounds = YES; opaque = NO; userInteractionEnabled = NO; layer = <CALayer: 0x1663af20>>
   |    |    |    |    |    |    |    |    | <UIButton: 0x16962c90; frame = (182 7.5; 35 35); opaque = NO; tag = 10003; layer = <CALayer: 0x16962d90>>
   |    |    |    |    |    |    |    |    |    | <UIImageView: 0x168921f0; frame = (0 0; 35 35); clipsToBounds = YES; opaque = NO; userInteractionEnabled = NO; layer = <CALayer: 0x16891e00>>
   |    |    |    |    |    |    |    |    | <UIButton: 0x1666a6b0; frame = (261 7.5; 35 35); opaque = NO; tag = 10004; layer = <CALayer: 0x166d9060>>
   |    |    |    |    |    |    |    |    |    | <UIImageView: 0x166e8490; frame = (0 0; 35 35); clipsToBounds = YES; opaque = NO; userInteractionEnabled = NO; layer = <CALayer: 0x166e2f40>>
   |    |    |    |    |    |    |    | <MMTableView: 0x15bbb400; baseClass = UITableView; frame = (0 0; 320 568); clipsToBounds = YES; gestureRecognizers = <NSArray: 0x16610770>; layer = <CALayer: 0x16832160>; contentOffset: {0, 178}; contentSize: {320, 696}>
   |    |    |    |    |    |    |    |    | <UITableViewWrapperView: 0x168a4970; frame = (0 0; 320 568); gestureRecognizers = <NSArray: 0x1625c4d0>; layer = <CALayer: 0x162f7920>; contentOffset: {0, 0}; contentSize: {320, 568}>
   |    |    |    |    |    |    |    |    |    | <MultiSelectTableViewCell: 0x1678b220; baseClass = UITableViewCell; frame = (0 163; 320 41); autoresize = W; layer = <CALayer: 0x16348f10>>
   |    |    |    |    |    |    |    |    |    |    | <UITableViewCellContentView: 0x167cf6e0; frame = (0 0; 320 41); gestureRecognizers = <NSArray: 0x16131680>; layer = <CALayer: 0x167cf750>>
   |    |    |    |    |    |    |    |    |    |    |    | <UIView: 0x14d34a60; frame = (0 10; 320 28); layer = <CALayer: 0x14d2c090>>
   |    |    |    |    |    |    |    |    |    |    |    |    | <UIImageView: 0x168ceac0; frame = (139 0; 42 20); opaque = NO; userInteractionEnabled = NO; tag = 87460; layer = <CALayer: 0x16835130>>
   |    |    |    |    |    |    |    |    |    |    |    |    | <MMUILabel: 0x1664cab0; baseClass = UILabel; frame = (139 0; 42 20); text = '14:54'; userInteractionEnabled = NO; tag = 87459; layer = <_UILabelLayer: 0x16615720>>
   |    |    |    |    |    |    |    |    |    |    |    | <UIImageView: 0x161316c0; frame = (-27 0; 30 30); hidden = YES; opaque = NO; userInteractionEnabled = NO; layer = <CALayer: 0x16131740>>
   |    |    |    |    |    |    |    |    |    | <MultiSelectTableViewCell: 0x167ccaa0; baseClass = UITableViewCell; frame = (0 204; 320 62); autoresize = W; tag = 101; layer = <CALayer: 0x167f2ac0>>
   |    |    |    |    |    |    |    |    |    |    | <UITableViewCellContentView: 0x167f2b30; frame = (0 0; 320 62); gestureRecognizers = <NSArray: 0x167ccd90>; layer = <CALayer: 0x167ccc40>>
   |    |    |    |    |    |    |    |    |    |    |    | <VoiceMessageNodeView: 0x1621bf20; frame = (183 0; 128 59); layer = <CALayer: 0x166581b0>>
   |    |    |    |    |    |    |    |    |    |    |    |    | <UIView: 0x16228e10; frame = (0 0; 80 54); layer = <CALayer: 0x16228e80>>
   |    |    |    |    |    |    |    |    |    |    |    |    |    | <UIImageView: 0x169834d0; frame = (0 0; 80 54); opaque = NO; userInteractionEnabled = NO; layer = <CALayer: 0x1699e420>>
   |    |    |    |    |    |    |    |    |    |    |    |    |    | <UIImageView: 0x16728700; frame = (45.5 14; 12.5 17); opaque = NO; userInteractionEnabled = NO; layer = <CALayer: 0x16728780>>
   |    |    |    |    |    |    |    |    |    |    |    |    |    | <MMUILabel: 0x1699e680; baseClass = UILabel; frame = (-17 8; 17 39); text = '2'''; userInteractionEnabled = NO; layer = <_UILabelLayer: 0x1670d090>>
   |    |    |    |    |    |    |    |    |    |    |    |    | <UIView: 0x16640340; frame = (83 1; 47 47); gestureRecognizers = <NSArray: 0x1662b820>; layer = <CALayer: 0x166402d0>>
   |    |    |    |    |    |    |    |    |    |    |    |    |    | <UIButton: 0x16674b10; frame = (3 1; 40 40); opaque = NO; tag = 100001; layer = <CALayer: 0x166403b0>>
   |    |    |    |    |    |    |    |    |    |    |    |    |    |    | <UIImageView: 0x1689fe00; frame = (0 0; 40 40); clipsToBounds = YES; opaque = NO; userInteractionEnabled = NO; layer = <CALayer: 0x162acf00>>
   |    |    |    |    |    |    |    |    |    |    |    |    |    | <UIImageView: 0x1662b6c0; frame = (0 0; 46 46); opaque = NO; userInteractionEnabled = NO; layer = <CALayer: 0x1664fb40>>
   |    |    |    |    |    |    |    |    |    |    |    | <UIImageView: 0x167ccde0; frame = (-26.5 8; 30 30); opaque = NO; userInteractionEnabled = NO; layer = <CALayer: 0x167cce60>>
   |    |    |    |    |    |    |    |    |    | <MultiSelectTableViewCell: 0x167297a0; baseClass = UITableViewCell; frame = (0 594; 320 102); autoresize = W; tag = 101; layer = <CALayer: 0x16729770>>
   |    |    |    |    |    |    |    |    |    |    | <UITableViewCellContentView: 0x16729940; frame = (0 0; 320 102); gestureRecognizers = <NSArray: 0x16740830>; layer = <CALayer: 0x167299b0>>
   |    |    |    |    |    |    |    |    |    |    |    | <TextMessageNodeView: 0x162048a0; frame = (0 0; 325 99); layer = <CALayer: 0x1665b2a0>>
   |    |    |    |    |    |    |    |    |    |    |    |    | <UIView: 0x168a7150; frame = (55 0; 222 94); layer = <CALayer: 0x168a71c0>>
   |    |    |    |    |    |    |    |    |    |    |    |    |    | <UIImageView: 0x168aa650; frame = (0 0; 222 94); clipsToBounds = YES; opaque = NO; layer = <CALayer: 0x168a9850>>
   |    |    |    |    |    |    |    |    |    |    |    |    |    | <RichTextView: 0x168a9700; baseClass = UILabel; frame = (19 14; 187 60); opaque = NO; layer = <_UILabelLayer: 0x168a9690>>
   |    |    |    |    |    |    |    |    |    |    |    |    |    |    | <_UILabelContentLayer: 0x167d3190> (layer)
   |    |    |    |    |    |    |    |    |    |    |    |    | <UIView: 0x168a5e80; frame = (7 1; 47 47); gestureRecognizers = <NSArray: 0x168a67e0>; layer = <CALayer: 0x168a5e00>>
   |    |    |    |    |    |    |    |    |    |    |    |    |    | <UIButton: 0x168a5ef0; frame = (3 1; 40 40); opaque = NO; tag = 100001; layer = <CALayer: 0x168a5ff0>>
   |    |    |    |    |    |    |    |    |    |    |    |    |    |    | <UIImageView: 0x167f2e20; frame = (0 0; 40 40); clipsToBounds = YES; opaque = NO; userInteractionEnabled = NO; layer = <CALayer: 0x1670e720>>

```
	
从上面打印的信息来看，聊天界面中的`cell`就是`MultiSelectTableViewCell` ， 它里面包含着不同的自定义`View` ,有的是`TextMessageNodeView` 有的是`VoiceMessageNodeView` 。 从字面来看文本消息应该对应的是`TextMessageNodeView` ，音频消息应该对应的是`VoiceMessageNodeView` 。
	
下面验证我们的猜测
	
我们来看一下这条信息：

```
	<MultiSelectTableViewCell: 0x168a4970; baseClass = UITableViewCell; frame = (0 821; 320 102); autoresize = W; tag = 101; layer = <CALayer: 0x168a4b10>>
	   |    |    |    |    |    |    |    |    |    |    | <UITableViewCellContentView: 0x1689cab0; frame = (0 0; 320 102); gestureRecognizers = <NSArray: 0x168a7350>; layer = <CALayer: 0x1689cb20>>
	   |    |    |    |    |    |    |    |    |    |    |    | <TextMessageNodeView: 0x162695f0; frame = (0 0; 325 99); layer = <CALayer: 0x14d2c090>>
	   |    |    |    |    |    |    |    |    |    |    |    |    | <UIView: 0x161e7710; frame = (55 0; 222 94); layer = <CALayer: 0x16122d00>>
	   |    |    |    |    |    |    |    |    |    |    |    |    |    | <UIImageView: 0x166e6760; frame = (0 0; 222 94); clipsToBounds = YES; opaque = NO; layer = <CALayer: 0x16275840>>
	   |    |    |    |    |    |    |    |    |    |    |    |    |    | <RichTextView: 0x16798140; baseClass = UILabel; frame = (19 14; 187 60); opaque = NO; layer = <_UILabelLayer: 0x16725e20>>
	   |    |    |    |    |    |    |    |    |    |    |    |    |    |    | <_UILabelContentLayer: 0x163a63d0> (layer)
	   |    |    |    |    |    |    |    |    |    |    |    |    | <UIView: 0x161186b0; frame = (7 1; 47 47); gestureRecognizers = <NSArray: 0x163d5740>; layer = <CALayer: 0x169a3d50>>
	   |    |    |    |    |    |    |    |    |    |    |    |    |    | <UIButton: 0x1631ef80; frame = (3 1; 40 40); opaque = NO; tag = 100001; layer = <CALayer: 0x1679ecd0>>
	   |    |    |    |    |    |    |    |    |    |    |    |    |    |    | <UIImageView: 0x16345e10; frame = (0 0; 40 40); clipsToBounds = YES; opaque = NO; userInteractionEnabled = NO; layer = <CALayer: 0x1670d540>>
	   |    |    |    |    |    |    |    |    |    |    |    |    |    | <UIImageView: 0x161333e0; frame = (0 0; 46 46); opaque = NO; userInteractionEnabled = NO; layer = <CALayer: 0x16721ff0>>
	   |    |    |    |    |    |    |    |    |    |    |    | <UIImageView: 0x168a73a0; frame = (-26.5 8; 30 30); opaque = NO; userInteractionEnabled = NO; layer = <CALayer: 0x168a7420>>

其中`VoiceMessageNodeView` 的地址是 `0x1621bf20` 我们来改变它的背景颜色来看看界面会不会有变化

cy# #0x162695f0.backgroundColor = [UIColor redColor]

```


执行以上命令之后界面中的一个`cell`变成了红色

<img src="https://github.com/wmf00032/imagesRecource/blob/master/wechat3_images/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%872.png?raw=true" width = "320" height = "568" alt="图片名称" align=center />
	
	
是的，`TextMessageNodeView` 就是文本 文本聊天消息上面的自定义`view` , 其中它里面包含着这个子控件叫`RichTextView` ，如下：

```
<RichTextView: 0x16798140; baseClass = UILabel; frame = (19 14; 187 60); opaque = NO; layer = <_UILabelLayer: 0x16725e20>

```
它的命名和出现的位置实在是太可疑了，如果不出我们推断它应该就是现实 文本消息的 `label`，下面证明我们的推断。证明的方式就是通过`RichTextView` 里面的方法重新给它赋值，使其显示内容改变（方式很多，我想通过重新赋值的方式顺便看看它里面都有哪些方法）。  在之前我们到处我所有头文件中找到`RichTextView` 

![](https://github.com/wmf00032/imagesRecource/blob/master/wechat3_images/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%873.png?raw=true)
	
	
打开看一下这个头文件的内容：


```

@class NSMutableArray, NSString, UIColor, UIFont, UIImage;
	
@interface RichTextView : MMCPLabel <TextLayoutDelegate, WCForceTouchPreviewProtocol, WCForceTouchTriggerLongPressProtocol>
{
    NSMutableArray *_arrParserObjects;
    UIColor *_oNormalBackgroundColor;
    UIColor *_oHighlightedBGColor;
    BOOL _bWholeField;
    BOOL _bHightlighted;
    BOOL _bEnableBGColor;
    UIFont *_oFont;
    float _fWidth;
    UIColor *_oTextColor;
    UIColor *_oTextHLColor;
    NSMutableArray *_arrStyles;
    NSString *_nsContent;
    struct CGRect _touchedRect;
    BOOL _bSourceUrlForLP;
    unsigned int _parserType;
    id <RichTextLayoutDelegate> _layoutDelegate;
    id <ILinkEventExt> _linkDelegate;
    BOOL _bIsLongPressHandled;
    BOOL _bDismissHightLightOutside;
    BOOL _bHandleLongPress;
    BOOL _bHandleTextClick;
    BOOL _bTouchesPassOn;
    UIImage *_oImage;
    BOOL _bTouchBeginOnLink;
    UIColor *_oNormalBGColor;
}
	
+ (id)getStyleString:(id)arg1 font:(id)arg2 width:(float)arg3 parserType:(unsigned int)arg4;
+ (id)getStyleString:(id)arg1 parserType:(unsigned int)arg2;
+ (struct CGSize)sizeForPrefixContent:(id)arg1 TargetContent:(id)arg2 TargetParserString:(id)arg3 SuffixContent:(id)arg4 font:(id)arg5 width:(float)arg6 parserType:(unsigned int)arg7 delegate:(id)arg8 outArrStyles:(id *)arg9;
+ (struct CGSize)sizeForPrefixContent:(id)arg1 TargetContent:(id)arg2 TargetParserString:(id)arg3 SuffixContent:(id)arg4 font:(id)arg5 width:(float)arg6 parserType:(unsigned int)arg7 delegate:(id)arg8;

```
	
在里面找找有没有直接设置text的方法或者属性， 很遗憾没有找到，那就去它的父类`MMCPLabel` 里面去找，`MMCPLabel` 的定义如下：
	
```

@class NSString, UILongPressGestureRecognizer;
	
@interface MMCPLabel : MMUILabel
{
    NSString *m_cpKey;
    UILongPressGestureRecognizer *m_recognizer;
    BOOL _showRestoreMenuItem;
    id <MMCPLabelDelegate> _cpLabelDelegate;
}
	
@property(nonatomic) __weak id <MMCPLabelDelegate> cpLabelDelegate; // @synthesize cpLabelDelegate=_cpLabelDelegate;
@property(nonatomic) BOOL showRestoreMenuItem; // @synthesize showRestoreMenuItem=_showRestoreMenuItem;
@property(retain, nonatomic) NSString *cpKey; // @synthesize cpKey=m_cpKey;
- (void).cxx_destruct;
- (struct CGSize)sizeThatFits:(struct CGSize)arg1;
- (void)drawLayer:(id)arg1 inContext:(struct CGContext *)arg2;
- (void)layoutSublayersOfLayer:(id)arg1;
- (void)handleLongPressGestureRecognizer:(id)arg1;
- (void)onRestore:(id)arg1;
- (BOOL)canPerformAction:(SEL)arg1 withSender:(id)arg2;
- (BOOL)canBecomeFirstResponder;
- (void)setText:(id)arg1 highlightKeyWord:(id)arg2 range:(struct _NSRange)arg3;
- (void)setText:(id)arg1 highlightKeyWord:(id)arg2 startIndex:(unsigned int)arg3;
- (void)setText:(id)arg1 highlightKeyWord:(id)arg2;
	
@end

```
	
其中这几个方法有点可疑：

```
- (void)setText:(id)arg1 highlightKeyWord:(id)arg2 range:(struct _NSRange)arg3;
- (void)setText:(id)arg1 highlightKeyWord:(id)arg2 startIndex:(unsigned int)arg3;
- (void)setText:(id)arg1 highlightKeyWord:(id)arg2;
```

好吧，试一下看能不能改变它显示的值。

```
cy# [#0x16798140 setText:@"hello haha" highlightKeyWord:nil]

```
执行上面命令之后发现界面没有任何的改变。好吧，不是的！！再看看`MMCPLabel` 的父类 `MMUILabel`。定义如下：

```
	
@interface MMUILabel : UILabel
{
}
	
- (void)setFrame:(struct CGRect)arg1;
- (id)initWithFrame:(struct CGRect)arg1;
- (id)init;
	
@end

```
	
好吧，已经追踪到头了，`MMUILabel` 没有设置的方法，但是它是继承 `UILabel`的，我们试试UILabel中的text属性看能不能改变其值。

```

cy# #0x16798140.text = @"hello haha"
@"hello haha"

```
	
执行完后界面依然没有任何改变，是不是有点绝望了！ 我把这个掉进坑里的过程写下来，或许大家能够感受到一点点挫折！哈哈，不要灰心，如果没有困难又怎么能体会到成功后的喜悦尼？然我们重新整理思路，再次迎接挑战！
	
我们再次回到`RichTextView` 类，然后顺着继承的方向找下去， 认真观察里面的方法， 一定能找到蛛丝马迹让真相大白于天下的。
我们把 `RichTextView` 所有代码（以及100多行）粘出来，谢谢观察一番：

```
	
@class NSMutableArray, NSString, UIColor, UIFont, UIImage;
	
@interface RichTextView : MMCPLabel <TextLayoutDelegate, WCForceTouchPreviewProtocol, WCForceTouchTriggerLongPressProtocol>
{
    NSMutableArray *_arrParserObjects;
    UIColor *_oNormalBackgroundColor;
    UIColor *_oHighlightedBGColor;
    BOOL _bWholeField;
    BOOL _bHightlighted;
    BOOL _bEnableBGColor;
    UIFont *_oFont;
    float _fWidth;
    UIColor *_oTextColor;
    UIColor *_oTextHLColor;
    NSMutableArray *_arrStyles;
    NSString *_nsContent;
    struct CGRect _touchedRect;
    BOOL _bSourceUrlForLP;
    unsigned int _parserType;
    id <RichTextLayoutDelegate> _layoutDelegate;
    id <ILinkEventExt> _linkDelegate;
    BOOL _bIsLongPressHandled;
    BOOL _bDismissHightLightOutside;
    BOOL _bHandleLongPress;
    BOOL _bHandleTextClick;
    BOOL _bTouchesPassOn;
    UIImage *_oImage;
    BOOL _bTouchBeginOnLink;
    UIColor *_oNormalBGColor;
}
	
+ (id)getStyleString:(id)arg1 font:(id)arg2 width:(float)arg3 parserType:(unsigned int)arg4;
+ (id)getStyleString:(id)arg1 parserType:(unsigned int)arg2;
+ (struct CGSize)sizeForPrefixContent:(id)arg1 TargetContent:(id)arg2 TargetParserString:(id)arg3 SuffixContent:(id)arg4 font:(id)arg5 width:(float)arg6 parserType:(unsigned int)arg7 delegate:(id)arg8 outArrStyles:(id *)arg9;
+ (struct CGSize)sizeForPrefixContent:(id)arg1 TargetContent:(id)arg2 TargetParserString:(id)arg3 SuffixContent:(id)arg4 font:(id)arg5 width:(float)arg6 parserType:(unsigned int)arg7 delegate:(id)arg8;
+ (float)heightForPrefixContent:(id)arg1 TargetContent:(id)arg2 TargetParserString:(id)arg3 SuffixContent:(id)arg4 font:(id)arg5 width:(float)arg6 parserType:(unsigned int)arg7 delegate:(id)arg8 outArrStyles:(id *)arg9;
+ (float)heightForPrefixContent:(id)arg1 TargetContent:(id)arg2 TargetParserString:(id)arg3 SuffixContent:(id)arg4 font:(id)arg5 width:(float)arg6 parserType:(unsigned int)arg7 delegate:(id)arg8;
+ (id)getParserString:(id)arg1;
+ (id)getParserString:(id)arg1 parserType:(unsigned int)arg2;
+ (id)getStyleString:(id)arg1 font:(id)arg2 width:(float)arg3 parserType:(unsigned int)arg4 delegate:(id)arg5;
+ (float)getHeightForContent:(id)arg1 font:(id)arg2 width:(float)arg3 parserType:(unsigned int)arg4 delegate:(id)arg5 outArrStyles:(id *)arg6;
+ (float)getHeightForContent:(id)arg1 font:(id)arg2 width:(float)arg3 parserType:(unsigned int)arg4 delegate:(id)arg5;
+ (float)getHeightForContent:(id)arg1 font:(id)arg2 width:(float)arg3 parserType:(unsigned int)arg4;
+ (void)initialize;
+ (id)pureStringForContent:(id)arg1;
@property(nonatomic) BOOL bTouchBeginOnLink; // @synthesize bTouchBeginOnLink=_bTouchBeginOnLink;
@property(readonly, nonatomic) NSMutableArray *arrStyles; // @synthesize arrStyles=_arrStyles;
@property(nonatomic) __weak id <ILinkEventExt> linkDelegate; // @synthesize linkDelegate=_linkDelegate;
@property(nonatomic) BOOL bTouchesPassOn; // @synthesize bTouchesPassOn=_bTouchesPassOn;
@property(nonatomic) BOOL bDismissHightLightOutside; // @synthesize bDismissHightLightOutside=_bDismissHightLightOutside;
@property(nonatomic) BOOL bEnableBGColor; // @synthesize bEnableBGColor=_bEnableBGColor;
@property(retain, nonatomic) UIColor *oHighlightedBGColor; // @synthesize oHighlightedBGColor=_oHighlightedBGColor;
@property(retain, nonatomic) UIColor *oNormalBGColor; // @synthesize oNormalBGColor=_oNormalBGColor;
@property(nonatomic) BOOL bSourceUrlForLP; // @synthesize bSourceUrlForLP=_bSourceUrlForLP;
@property(nonatomic) BOOL bHandleTextClick; // @synthesize bHandleTextClick=_bHandleTextClick;
@property(nonatomic) BOOL bHandleLongPress; // @synthesize bHandleLongPress=_bHandleLongPress;
@property(nonatomic) __weak id <RichTextLayoutDelegate> layoutDelegate; // @synthesize layoutDelegate=_layoutDelegate;
@property(nonatomic) unsigned int parserType; // @synthesize parserType=_parserType;
@property(nonatomic) float fWidth; // @synthesize fWidth=_fWidth;
@property(retain, nonatomic) UIFont *oFont; // @synthesize oFont=_oFont;
@property(retain, nonatomic) UIColor *oTextHLColor; // @synthesize oTextHLColor=_oTextHLColor;
@property(retain, nonatomic) UIColor *oTextColor; // @synthesize oTextColor=_oTextColor;
- (void).cxx_destruct;
- (void)addStylesParserByPatternString:(id)arg1;
- (struct CGRect)getPreviewLinkFrameForLocation:(struct CGPoint)arg1 inView:(id)arg2;
- (id)getPreviewLinkForLocation:(struct CGPoint)arg1 inView:(id)arg2;
- (struct CGRect)previewingSourceRectForLocation:(struct CGPoint)arg1 inCoordinateView:(id)arg2;
- (id)viewControllerToPreviewWithFromController:(id)arg1 forLocation:(struct CGPoint)arg2 inCoordinateView:(id)arg3;
- (BOOL)canPeek;
- (void)updateAccessibilityLabel;
- (void)touchesCancelled:(id)arg1 withEvent:(id)arg2;
- (void)touchesEnded:(id)arg1 withEvent:(id)arg2;
- (void)delayedTouchesEnded:(id)arg1;
- (void)triggerLongPressFor3DTouchAtLocation:(struct CGPoint)arg1 inCoordinateView:(id)arg2;
- (void)touchesBegan:(id)arg1 withEvent:(id)arg2;
- (BOOL)pointInside:(struct CGPoint)arg1 withEvent:(id)arg2;
- (void)longPressOnTextEvent:(id)arg1;
- (void)clickOnTextEvent:(id)arg1;
- (void)longPressOnPhoneEvent:(id)arg1;
- (void)longPressOnLinkEvent:(id)arg1;
- (void)clickOnPhoneEvent:(id)arg1;
- (void)clickOnLinkEvent:(id)arg1;
- (void)drawRect:(struct CGRect)arg1;
- (void)dismissHighLight;
- (BOOL)setContent:(id)arg1 StyleString:(id)arg2;
- (BOOL)setPrefixContent:(id)arg1 TargetContent:(id)arg2 TargetParserString:(id)arg3 SuffixContent:(id)arg4;
- (BOOL)getStylesForContent:(id)arg1 parserString:(id)arg2 parserPosition:(struct _NSParserPosition)arg3;
- (id)getParserString:(id)arg1;
- (id)getStyleString:(id)arg1;
- (id)getPatternStringFromContent:(id)arg1 patternGenerator:(id)arg2;
- (id)getParserByPaserType:(unsigned int)arg1;
- (BOOL)shouldOpenNewLineAtY:(float)arg1 withLineHeight:(float)arg2;
- (float)originXForLineAtHeight:(float)arg1;
- (void)setContent:(id)arg1;
- (id)createParser:(unsigned int)arg1;
- (void)updateFrame:(float)arg1;
- (void)resetFrameForMinHeight:(float)arg1;
- (void)setArrStyles:(id)arg1 withContent:(id)arg2;
- (id)initWithCoder:(id)arg1;
- (id)initWithFrame:(struct CGRect)arg1;
- (id)init;
- (void)baseInit;
	
// Remaining properties
@property(readonly, copy) NSString *debugDescription;
@property(readonly, copy) NSString *description;
@property(readonly) unsigned int hash;
@property(readonly) Class superclass;
	
@end

```
	
其中有个方法 ` - (void)setContent:(id)arg1  `有没有感觉很可疑的样子。“设置内容”这名字跟隔壁老王太像了。推断其参数就是NSString.下面验证我们的推断：

```

cy# [#0x1665cc50 setContent:@"i found you"]

```
	
界面变成了

<img src="https://github.com/wmf00032/imagesRecource/blob/master/wechat3_images/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%874.png?raw=true" width = "320" height = "568" alt="图片名称" align=center />
	
好了，终于成功了！有朋友可能开始想了，能不能通过`hook`  `RichTextView` 的`setContent`对消息进行加密后重新赋值？理论上是可以的，但是消息的内容不应该在view中修改，view只管显示，而且这里的`view`是`cell`的子类，还要考虑`cell`复用的问题！ 所以我还是打算从model的角度去加密消息，因为每一个序号下的cell对应一个唯一的`Model`.
	
	
聊天界面控制器`BaseMsgContentViewController`
	
寻找当前控制器的方式前面已经讲过很多，当前的tablevView信息如下：

```

<MMTableView: 0x15341e00; baseClass = UITableView; frame = (0 0; 320 568); clipsToBounds = YES; gestureRecognizers = <NSArray: 0x16105240>; layer = <CALayer: 0x1611d6f0>; contentOffset: {0, 774}; contentSize: {320, 1292}>

	
cy# [#0x15341e00 nextResponder]
#"<UIView: 0x16642230; frame = (0 0; 320 568); autoresize = W+H; layer = <CALayer: 0x16098ff0>>"
cy# [#0x16642230 nextResponder]
#"<BaseMsgContentViewController: 0x15c39200>"

```

`BaseMsgContentViewController`就是当前界面的控制器
	
`TextMessageNodeView`深究
好了，现在我们来深究一下`TextMessageNodeView` 。我们知道聊天界面的`cell`就是`MultiSelectTableViewCell` 根据消息类型的不同其子控件有 `TextMessageNodeView` 和`VoiceMessageNodeView` 等。因为每一个`model`都对应一个`cell`,所以我可以推测出 `TextMessageNodeView` 应该绑定着文本消息model 而`VoiceMessageNodeView` 应该绑定着录音消息`model` 。下面我们先从`TextMessageNodeView` 进行切入。
	
我们打开`TextMessageNodeView` 头文件，里面的代码像这样的：

```
	
@class InteractionLabel, MMTipsViewController, MMUILabel, NSString, RichTextView, TextFloatPreview, UIButton, UIImageView, UIView, WCUIActionSheet;
	
@interface TextMessageNodeView : BaseMessageNodeView <WCActionSheetDelegate, RichTextLayoutDelegate, ILinkEventExt, IMsgRevokeExt, TextFloatPreviewDelegate, UIAlertViewDelegate, ITranslateMsgMgrExt, MMTipsViewControllerDelegate>
{
    RichTextView *m_oRichTextView;
    TextFloatPreview *m_floatPreview;
    UIView *m_oContainerTextView;
    UIImageView *m_oTranslateLineView;
    UIButton *m_oTranslateStatusButton;
    BOOL m_bTranslateAnimation;
    MMTipsViewController *m_oTranslateIntroView;
    WCUIActionSheet *_uiActionSheet;
    MMUILabel *_labelTitle;
    UIView *_labelTitleLine;
    BOOL _bShowFullText;
    float _fulltextHeight;
    float _limitHeight;
    InteractionLabel *_fullTextLabel;
}
	
+ (BOOL)canCreateMessageNodeViewInstanceWithMessageWrap:(id)arg1;
- (void).cxx_destruct;
- (struct CGRect)previewingSourceRectForLocation:(struct CGPoint)arg1 inCoordinateView:(id)arg2;
- (id)viewControllerToPreviewWithFromController:(id)arg1 forLocation:(struct CGPoint)arg2 inCoordinateView:(id)arg3;
- (BOOL)canPeek;
- (void)onPhoneClicked:(id)arg1 withRect:(struct CGRect)arg2;
- (id)getMoreMainInfomationAccessibilityDescription;
- (void)triggerLongPressFor3DTouchAtLocation:(struct CGPoint)arg1 inCoordinateView:(id)arg2;
- (BOOL)shouldOpenNewLineAtY:(float)arg1 withLineHeight:(float)arg2 richTextView:(id)arg3;
- (void)onLinkClicked:(id)arg1 withRect:(struct CGRect)arg2;
- (void)onTranslateMessageFailed:(id)arg1 errTip:(id)arg2;
- (void)handleChangeForTranslateMsg;
- (void)resizeFrameForTranslate;
- (void)onTranslateMessageChanged:(id)arg1;
- (void)onClickTipsBtn:(unsigned int)arg1;
- (id)patternText;
- (id)titleText;
- (id)bigTitleText;
- (id)getContactDisplayName:(id)arg1;
- (void)dealloc;
- (void)onMenuItemWillHide;
- (void)onTouchCancel;
- (void)onLongTouch;
- (void)onTouchUpInside;
- (void)onHide;
- (void)onWindowHide;
- (void)onTouchDownRepeat;
- (void)onTouchDown;
- (void)alertView:(id)arg1 clickedButtonAtIndex:(int)arg2;
- (void)OnMsgRevoked:(id)arg1 n64MsgId:(long long)arg2 SysMsg:(id)arg3;
- (id)accessibilityLabel;
- (BOOL)isAccessibilityElement;
- (void)onMoreOperate:(id)arg1;
- (void)addFavorite;
- (void)onFavoriteAdd:(id)arg1;
- (void)onCopy:(id)arg1;
- (void)onForward:(id)arg1;
- (void)onTranslateMsg:(id)arg1;
- (void)translateMsg;
- (BOOL)isSupportingTranslation;
- (void)updateSubviewsForTranslate;
- (void)onStopTranslateAnimation;
- (void)initTranslateStatusButton;
- (void)reLayoutSubviews;
- (BOOL)canShowTranslateBottomView;
- (void)showOpearation;
- (void)onOrientationChanged;
- (void)actionSheet:(id)arg1 didDismissWithButtonIndex:(int)arg2;
- (void)actionSheet:(id)arg1 clickedButtonAtIndex:(int)arg2;
- (void)onLinkLongPressed:(id)arg1 withRect:(struct CGRect)arg2;
- (void)onCopyLinkText:(id)arg1;
- (BOOL)onlyContainsLink;
- (struct CGRect)getPreviewLinkFrameForLocation:(struct CGPoint)arg1 inView:(id)arg2;
- (id)getPreviewLinkForLocation:(struct CGPoint)arg1 inView:(id)arg2;
- (void)layoutSubviewsInternal;
- (float)calculateTranslatedRichTextWidth;
- (float)calculateOriginRichTextHeight;
- (BOOL)shouldShowTranslatedText;
- (void)setRichtextViewContent;
- (void)updateSubviews;
- (void)updateBkgImage:(BOOL)arg1;
- (void)updateFrameForShowFullText;
- (void)showFullText;
- (struct CGSize)sizeForFullTextLabel;
- (struct CGSize)sizeForFrame:(struct CGRect)arg1;
- (void)initLabelTitle;
- (float)labelWidth;
- (void)updateStatus:(id)arg1;
- (id)getSystemFont;
- (id)initWithMessageWrap:(id)arg1 Contact:(id)arg2 ChatContact:(id)arg3;
	
// Remaining properties
@property(readonly, copy) NSString *debugDescription;
@property(readonly, copy) NSString *description;
@property(readonly) unsigned int hash;
@property(readonly) Class superclass;
	
@end

```

是不是已经发现让你兴奋的东西啦

```

- (void)onMenuItemWillHide;
- (void)onTouchCancel;
- (void)onLongTouch;
- (void)onTouchUpInside;
- (void)onHide;
- (void)onWindowHide;
- (void)onTouchDownRepeat;
- (void)onTouchDown;

```

这几个方法也太明显了，还是那句话，有屁股都能想到是干嘛的！但是我们还是要验证一下：

```
cy# [#0x167929a0 onLongTouch]

```
	
执行上面的命令后`cell`弹出了菜单

<img src="https://github.com/wmf00032/imagesRecource/blob/master/wechat3_images/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%875.png?raw=true" width = "320" height = "568" alt="图片名称" align=center />
	
	
我们再调用一下双击的方法试试：

```
cy# [#0x167929a0 onTouchDownRepeat]

```
	
界面变成了这样：

<img src="https://github.com/wmf00032/imagesRecource/blob/master/wechat3_images/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%876.png?raw=true" width = "320" height = "568" alt="图片名称" align=center />
	
	
因为之前我们仅仅是改变了 `cell` 上 `RichTextView` 的显示，而没有改变`model`中的值，所以这里又显示回了原来的值而不是`“I found you”`，这就是之前我们所说的为什么不通过hook `RichTextView` 的 `setContent`方法来加密的原因之一了。
	
我们在看看`TextMessageNodeView` 类，看看里面有没有绑定`messageModel` 。`TextMessageNodeView` 没有找到，我们就看看其父类`BaseMessageNodeView`，它头文件中的代码如下：

```

@interface BaseMessageNodeView : MMUIView <IHeadImageExt, IContactMgrExt, IQQContactMgrExt, IStrangerContactMgrExt, WCActionSheetDelegate, IAppDataExt, MMIconActionSheetDelegate, EmoticonDescMgrExt, IKFContactExt, IEnterpriseContactMgrExt, UIViewForceTouchShakeProtocol, WCForceTouchPreviewProtocol, WCForceTouchTriggerLongPressProtocol>
{
    CMessageWrap *m_oMessageWrap;
    CBaseContact *m_oContact;
    CBaseContact *m_oChatContact;
    id <messageNodeViewDelegate> m_delegate;
    unsigned long m_uiTouchBeginTime;
    BOOL m_bIsLongPressHandled;
    unsigned long m_eNodeType;
    UIView *m_oContentView;
    UIView *m_oHeadImageView;
    UIButton *m_oCommentButton;
    MMCPLabel *m_oChatRoomNameLabel;
    UIImageView *m_oBkgImageView;
    BOOL m_bHasLayout;
    float m_lastScreenWidth;
    float m_fContentViewLeftMargin;
    float m_fContentViewRightMargin;
    int m_orientation;
    NSArray *m_arrMenuItems;
    UIButton *m_oSendFailButton;
    UIImageView *m_oSendOKView;
    UIActivityIndicatorView *m_oActivityIndicator;
    UIButton *m_oAppBottomButton;
    AppMessageBlockButton *m_oAppMessageBlockButton;
    BOOL m_donorIconHidden;
    BOOL m_bSendOKShowAnimate;
    float m_fHeightInSizeForFrame;
    BOOL m_bMenuVC;
    MMIconActionSheet *m_iconActionSheet;
    NSString *m_cpKeyForChatRoomMessage;
    BOOL m_isChatRoomMessageUnsafe;
    BOOL m_touchEnded;
    CTRichTextView *m_crashWarningLabel;
    NSString *m_cpKeyForChatRoomDisplayName;
    BOOL m_isChatRoomDisplayNameUnsafe;
    BOOL m_isConverting3dTouchToLongPress;
}

```

好吧，第一个属性 `m_oMessageWrap`看上去很可疑，它或许就是我们一直要找的绑定在 `cell`上面的`model`，我们来看看它的头文件都有啥（复制的部分内容）：
	
```
	
@interface CMessageWrap : MMObject <IAppMsgPathMgr, ISysNewXmlMsgExtendOperation, IMsgExtendOperation, NSCopying>
{
    BOOL m_bIsSplit;
    BOOL m_bNew;
    unsigned long m_uiMesLocalID;
    long long m_n64MesSvrID;
    NSString *m_nsFromUsr;
    NSString *m_nsToUsr;
    unsigned long m_uiMessageType;
    NSString *m_nsContent;
    unsigned long m_uiStatus;
    unsigned long m_uiImgStatus;
    unsigned long m_uiMsgFlag;
    unsigned long m_uiCreateTime;
    NSString *m_nsPushContent;
    NSString *m_nsMsgSource;
    NSString *m_nsRealChatUsr;
    NSData *m_dtThumbnail;
    unsigned long m_uiSendTime;
    unsigned long m_uiEmojiStatFlag;
    NSString *m_nsPattern;
    BOOL m_bForward;
    BOOL m_bCdnForward;
    unsigned long m_uiPercent;
    unsigned long m_uiDownloadStatus;
    id <IMsgExtendOperation> m_extendInfoWithMsgTypeForBiz;
    id <IMsgExtendOperation> m_extendInfoWithMsgType;
    id <IMsgExtendOperation> m_extendInfoWithFromUsr;
    NSString *m_nsLastDisplayContent;
    BOOL m_isTempSessionMsg;
    BOOL m_isEnterpriseMsg;
    unsigned long m_sequenceId;
    NSString *m_nsKFWorkerOpenID;
    NSString *m_nsBizClientMsgID;
    NSString *m_nsBizChatId;
    unsigned long m_uiBizChatVer;
    NSString *m_nsAtUserList;
    unsigned int watchMsgSourceType;
    NSString *m_nsDisplayName;
}
	
+ (id)createMaskedThumbImageForMessageWrap:(id)arg1;
+ (id)GetCdnDownloadPathOfMsgThumb:(id)arg1;
+ (id)GetTempPathOfMesShortVideoWithMessageWrap:(id)arg1;
+ (id)GetPathOfMesVideoWithMessageWrap:(id)arg1;
+ (id)getMaskedMsgImgThumb:(id)arg1;
+ (id)getMsgImgThumb:(id)arg1;
+ (id)getPathOfVideoMsgImgThumb:(id)arg1;
+ (id)GetPathOfMaskedSquareMesImgThumbDir:(id)arg1;
+ (id)GetPathOfSquareMesImgThumb:(id)arg1;
+ (id)getPathOfMaskedMsgImgThumb:(id)arg1;
+ (id)getPathOfMessageImageCache;
+ (id)getOldPathOfMessageImageCache;
+ (id)getPathOfMsgImgThumb:(id)arg1;
+ (id)getMsgImgData:(id)arg1;
+ (id)getMsgImg:(id)arg1;
+ (id)getPathOfMsgImg:(id)arg1;
+ (id)getMsgHDImgData:(id)arg1;
+ (id)getMsgHDImg:(id)arg1;
+ (id)getPathOfMsgHDImg:(id)arg1;
+ (id)getUserNameFromMsgWrap:(id)arg1;
+ (BOOL)isSenderFromMsgWrap:(id)arg1;
+ (BOOL)IsRecordMsg:(id)arg1;
+ (BOOL)SaveMesImg:(id)arg1 MsgWrap:(id)arg2;
+ (BOOL)SaveMsgThumbWithMsgWrap:(id)arg1;
+ (void)clearLocalMaskedThumbImage:(id)arg1;
+ (void)clearLocalImage:(id)arg1;
+ (id)FormMessageWrapFromAddMsg:(id)arg1;
+ (id)FormMessageWrapFromBuffer:(id)arg1;
+ (void)initialize;
+ (void)GetPathOfAppRemindAttach:(id)arg1 retStrPath:(id *)arg2;
+ (void)GetPathOfAppThumb:(id)arg1 LocalID:(unsigned long)arg2 retStrPath:(id *)arg3;
+ (void)GetPathOfMaskedAppThumb:(id)arg1 LocalID:(unsigned long)arg2 retStrPath:(id *)arg3;
+ (void)GetPathOfAppDataTemp:(id)arg1 LocalID:(unsigned long)arg2 retStrPath:(id *)arg3;
+ (void)GetPathOfAppDataTemp:(id)arg1 LocalID:(unsigned long)arg2 AttachId:(id)arg3 retStrPath:(id *)arg4;
+ (void)GetPathOfAppDataByUserName:(id)arg1 andMessageWrap:(id)arg2 retStrPath:(id *)arg3;
+ (void)GetPathOfAppDataByUserName:(id)arg1 andMessageWrap:(id)arg2 andAttachId:(id)arg3 andAttachFileExt:(id)arg4 retStrPath:(id *)arg5;
+ (void)GetPathOfAppData:(id)arg1 LocalID:(unsigned long)arg2 FileExt:(id)arg3 retStrPath:(id *)arg4;
+ (void)GetPathOfAppImgCacheDir:(id)arg1 retStrPath:(id *)arg2;
+ (void)GetPathOfAppDir:(id)arg1 retStrPath:(id *)arg2;
+ (id)getMessageListStatusImage:(unsigned long)arg1;
@property(retain, nonatomic) NSString *m_nsDisplayName; // @synthesize m_nsDisplayName;
@property(nonatomic) unsigned long m_sequenceId; // @synthesize m_sequenceId;
@property(nonatomic) BOOL m_isEnterpriseMsg; // @synthesize m_isEnterpriseMsg;
@property(nonatomic) BOOL m_isTempSessionMsg; // @synthesize m_isTempSessionMsg;
@property(nonatomic) unsigned int watchMsgSourceType; // @synthesize watchMsgSourceType;
@property(retain, nonatomic) NSString *m_nsAtUserList; // @synthesize m_nsAtUserList;
@property(nonatomic) unsigned long m_uiBizChatVer; // @synthesize m_uiBizChatVer;
@property(retain, nonatomic) NSString *m_nsBizChatId; // @synthesize m_nsBizChatId;
@property(retain, nonatomic) NSString *m_nsBizClientMsgID; // @synthesize m_nsBizClientMsgID;
@property(retain, nonatomic) NSString *m_nsKFWorkerOpenID; // @synthesize m_nsKFWorkerOpenID;
@property(nonatomic) unsigned long m_uiDownloadStatus; // @synthesize m_uiDownloadStatus;
@property(nonatomic) unsigned long m_uiPercent; // @synthesize m_uiPercent;
@property(retain, nonatomic) NSString *m_nsPattern; // @synthesize m_nsPattern;
@property(nonatomic) unsigned long m_uiEmojiStatFlag; // @synthesize m_uiEmojiStatFlag;
@property(nonatomic) unsigned long m_uiSendTime; // @synthesize m_uiSendTime;
@property(nonatomic) BOOL m_bNew; // @synthesize m_bNew;
@property(nonatomic) BOOL m_bIsSplit; // @synthesize m_bIsSplit;
@property(retain, nonatomic) NSData *m_dtThumbnail; // @synthesize m_dtThumbnail;
@property(nonatomic) BOOL m_bCdnForward; // @synthesize m_bCdnForward;
@property(nonatomic) BOOL m_bForward; // @synthesize m_bForward;
@property(retain, nonatomic) NSString *m_nsRealChatUsr; // @synthesize m_nsRealChatUsr;
@property(retain, nonatomic) id <IMsgExtendOperation> m_extendInfoWithFromUsr; // @synthesize m_extendInfoWithFromUsr;
@property(retain, nonatomic) id <IMsgExtendOperation> m_extendInfoWithMsgType; // @synthesize m_extendInfoWithMsgType;
@property(retain, nonatomic) id <IMsgExtendOperation> m_extendInfoWithMsgTypeForBiz; // @synthesize m_extendInfoWithMsgTypeForBiz;
@property(retain, nonatomic) NSString *m_nsMsgSource; // @synthesize m_nsMsgSource;
@property(retain, nonatomic) NSString *m_nsPushContent; // @synthesize m_nsPushContent;
@property(nonatomic) unsigned long m_uiCreateTime; // @synthesize m_uiCreateTime;
@property(nonatomic) unsigned long m_uiMsgFlag; // @synthesize m_uiMsgFlag;
@property(nonatomic) unsigned long m_uiImgStatus; // @synthesize m_uiImgStatus;
@property(nonatomic) unsigned long m_uiStatus; // @synthesize m_uiStatus;
@property(retain, nonatomic) NSString *m_nsContent; // @synthesize m_nsContent;
@property(nonatomic) unsigned long m_uiMessageType; // @synthesize m_uiMessageType;
@property(retain, nonatomic) NSString *m_nsToUsr; // @synthesize m_nsToUsr;
@property(retain, nonatomic) NSString *m_nsFromUsr; // @synthesize m_nsFromUsr;
@property(nonatomic) long long m_n64MesSvrID; // @synthesize m_n64MesSvrID;
@property(nonatomic) unsigned long m_uiMesLocalID; // @synthesize m_uiMesLocalID;
- (void).cxx_destruct;
- (BOOL)isSentOK;
- (BOOL)IsRoomAnnouncement;
- (BOOL)IsAtMe;
- (BOOL)isShowAppMessageBlockButton;
- (BOOL)isShowAppBottomButton;
- (BOOL)isShowCommentButton;
- (id)keyDescription;
@property(readonly, copy) NSString *description;
- (BOOL)IsNeedChatExt;
- (BOOL)genImageFromMMAssetAndNotify:(id)arg1;
- (id)GetDisplayContent;
- (id)GetThumb;
- (id)GetImg;
- (id)GetMsgClientMsgID;
- (BOOL)IsSameMsgWithFullCheck:(id)arg1;
- (BOOL)IsSameMsg:(id)arg1;
- (BOOL)IsSendBySendMsg;
- (BOOL)IsAppMessage;
- (BOOL)IsShortMovieMsg;
- (BOOL)IsVideoMsg;
- (BOOL)IsImgMsg;
- (BOOL)IsTextMsg;
- (BOOL)IsChatRoomMessage;
- (BOOL)IsMassSendMessage;
- (BOOL)IsBottleMessage;
- (BOOL)IsQQMessage;
- (BOOL)IsSxMessage;
- (id)GetChatName;
- (void)AddTagToMsgSource:(id)arg1 value:(id)arg2;
- (void)UpdateMsgSource;
- (void)ChangeForDisplay;
- (void)ChangeForBackup;
- (void)fillMsgSourceFromContact:(id)arg1 isFromTempSession:(BOOL)arg2;
- (void)ChangeForMsgSource;
- (void)ChangeForChatRoom;
- (id)forwardingTargetForSelector:(SEL)arg1;
- (void)forwardInvocation:(id)arg1;
- (id)methodSignatureForSelector:(SEL)arg1;
- (id)copyWithZone:(struct _NSZone *)arg1;
- (id)initWithMsgType:(int)arg1 nsFromUsr:(id)arg2;
- (id)initWithMsgType:(int)arg1;
- (id)init;
- (id)wishingString;
- (BOOL)bIsAppUrlTypeWithCanvas;
- (id)nativeUrl;
- (int)yoType;
- (unsigned int)yoCount;
- (id)getNodeBtnList;
- (int)compareQQAscending:(id)arg1;
- (int)compareSXAscending:(id)arg1;
- (int)compareMessageAscending:(id)arg1;
- (int)compareMessageLocalIDDescending:(id)arg1;
- (BOOL)isAd;
- (id)GetCdnDownloadThumbPathOfVideo;
- (id)GetCdnDownloadPathOfVideo;

```
	
里面有几个方法看上去很可疑：

```

- (id)GetDisplayContent;
- (id)GetThumb;
- (id)GetImg;
- (id)GetMsgClientMsgID;
- (BOOL)IsSameMsgWithFullCheck:(id)arg1;
- (BOOL)IsSameMsg:(id)arg1;
- (BOOL)IsSendBySendMsg;
- (BOOL)IsAppMessage;
- (BOOL)IsShortMovieMsg;
- (BOOL)IsVideoMsg;
- (BOOL)IsImgMsg;
- (BOOL)IsTextMsg;
- (BOOL)IsChatRoomMessage;
- (BOOL)IsMassSendMessage;
- (BOOL)IsBottleMessage;
- (BOOL)IsQQMessage;
- (BOOL)IsSxMessage;
- (id)GetChatName;


```

是的，我都不想多说了，你应该可以猜到是什么意思。其中 `GetDisplayContent` 应该就是获取当前消息显示内容的方法
我们来验证一下：
	
为了不被其它的干扰，我重新打开了一个界面，里面只有一条聊天消息，如下：

<img src="https://github.com/wmf00032/imagesRecource/blob/master/wechat3_images/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%877.png?raw=true" width = "320" height = "568" alt="图片名称" align=center />

对应的`cell` 信息如下：

```

<MultiSelectTableViewCell: 0x16395fe0; baseClass = UITableViewCell; frame = (0 41; 320 62); autoresize = W; tag = 101; layer = <CALayer: 0x1632c680>>
   |    |    |    |    |    |    |    |    |    |    | <UITableViewCellContentView: 0x169804f0; frame = (0 0; 320 62); gestureRecognizers = <NSArray: 0x1678a410>; layer = <CALayer: 0x16326d50>>
   |    |    |    |    |    |    |    |    |    |    |    | <TextMessageNodeView: 0x1631f780; frame = (162 0; 149 59); layer = <CALayer: 0x16760930>>
   |    |    |    |    |    |    |    |    |    |    |    |    | <UIView: 0x167709e0; frame = (0 0; 101 54); layer = <CALayer: 0x16167ca0>>
   |    |    |    |    |    |    |    |    |    |    |    |    |    | <UIImageView: 0x16737390; frame = (0 0; 101 54); clipsToBounds = YES; opaque = NO; layer = <CALayer: 0x16713760>>
   |    |    |    |    |    |    |    |    |    |    |    |    |    | <RichTextView: 0x167d9840; baseClass = UILabel; frame = (18 14; 64 20); opaque = NO; layer = <_UILabelLayer: 0x14fa2540>>
   |    |    |    |    |    |    |    |    |    |    |    |    |    |    | <_UILabelContentLayer: 0x14f58480> (layer)
   |    |    |    |    |    |    |    |    |    |    |    |    | <UIView: 0x16783120; frame = (104 1; 47 47); gestureRecognizers = <NSArray: 0x16671f40>; layer = <CALayer: 0x163184c0>>
   |    |    |    |    |    |    |    |    |    |    |    |    |    | <UIButton: 0x14f95670; frame = (3 1; 40 40); opaque = NO; tag = 100001; layer = <CALayer: 0x16345890>>
   |    |    |    |    |    |    |    |    |    |    |    |    |    |    | <UIImageView: 0x16361240; frame = (0 0; 40 40); clipsToBounds = YES; opaque = NO; userInteractionEnabled = NO; layer = <CALayer: 0x1674c0c0>>
   
```
	
我们来尝试着获取`TextMessageNodeView`中的`m_oMessageWrap`的值：

```

cy# var myMsg = [#0x1631f780 valueForKey:@"m_oMessageWrap"]
#"{m_uiMesLocalID=1, m_ui64MesSvrID=4285525108122338937, m_nsFromUsr=wxi*j22~19, m_nsToUsr=wxi*w21~19, m_uiStatus=2, type=1, msgSource=\"(null)\"} "
cy# [myMsg GetDisplayContent]
@"i like you"

```
	
是的，就是它了，`cell` 绑定的`model`就是`CMessageWrap` , `CMessageWrap`中的`GetDisplayContent`返回一个值给`cell` 的 `TextMessageNodeView`去显示，以就是说，我们只需要hook `CMessageWrap`的`GetDisplayContent`方法，让其对指定的联系人返回一段密文就可以啦


	
**下一篇：（非越狱版）iOS微信聊天加密插件 （四）终结篇**

**郑重声明：未经本人允许严禁任何个人或组织转载！**
