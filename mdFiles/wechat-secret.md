---
title: （非越狱版）iOS微信聊天加密插件 （一）
toc: true
date: 2016-09-06 11:29:46
categories:
tags:
---

# 简介
之前见过别人做过微信自动抢红包插件，感觉挺酷，自己也看了一堆关于iOS应用逆向和安全的书，打算自己也做个小玩意出来，希望以此激发自己不断探索和学习的激情。

做个什么好尼？别人已经做过的 再去做就没有意思啦，打算做一个别人都没有尝试过的玩意。那就做个微信聊天加密插件吧！产生这样的需求的原因： 每次见到女朋友第一时间就是交上手机，然后微信、QQ聊天记录让她一个一个的查看，真的好烦，不给手机还不行，不给手机她就给你哭爹喊娘要死要活，说你做贼心虚。好吧！有的时候有些聊天信息来不及删除，万一被看见那就大祸临头了。如果能过对指定的聊天人信息进行加密，而又不引起她（他）的察觉该多好！

哈哈，宅男、浪女们是不是有点小激动。先给大家视频演示一下效果**（Youtube视频，观看需翻墙）**，然后给大家慢慢解释实现原理：

无法播放请点击该地址：
http://www.le.com/ptv/vplay/26482914.html

<embed bgcolor = "0x000000" src='http://player.56.com/v_MTQxNzgyODk1.swf' type='application/x-shockwave-flash' width='560' height='315'></embed>

<iframe bgcolor = "0x000000" width="560" height="315" src="https://www.youtube.com/embed/1LpJ-zd5XLE" frameborder="0" allowfullscreen></iframe>


我们的产品要求：
> 	1. 获取微信所有联系人
> 	2. 建立联系人“加密列表”
> 	3. 可以添加和删除指定的联系人到“加密列表”，操作时必须输入设置的密码
> 	4. 第一次使用用户有 “设置密码”按钮引导设置密码
> 	5. 已经设置密码的用户“设置密码”按钮变为“修改密码”按钮
> 	6. 已经在“加密列表”的联系人聊天界面显示全部是密文
> 	7. 在密文聊天界面长按聊天消息弹出解密提示，只有输正确了密码当前界面才会更新显示明文，在非解密的状态下聊天录音不予播放。
	
先大概说一下思路，我们需要通过逆向工程获取`WeChat`里面的所有头文件，通过`Cycript`调试`WeChat`去获取去定位里面的方法和界面布局等，还需要使用IEA去获取具体指定头文件方法的具体实现，这部分实现代码都是汇编代码。



# 	2. 微信砸壳

	
首先这个过程你需要一台越狱手机，然后通过数据线连接你的手机，当然你也可以使用SSH远程连接你的手机（速度很慢，所以我不喜欢），前提是必须在同一个局域网。下面我说说如何通过数据线连接越狱手机调试`WeChat`。
	
从`AppStore`上面下载的`app`需要砸壳，不然无法获取到里面的头文件，
打开mac终端

```
映射端口：./tcprelay.py -t 22:2222

```

![](https://github.com/wmf00032/imagesRecource/blob/master/WeChatImage/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%87.png?raw=true)
	
再按command+t打开一个新的终端

```
通过端口链接：ssh root@localhost -p 2222

```
![](https://github.com/wmf00032/imagesRecource/blob/master/WeChatImage/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%872.png?raw=true)
	
这个时候就已经连接到手机了，可以进行`cycript`调试指定的进程。
	
手机打开微信，运行着的app才能称之为进程，输入一下命令：

```
iPhone:~ root# ps -e | grep WeChat
23768 ??         1:18.84 /var/mobile/Containers/Bundle/Application/1D35C333-4F71-4DBB-89F7-8348A3130FD0/WeChat.app/WeChat

```
然后附着在WeChat进程

```
	-iPhone:~ root# cycript -p 23768
	cy# UIApp.keyWindow.recursiveDescription().toString()
	`<iConsoleWindow: 0x17d23ac0; baseClass = UIWindow; frame = (0 0; 320 568); autoresize = W+H; gestureRecognizers = <NSArray: 0x17d22b70>; layer = <UIWindowLayer: 0x17d237c0>>
	   | <UILayoutContainerView: 0x19297b30; frame = (0 0; 320 568); autoresize = W+H; layer = <CALayer: 0x193a3650>>
	   |    | <UITransitionView: 0x17f69b10; frame = (0 0; 320 568); clipsToBounds = YES; autoresize = W+H; layer = <CALayer: 0x191b8d00>>
	   |    |    | <UIViewControllerWrapperView: 0x191eee60; frame = (0 0; 320 568); autoresize = W+H; layer = <CALayer: 0x19142080>>
	   |    |    |    | <UILayoutContainerView: 0x17d178c0; frame = (0 0; 320 568); autoresize = W+H; gestureRecognizers = <NSArray: 0x1924fad0>; layer = <CALayer: 0x19272270>>
	   |    |    |    |    | <UINavigationTransitionView: 0x19231e20; frame = (0 0; 320 568); clipsToBounds = YES; autoresize = W+H; layer = <CALayer: 0x192053b0>>
	   |    |    |    |    |    | <UIViewControllerWrapperView: 0x191a8d20; frame = (0 0; 320 568); layer = <CALayer: 0x17f9f850>>
	   |    |    |    |    |    |    | <MMUIHookView: 0x190a9670; frame = (0 0; 320 519); layer = <CALayer: 0x19399010>>
	   |    |    |    |    |    |    |    | <MMMainTableView: 0x18c55000; baseClass = UITableView; frame = (0 0; 320 568); clipsToBounds = YES; gestureRecognizers = <NSArray: 0x17f4b330>; layer = <CALayer: 0x17f4b2c0>; contentOffset: {0, -64}; contentSize: {320, 499}>
	   |    |    |    |    |    |    |    |    | <UIView: 0x1924ad70; frame = (0 -64; 320 568); autoresize = W+H; layer = <CALayer: 0x19287470>>
	   |    |    |    |    |    |    |    |    |    | <UIView: 0x19287510; frame = (0 0; 320 64); alpha = nan; tag = 251658531; layer = <CALayer: 0x19287580>>
	   |    |    |    |    |    |    |    |    | <UITableViewWrapperView: 0x19161d20; frame = (0 0; 320 504); gestureRecognizers = <NSArray: 0x191a33f0>; layer = <CALayer: 0x191a3350>; contentOffset: {0, 0}; contentSize: {320, 504}>
	   |    |    |    |    |    |    |    |    |    | <NewMainFrameCell: 0x1941a5f0; baseClass = UITableViewCell; frame = (0 304; 320 65); autoresize = W; gestureRecognizers = <NSArray: 0x1941b2c0>; layer = <CALayer: 0x1941a570>>
```

23768是`WeChat`进程序，当然你也可以直接输入以下命令直接附着`WeChat`

```
-iPhone:~ root# cycript -p WeChat
cy# 
```

这个就省去了寻找进程号的步骤了。
`UIApp.keyWindow.recursiveDescription().toString()`
	
命令是递归打印当前界面的所有控件
关于`cycript`的使用请看文档：
http://www.cycript.org/
	
当然关于`cycript`的调试我们在后面会专门来细讲，这里我们主要是要获取到`WeChat`的沙盒路径，输入一下命令：

```
cy# [[NSFileManager defaultManager] URLsForDirectory:NSDocumentDirectory
                                              inDomains:NSUserDomainMask][0]
#"file:///var/mobile/Containers/Data/Application/DE952C97-6274-4F97-BC0D-B3D9D522FD9C/Documents/"

```
	
手机上`/var/mobile/Containers/Data/Application/DE952C97-6274-4F97-BC0D-B3D9D522FD9C/Documents/` 就是`WeChat`的沙盒路径
	
将`dumpdecrypted.dylib` 拷贝到`WeChat`的`Documents`的目录下面
拷贝有两种方式：
> 1. 使用scp命令
> 2. 使用PP助手直接找到WeChat的Documents目录进行拷贝


我们来到`Documents` 目录看一下

```
-iPhone:~ root# cd /var/mobile/Containers/Data/Application/DE952C97-6274-4F97-BC0D-B3D9D522FD9C/Documents/
-iPhone:/var/mobile/Containers/Data/Application/DE952C97-6274-4F97-BC0D-B3D9D522FD9C/Documents root# ls
00000000000000000000000000000000/  SafeMode.dat
CrashReport/                       b783b480395e7407834d7ca23f062501/
Ksid                               dumpdecrypted.dylib
LocalInfo.lst                      encription.fo
MMResourceMgr/                     mmupdateinfo.archive
MMappedKV/

```

`dumpdecrypted.dylib`动态库已经拷贝到了改目录。然后使用`dumpdecrypted.dylib`进行砸壳，命令：`DYLD_INSERT_LIBRARIES=dumpdecrypted.dylib` 可执行二进制文件
如下：

```
-iPhone:/var/mobile/Containers/Data/Application/DE952C97-6274-4F97-BC0D-B3D9D522FD9C/Documents root# DYLD_INSERT_LIBRARIES=dumpdecrypted.dylib /var/mobile/Containers/Bundle/Application/1D35C333-4F71-4DBB-89F7-8348A3130FD0/WeChat.app/WeChat
mach-o decryption dumper

DISCLAIMER: This tool is only meant for security research purposes, not for application crackers.

[+] detected 32bit ARM binary in memory.
[+] offset to cryptid found: @0x7ca4c(from 0x7c000) = a4c
[+] Found encrypted data at address 00004000 of length 40910848 bytes - type 1.
[+] Opening /private/var/mobile/Containers/Bundle/Application/1D35C333-4F71-4DBB-89F7-8348A3130FD0/WeChat.app/WeChat for reading.
[+] Reading header
[+] Detecting header type
[+] Executable is a FAT image - searching for right architecture
[+] Correct arch is at offset 16384 in the file
[+] Opening WeChat.decrypted for writing.
[+] Copying the not encrypted start of the file
[+] Dumping the decrypted data into the file
[+] Copying the not encrypted remainder of the file
[+] Setting the LC_ENCRYPTION_INFO->cryptid to 0 at offset 4a4c
[+] Closing original file
[+] Closing dump file
```
如上输出标示砸壳成功，提别提示！`WeChat.app`目录和 `Documents` 目录并不是同一个目录，看清楚了，上面出现的两个目录是不一样的。我们再看一下Documents 目录下面文件结构，输入命令如下：

```
9D522FD9C/Documents root# ls -l
total 100020
drwxr-xr-x  6 mobile mobile       272 Sep  3 10:44 00000000000000000000000000000000/
drwxr-xr-x  3 mobile mobile       136 Sep  3 17:38 CrashReport/
-rw-r--r--  1 mobile mobile       286 Sep  3 10:46 Ksid
-rw-r--r--  1 mobile mobile      1097 Sep  3 23:03 LocalInfo.lst
drwxr-xr-x  5 mobile mobile       272 Sep  3 23:03 MMResourceMgr/
drwxr-xr-x  2 mobile mobile       680 Sep  3 10:45 MMappedKV/
-rw-r--r--  1 mobile mobile        15 Sep  3 23:40 SafeMode.dat
-rw-r--r--  1 root   mobile 102197168 Sep  3 23:36 WeChat.decrypted
drwxr-xr-x 22 mobile mobile      1258 Sep  3 23:40 b783b480395e7407834d7ca23f062501/
-rw-r--r--  1 root   mobile    197528 Sep  3 23:32 dumpdecrypted.dylib
-rw-r--r--  1 mobile mobile       358 Sep  3 17:38 encription.fo
-rw-r--r--  1 mobile mobile       448 Sep  3 12:24 mmupdateinfo.archive

```
多了一个文件`WeChat.decrypted`，它就是我们砸壳解密后的`WeChat`二进制文件。将该文件`copy`到mac上面，我们要提取它里面的所有` .h` 头文件。

# class-dump 提取头文件
使用`class-dump`提取砸壳后的头文件

```
wangmengfa:~ Mango$ class-dump --help
class-dump 3.5 (64 bit)
Usage: class-dump [options] <mach-o-file>

  where options are:
        -a             show instance variable offsets
        -A             show implementation addresses
        --arch <arch>  choose a specific architecture from a universal binary (ppc, ppc64, i386, x86_64, armv6, armv7, armv7s, arm64)
        -C <regex>     only display classes matching regular expression
        -f <str>       find string in method name
        -H             generate header files in current directory, or directory specified with -o
        -I             sort classes, categories, and protocols by inheritance (overrides -s)
        -o <dir>       output directory used for -H
        -r             recursively expand frameworks and fixed VM shared libraries
        -s             sort classes and categories by name
        -S             sort methods by name
        -t             suppress header in output, for testing
        --list-arches  list the arches in the file, then exit
        --sdk-ios      specify iOS SDK version (will look in /Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS<version>.sdk
        --sdk-mac      specify Mac OS X version (will look in /Developer/SDKs/MacOSX<version>.sdk
        --sdk-root     specify the full SDK root path (or use --sdk-ios/--sdk-mac for a shortcut)

```

终端输入`class-dump`列出了它的用法，使用` -H ` 输入头文件，`-o` 是指定输出目录，`--arch <arch>`指定处理器架构。

开始操作，输入：

```
wangmengfa:~ Mango$ class-dump -H /Users/wangmengfa/Desktop/WeChat.decrypted -o /Users/wangmengfa/Desktop/myWeChatHeaders 

```

![](https://github.com/wmf00032/imagesRecource/blob/master/WeChatImage/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%873.png?raw=true)

输入如上命令，只在`myWeChatHeaders` 文件见下生成了一个`CDStructures.h`头文件，其它头文件尼？很显然，操作失败了。正确的操作应该是使用class-dump的--arch <arch>指定 WeChat的处理器架构。

先看一下`WeChat`支持的处理器架构，这里使用的是 `otool`命令：

```
wangmengfa:~ Mango$ otool -hV /Users/wangmengfa/Desktop/WeChat.decrypted
/Users/wangmengfa/Desktop/WeChat.decrypted (architecture armv7):
Mach header
      magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
   MH_MAGIC     ARM         V7  0x00     EXECUTE    75       7288   NOUNDEFS DYLDLINK TWOLEVEL WEAK_DEFINES BINDS_TO_WEAK PIE
/Users/wangmengfa/Desktop/WeChat.decrypted (architecture arm64):
Mach header
      magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
MH_MAGIC_64   ARM64        ALL  0x00     EXECUTE    75       8040   NOUNDEFS DYLDLINK TWOLEVEL WEAK_DEFINES BINDS_TO_WEAK PIE

```

好了，看的很清楚了，`WeChat` 这里支持 `armv7`、`arm64`两种架构。

再次使用`class-dump`命令导出头文件如下：

```
wangmengfa:~ Mango$ class-dump --arch armv7 -H /Users/wangmengfa/Desktop/WeChat.decrypted -o /Users/wangmengfa/Desktop/myWeChatHeaders 

```
输出了一大坨文件:

![](https://github.com/wmf00032/imagesRecource/blob/master/WeChatImage/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%874.png?raw=true)


接下来最好的就是讲这坨头文件导入到一个`Xcode`测试项目中去，方便查看与查找，很可惜这坨头文件实在太大啦，每次往`Xcode`项目中导入的时候都会把`Xcode`干崩溃。还是乖乖的在文件中查找吧。


# cycript调试
## 第一步获取所有的联系人信息

开始分析：
我们知道在微信的“通讯录”界面会显示当前用户的所有联系人，那么我们猜想：肯定在“通讯录”界面的代码中能够找到获取所有联系人的方法。
	
我们通过操作“通讯录”`Tab`发现，并不是每次点击这个`tab`都会请求网络，即使是`kill`应用重启，点击“通讯录”`Tab`也会闪电般的显示数据而没有任何请求网络的意思。很显然数据是在第一次请求网络获取联系人数据之后就保存到本地了，然后每次启动只要发现本地有联系人数据就从本地加载。
	
如果是你，开发这个“通讯录”`tab`你会在`Controller`启动的时候时机加载本地数据尼？
是的，很有可能是`- (void)viewDidLoad` 方法中。
	
首先，我们需要找到“通讯录”`tab` 的控制是哪个！
	
接下来进行`cycript`调试，寻找界面元素和数据，通过以下命令链接手机
> 
> 映射端口：./tcprelay.py -t 22:2222
> 
> 通过端口链接：ssh root@localhost -p 2222
> 

附着在`WeChat`上面

```
wangmengfa:python-client Mango$ ssh root@localhost -p 2222
root@localhost's password: 
wutde-iPhone:~ root# cycript -p WeChat
cy# 

```
然后通过`UIApp.keyWindow.recursiveDescription().toString()`命令打印当前界面所有元素。

<img src="https://github.com/wmf00032/imagesRecource/blob/master/WeChatImage/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%875.png?raw=true" width = "320" height = "568" alt="图片名称" align=center />
	
打印如下：

```
	<iConsoleWindow: 0x156f59a0; baseClass = UIWindow; frame = (0 0; 320 568); autoresize = W+H; gestureRecognizers = <NSArray: 0x156f6ab0>; layer = <UIWindowLayer: 0x156f5de0>>
	   | <UILayoutContainerView: 0x16a214d0; frame = (0 0; 320 568); autoresize = W+H; layer = <CALayer: 0x16a21470>>
	   |    | <UITransitionView: 0x16a21d20; frame = (0 0; 320 568); clipsToBounds = YES; autoresize = W+H; layer = <CALayer: 0x16a21f10>>
	   |    |    | <UIViewControllerWrapperView: 0x16b1fdc0; frame = (0 0; 320 568); autoresize = W+H; layer = <CALayer: 0x16aba1f0>>
	   |    |    |    | <UILayoutContainerView: 0x16a0ca90; frame = (0 0; 320 568); autoresize = W+H; gestureRecognizers = <NSArray: 0x16a10870>; layer = <CALayer: 0x16a0cb10>>
	   |    |    |    |    | <UINavigationTransitionView: 0x16a0dd30; frame = (0 0; 320 568); clipsToBounds = YES; autoresize = W+H; layer = <CALayer: 0x16a0ddc0>>
	   |    |    |    |    |    | <UIViewControllerWrapperView: 0x169fa770; frame = (0 0; 320 568); layer = <CALayer: 0x169fa7e0>>
	   |    |    |    |    |    |    | <MMUIHookView: 0x16c66080; frame = (0 0; 320 568); layer = <CALayer: 0x169f8770>>
	   |    |    |    |    |    |    |    | <MMMainTableView: 0x164e3000; baseClass = UITableView; frame = (0 0; 320 568); clipsToBounds = YES; gestureRecognizers = <NSArray: 0x169321e0>; layer = <CALayer: 0x156f7ca0>; contentOffset: {0, -64}; contentSize: {320, 6859}>
	   |    |    |    |    |    |    |    |    | <UIView: 0x16c59cf0; frame = (0 -64; 320 568); autoresize = W+H; layer = <CALayer: 0x169d2610>>
	   |    |    |    |    |    |    |    |    | <UITableViewWrapperView: 0x169be700; frame = (0 0; 320 568); gestureRecognizers = <NSArray: 0x16c4c6d0>; layer = <CALayer: 0x16c48630>; contentOffset: {0, 0}; contentSize: {320, 568}>
	   |    |    |    |    |    |    |    |    |    | <NewContactsItemCell: 0x16a02c50; baseClass = UITableViewCell; frame = (0 495; 320 55); autoresize = W; gestureRecognizers = <NSArray: 0x16af5f10>; layer = <CALayer: 0x16a02b50>>
	   |    |    |    |    |    |    |    |    |    |    | <UITableViewCellContentView: 0x16a02f40; frame = (0 0; 320 55); gestureRecognizers = <NSArray: 0x16a03140>; layer = <CALayer: 0x16a02fb0>>
	   |    |    |    |    |    |    |    |    |    |    |    | <UIView: 0x16a03b50; frame = (0 0; 320 55); layer = <CALayer: 0x16af5db0>>
	   |    |    |    |    |    |    |    |    |    |    |    |    | <ContactsItemView: 0x16890680; frame = (0 0; 320 55); layer = <CALayer: 0x16890740>>
	   |    |    |    |    |    |    |    |    |    |    |    |    |    | <MMHeadImageView: 0x16af9870; frame = (10 9.5; 36 36); layer = <CALayer: 0x16890860>>
	   |    |    |    |    |    |    |    |    |    |    |    |    |    |    | <MMUILongPressImageView: 0x16890770; baseClass = UIImageView; frame = (0 0; 36 36); opaque = NO; layer = <CALayer: 0x16890800>>
	   |    |    |    |    |    |    |    |    |    |    |    |    |    |    | <UIButton: 0x16af9950; frame = (0 0; 36 36); hidden = YES; opaque = NO; layer = <CALayer: 0x16af9a50>>
	   |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    | <UIImageView: 0x16ab8440; frame = (0 0; 36 36); clipsToBounds = YES; opaque = NO; userInteractionEnabled = NO; layer = <CALayer: 0x16ab84c0>>
	   |    |    |    |    |    |    |    |    |    |    |    |    |    | <MMCPLabel: 0x16af7200; baseClass = UILabel; frame = (56 5.5; 112 44); text = 'A-\u8303\u513f\ud83d\udc8b\ud83d\udc8b\ud83d\udc8b'; userInteractionEnabled = NO; layer = <_UILabelLayer: 0x16af7100>>
	   |    |    |    |    |    |    |    |    |    |    |    |    |    |    | <_UILabelContentLayer: 0x16b15930> (layer)
	   |    |    |    |    |    |    |    |    |    |    | <_UITableViewCellSeparatorView: 0x16af7dd0; frame = (10 54.5; 310 0.5); layer = <CALayer: 0x16af7e50>>
	   |    |    |    |    |    |    |    |    |    | <NewContactsItemCell: 0x16af8100; baseClass = UITableViewCell; frame = (0 440; 320 55); autoresize = W; gestureRecognizers = <NSArray: 0x16ab20b0>; layer = <CALayer: 0x16b64610>>
	   |    |    |    |    |    |    |    |    |    |    | <UITableViewCellContentView: 0x168531c0; frame = (0 0; 320 55); gestureRecognizers = <NSArray: 0x16837eb0>; layer = <CALayer: 0x16853230>>
	   |    |    |    |    |    |    |    |    |    |    |    | <UIView: 0x16ad6980; frame = (0 0; 320 55); layer = <CALayer: 0x16ad69f0>>
	   |    |    |    |    |    |    |    |    |    |    |    |    | <ContactsItemView: 0x16b415e0; frame = (0 0; 320 55); layer = <CALayer: 0x16b416a0>>
	   |    |    |    |    |    |    |    |    |    |    |    |    |    | <MMHeadImageView: 0x16a003f0; frame = (10 9.5; 36 36); layer = <CALayer: 0x16b416d0>>
	   |    |    |    |    |    |    |    |    |    |    |    |    |    |    | <MMUILongPressImageView: 0x1688bd70; baseClass = UIImageView; frame = (0 0; 36 36); opaque = NO; layer = <CALayer: 0x16a004d0>>
	   |    |    |    |    |    |    |    |    |    |    |    |    |    |    | <UIButton: 0x16b1a280; frame = (0 0; 36 36); hidden = YES; opaque = NO; layer = <CALayer: 0x16a00500>>
	   |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    | <UIImageView: 0x16af6670; frame = (0 0; 36 36); clipsToBounds = YES; opaque = NO; userInteractionEnabled = NO; layer = <CALayer: 0x16af6740>>
	   |    |    |    |    |    |    |    |    |    |    |    |    |    | <MMCPLabel: 0x16af82d0; baseClass = UILabel; frame = (56 5.5; 62 44); text = 'AA_\u6b23\u6b23'; userInteractionEnabled = NO; layer = <_UILabelLayer: 0x16b4bd40>>
	   |    |    |    |    |    |    |    |    |    |    |    |    |    |    | <_UILabelContentLayer: 0x16c495c0> (layer)
	   |    |    |    |    |    |    |    |    |    |    | <_UITableViewCellSeparatorView: 0x16a008a0; frame = (10 54.5; 310 0.5); layer = <CALayer: 0x16a00920>>
	
```
	
输出了一堆界面元素和相关信息，信息很多我这而只粘出部分数据，当前界面的层级结构已经很清楚了，`NewContactsItemCell` 就是用来显示当前联系人的`cell` ，`MMMainTableView`就是当前界面的`tableview`。但是这里显示的都是View信息，并没有直接显示`controller` , 接下来我们要`view`信息寻找到当前界面`controller`。
	
我们看见`MMMainTableView`的信息是这样的

```
<MMMainTableView: 0x164e3000; baseClass = UITableView; frame = (0 0; 320 568); clipsToBounds = YES; gestureRecognizers = <NSArray: 0x169321e0>; layer = <CALayer: 0x156f7ca0>; contentOffset: {0, -64}; contentSize: {320, 6859}>

```
其中`0x164e3000`表示的是`MMMainTableView`内存地址，我们可以通过0x164e3000操作`MMMainTableView`对象。
输入一下命令：

```
cy# [#0x164e3000 nextResponder]
#"<MMUIHookView: 0x16c66080; frame = (0 0; 320 568); layer = <CALayer: 0x169f8770>>"
cy# [#0x16c66080 nextResponder]
#"<ContactsViewController: 0x15bb3a00>"
```
	
好了，通过`nextResponder`寻找下一个事件响应者，我们找到了当前界面的`controller`就是 `ContactsViewController`
	
	
# IDA分析汇编代码
接下来我们要分析一下`ContactsViewController` 是如何加载所有的联系人数据的。
将砸壳后的`WeChat.decrypted`二进制文件用`IDA`打开，将会反编译出所有的文件的汇编代码。我们找到`ContactsViewController` 的`viewDidLoad`方法。看到汇编代码如下：
	
![](https://github.com/wmf00032/imagesRecource/blob/master/WeChatImage/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%876.png?raw=true)
	
非常明确了，在`ContactsViewController` 的viewDidLoad方法中做了几件很简单的事情：

> 	1. 第一步是调用父类的viewDidLoad方法
> 	2. 第二步是调用了 initData方法
> 	3. 第三步是调用了initView方法
> 	4. 第四步是调用了[MMServiceCenter defaultCenter] 方法然后通过getService方法获取到了有个服务
	
是的，用屁股想都能猜到，`initData`方法是用来初始化界面显示数据的。那么很有可能就是在这里面获取获取到当前界面所有联系人数据的。
	
同理查看`initData`汇编代码：
	
![](https://github.com/wmf00032/imagesRecource/blob/master/WeChatImage/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%877.png?raw=true)

![](https://github.com/wmf00032/imagesRecource/blob/master/WeChatImage/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%878.png?raw=true)
	
   
从代码的执行来看，`initData`会先判断 `m_contactsDataLogic` 成员变量是否为空，如果是空，则说明什么也不执行，如果不是空，就会执行创建 `m_contactsDataLogic`变量的工作。 `m_contactsDataLogic`变量是干嘛的尼？以至于初始化数据方法基本是围绕创建它的，从字面理解Logic是“逻辑学”的意思， `m_contactsDataLogic`大概意思就是“联系人数据逻辑操作”，是的联系人操作的工具类。从`m_contactsDataLogic`的创建方法来看它的类型是 `ContactsDataLogic`。我们在头文件中找到`ContactsDataLogic`类。
	
![](https://github.com/wmf00032/imagesRecource/blob/master/WeChatImage/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%879.png?raw=true)
	
里面有两个方法很诱惑人：
	
```	
	- (id)getAllContacts;
	- (id)getAllContactsDictionary;

```
还是那句话，用屁股想都能想到是干嘛的。除了获取所有联系人方法，里面还有其它对联系人操作的方法比如增加、删除、查找、修改等，我这儿只截出了一部分。
下面我们要验证一下上面获取所有联系人得方法是不是真的能够获取到所有联系人。
	
链接手机输入`cycript` 命令如下：

```
cy# [#0x16c66080 nextResponder]
#"<ContactsViewController: 0x15bb3a00>"
cy# var contactLogic = [#0x15bb3a00 valueForKey:@"m_contactsDataLogic"]
#"<ContactsDataLogic: 0x169d0960>"
```
	获取到了`ContactsViewController` 的 `m_contactsDataLogic`属性，然后我们调用`getAllContacts`方法看能不能打印出所有的联系人

```

cy# [#0x169d0960 getAllContacts]
@[#"{m_nsUsrName=wxi*c21~19, m_nsEncodeUserName=v1_b32e1ed63dee45756bf04c925bac2b3384e42addbd76ebfab3e4b31ee514f64e54e736a0a35f3787bd9f94dd1ff8865c@stranger, alias=kao*eld~8, m_nsNickName=\xe9\xab\x98\xe9\xa3\x9e, m_uiType=3, m_uiConType=0, m_nsRemark=,  m_nsCountry=CN m_nsProvince=Beijing m_nsCity=Chaoyang m_nsSignature=\xe5\x8c\x85\xe5\xae\xb9\xe4\xb8\x8d\xe8\x83\xbd\xe6\x94\xb9\xe5\x8f\x98\xe7\x9a\x84\xef\xbc\x8c\xe5\x8a\xaa\xe5\x8a\x9b\xe6\x94\xb9\xe5\x8f\x98\xe5\x8f\xaf\xe4\xbb\xa5\xe6\x94\xb9\xe5\x8f\x98\xe7\x9a\x84\xef\xbc\x8c\xe7\x94\xa8\xe6\x99\xba\xe6\x85\xa7\xe5\x8c\xba\xe5\x88\x86\xe5\x93\xaa\xe4\xba\x9b\xe5\x8f\xaf\xe4\xbb\xa5\xe6\x94\xb9\xe5\x8f\x98 \t m_uiSex=1 m_uiCerFlag=0 m_nsCer= scene=3 }",#"{m_nsUsrName=wxi*222~19, m_nsEncodeUserName=v1_c78736e3a7f04cf39a750dd214e842bb2cb65276dcd231ed4e1d1002156f1488a91201e8bd653f895de2d97b0c5ac7ad@stranger, alias=, m_nsNickName=\xe6\x9e\x9c\xe5\xad\x90, m_uiType=3, m_uiConType=0, m_nsRemark=\xe5\x93\xa5\xe5\x93\xa5,  m_nsCountry= m_nsProvince= m_nsCity= m_nsSignature= \t m_uiSex=0 m_uiCerFlag=0 m_nsCer= scene=10 }",#"{m_nsUsrName=wxi*i22~19, m_nsEncodeUserName=v1_d50b3448e35f1cd099956a44c5e5410b81a7d937b0e246436a985dff45cb51597cbee5429e9f8f34c14c4e254bc88ab8@stranger, alias=nih*520~13, m_nsNickName=\xe6\x9e\x9c\xe5\xad\x90, m_uiType=3, m_uiConType=0, m_nsRemark=\xe5\x93\xa5\xe5\x93\xa5,  m_nsCountry=CN
```	
一堆联系人信息输入出来。证明我们以上的猜想是正确的。（真实联系人敏感数据，我只截出了极少的一部分）只要我们获取到`ContactsDataLogic`对象，就可以自由的操作所有联系人数据了。
	
那么怎么样创建`ContactsDataLogic`对象尼？从`ContactsViewController` 的`initData`汇编代码实现来看，`ContactsDataLogic`是通过自身的方法 `initWithScene:delegate:sort:extendChatRoom:`方法创建的，下面我们来验证一下是否如此。

``` 

cycript操作如下：
cy# var myNewContactLogic = [[ContactsDataLogic alloc] initWithScene:nil delegae:nil sort:0 extendChatRoom:NO]
#"<ContactsDataLogic: 0x16a0bdc0>"
cy# [myNewContactLogic getAllContacts]
@[#"{m_nsUsrName=wxi*c21~19, m_nsEncodeUserName=v1_b32e1ed63dee45756bf04c925bac2b3384e42addbd76ebfab3e4b31ee514f64e54e736a0a35f3787bd9f94dd1ff8865c@stranger, alias=kao*eld~8, m_nsNickName=\xe9\xab\x98\xe9\xa3\x9e, m_uiType=3, m_uiConType=0, m_nsRemark=,  m_nsCountry=CN m_nsProvince=Beijing m_nsCity=Chaoyang m_nsSignature=\xe5\x8c\x85\xe5\xae\xb9\xe4\xb8\x8d\xe8\x83\xbd\xe6\x94\xb9\xe5\x8f\x98\xe7\x9a\x84\xef\xbc\x8c\xe5\x8a\xaa\xe5\x8a\x9b\xe6\x94\xb9\xe5\x8f\x98\xe5\x8f\xaf\xe4\xbb\xa5\xe6\x94\xb9\xe5\x8f\x98\xe7\x9a\x84\xef\xbc\x8c\xe7\x94\xa8\xe6\x99\xba\xe6\x85\xa7\xe5\x8c\xba\xe5\x88\x86\xe5\x93\xaa\xe4\xba\x9b\xe5\x8f\xaf\xe4\xbb\xa5\xe6\x94\xb9\xe5\x8f\x98 \t m_uiSex=1 m_uiCerFlag=0 m_nsCer= scene=3 }",#"{m_nsUsrName=wxi*222~19, m_nsEncodeUserName=v1_c78736e3a7f04cf39a750dd214e842bb2cb65276dcd231ed4e1d1002156f1488a91201e8bd653f895de2d97b0c5ac7ad@stranger, alias=, m_nsNickName=\xe6\x9e\x9c\xe5\xad\x90, m_uiType=3, m_uiConType=0, m_nsRemark=\xe5\x93\xa5\xe5\x93\xa5,  m_nsCountry= m_nsProvince= m_nsCity= m_nsSignature= \t m_uiSex=0 m_uiCerFlag=0 m_nsCer= scene=10 }",

```
	
是的，可以获取到正确的联系人数据。再一次证明我们的推断是正确的。我们查看ContactsDataLogic 头文件发现：

```

- (id)initWithScene:(unsigned long)arg1 delegate:(id)arg2;
- (id)initWithScene:(unsigned long)arg1 delegate:(id)arg2 sort:(BOOL)arg3 extendChatRoom:(BOOL)arg4;
- (id)initWithScene:(unsigned long)arg1 delegate:(id)arg2 sort:(BOOL)arg3;
```

这里面有三个`initWithScene`方法，其实我们只要使用参数最少那个就可以创建`ContactsDataLogic` 对象。



**郑重声明：未经本人允许严禁任何个人或组织转载！**