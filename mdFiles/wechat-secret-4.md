---
title: （非越狱版）iOS微信聊天加密插件 （四）终结篇
date: 2016-09-06 18:16:32
categories:
tags:
---




前面我们已经基本确定了聊天界面`MultiSelectTableViewCell`中的`TextMessageNodeView`绑定的`model`就是`CMessageWrap`，`CMessageWrap`中的`GetDisplayContent`负责返回文本数据给界面显示。
	
我们在之前的Tweak项目写下如下代码，然后观察一下聊天界面的变化：

```
	
#pragma mark - CMessageWrap hook

%hook CMessageWrap

- (id)GetDisplayContent{
    
    /* 聊天信息 */
    NSString *messageText = %orig;
	
    NSMutableString *tempStr = [[NSMutableString alloc] init];
    for (int i = 0; i < messageText.length ;i++) {
        [tempStr appendString:@"呱"];
    }
    return tempStr;
}
	
%end

```

使用之前我介绍的脚本一键编译安装。点开聊天界面如下：

<img src="https://github.com/wmf00032/imagesRecource/blob/master/wechat4_images/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%87.png?raw=true" width = "320" height = "568" alt="图片名称" align=center />


看见了吧，一片蛙声，你知道我们在聊什么吗？
	
	
里面还有一些细节，比如长按聊天信息的拦截，这个我在第三篇文章的时候已经讲过了。还有就是在加密状态下录音消息不能播放，这个也非常简单，读者有兴趣可以自己去完善。聊天消息的`model`绑定cell上面的`VoiceMessageNodeView`，而所所有的`cell`事件都在这是上面。
	
这篇文章有点长，写不动了，我们来看一看完整视频效果吧：

无法播放请点击该地址：
http://www.le.com/ptv/vplay/26482914.html

<embed bgcolor = "0x000000" src='http://player.56.com/v_MTQxNzgyODk1.swf' type='application/x-shockwave-flash' width='560' height='315'></embed>

<iframe bgcolor = "0x000000" width="560" height="315" src="https://www.youtube.com/embed/1LpJ-zd5XLE" frameborder="0" allowfullscreen></iframe>

插件下载：https://github.com/wmf00032/Resource

**郑重声明：未经本人允许严禁任何个人或组织转载！**