---
title: （非越狱版）iOS微信聊天加密插件 （二）
toc: true
date: 2016-09-06 16:15:18
categories:
tags:
---


# 定位首页Controller
前面我们已经找到如何获取`WeChat`全部联系人的方法，下面我们来这手新建项目并且尝试着将其编译成我们自己的动态库，并将其注入到`WeChat`二进制文件之中，然后打包安装到非越狱手机上面。

先明确一下我们的目标，现在我想在`WeChat`首页左上角增加一个按钮，按钮点击按钮进入一个我们自己写的新界面，里面能够展示所有的联系人列表。


<img src="https://github.com/wmf00032/imagesRecource/blob/master/wechat2_images/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%87.png?raw=true" width = "320" height = "568" alt="图片名称" align=center />

要想在首页左上角增加一个按钮，就必须要先回去到首页`Controller`。同理通过`Cycript`调试寻找首页`Controller` ：

```

cy# UIApp.keyWindow.recursiveDescription().toString()
`<iConsoleWindow: 0x156f59a0; baseClass = UIWindow; frame = (0 0; 320 568); autoresize = W+H; gestureRecognizers = <NSArray: 0x156f6ab0>; layer = <UIWindowLayer: 0x156f5de0>>
   | <UILayoutContainerView: 0x16be77d0; frame = (0 0; 320 568); autoresize = W+H; layer = <CALayer: 0x16be0790>>
   |    | <UITransitionView: 0x16f27a60; frame = (0 0; 320 568); clipsToBounds = YES; autoresize = W+H; layer = <CALayer: 0x16cd07b0>>
   |    |    | <UIViewControllerWrapperView: 0x16ca4380; frame = (0 0; 320 568); autoresize = W+H; layer = <CALayer: 0x1697abe0>>
   |    |    |    | <UILayoutContainerView: 0x16bcb620; frame = (0 0; 320 568); autoresize = W+H; gestureRecognizers = <NSArray: 0x16aaef60>; layer = <CALayer: 0x16a7a260>>
   |    |    |    |    | <UINavigationTransitionView: 0x16b5bd20; frame = (0 0; 320 568); clipsToBounds = YES; autoresize = W+H; layer = <CALayer: 0x1706fe30>>
   |    |    |    |    |    | <UIViewControllerWrapperView: 0x16bf24a0; frame = (0 0; 320 568); layer = <CALayer: 0x16bf2510>>
   |    |    |    |    |    |    | <MMUIHookView: 0x16fddc40; frame = (0 0; 320 519); layer = <CALayer: 0x16cc2910>>
   |    |    |    |    |    |    |    | <MMMainTableView: 0x15df4600; baseClass = UITableView; frame = (0 0; 320 568); clipsToBounds = YES; gestureRecognizers = <NSArray: 0x16a131b0>; layer = <CALayer: 0x16a13140>; contentOffset: {0, -64}; contentSize: {320, 499}>
   |    |    |    |    |    |    |    |    | <UIView: 0x16c88190; frame = (0 -64; 320 568); autoresize = W+H; layer = <CALayer: 0x16c88200>>
   |    |    |    |    |    |    |    |    |    | <UIView: 0x1696ba60; frame = (0 0; 320 64); alpha = nan; tag = 251658531; layer = <CALayer: 0x16c249e0>>
   |    |    |    |    |    |    |    |    | <UITableViewWrapperView: 0x16b8fae0; frame = (0 0; 320 504); gestureRecognizers = <NSArray: 0x16a20600>; layer = <CALayer: 0x16bab380>; contentOffset: {0, 0}; contentSize: {320, 504}>
   |    |    |    |    |    |    |    |    |    | <NewMainFrameCell: 0x16aca000; baseClass = UITableViewCell; frame = (0 369; 320 65); autoresize = W; gestureRecognizers = <NSArray: 0x169c57b0>; layer = <CALayer: 0x16b5ce80>>
   |    |    |    |    |    |    |    |    |    |    | <UITableViewCellContentView: 0x16b5cf70; frame = (0 0; 320 65); gestureRecognizers = <NSArray: 0x17056550>; layer = <CALayer: 0x16b5cfe0>>
   |    |    |    |    |    |    |    |    |    |    |    | <UIView: 0x16cbee30; frame = (0 0; 320 65); layer = <CALayer: 0x16cb3eb0>>
   |    |    |    |    |    |    |    |    |    |    |    |    | <MainFrameItemView: 0x16ce2000; frame = (0 0; 320 65); tag = 1001; layer = <CALayer: 0x16ce20a0>>
   |    |    |    |    |    |    |    |    |    |    |    |    |    | <MMHeadImageView: 0x16ce20d0; frame = (10 10; 45 45); layer = <CALayer: 0x16f8bf50>>
   |    |    |    |    |    |    |    |    |    |    |    |    |    |    | <MMUILongPressImageView: 0x16f8bf80; baseClass = UIImageView; frame = (0 0; 45 45); opaque = NO; layer = <CALayer: 0x16f8c010>>
   |    |    |    |    |    |    |    |    |    |    |    |    |    |    | <UIButton: 0x16f8c040; frame = (0 0; 45 45); hidden = YES; opaque = NO; layer = <CALayer: 0x16c79240>>
   |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    | <UIImageView: 0x16c8dd30; frame = (0 0; 45 45); clipsToBounds = YES; opaque = NO; userInteractionEnabled = NO; layer = <CALayer: 0x16c8ddb0>>

```

其中当前界面的`tableview`信息如下：

```
<MMMainTableView: 0x15df4600; baseClass = UITableView; frame = (0 0; 320 568); clipsToBounds = YES; gestureRecognizers = <NSArray: 0x16a131b0>; layer = <CALayer: 0x16a13140>; contentOffset: {0, -64}; contentSize: {320, 499}>
接下来通过nextResponder往下寻找：
cy# [#0x15df4600 nextResponder]
#"<MMUIHookView: 0x16fddc40; frame = (0 0; 320 519); layer = <CALayer: 0x16cc2910>>"
cy# [#0x16fddc40 nextResponder]
#"<NewMainFrameViewController: 0x15dbde00>"

```

好了，原来首页`Controller`就是`NewMainFrameViewController`

# 新建Tweak项目

选择`iOSOpenDev` ->`Logos Tweak` 

![](https://github.com/wmf00032/imagesRecource/blob/master/wechat2_images/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%872.png?raw=true)

当然你需要先安装 `iOSOpenDev` ，你也可以选择使用命令行新建Tweak项目，就像这样：

```

wangmengfa:~ Mango$ /opt/theos/bin/nic.pl
NIC 2.0 - New Instance Creator
------------------------------
  [1.] iphone/application
  [2.] iphone/library
  [3.] iphone/preference_bundle
  [4.] iphone/tool
  [5.] iphone/tweak
Choose a Template (required): 
```

选择`iphone/tweak` ，然后按照提示一步一步往下就能新建一个Tweak项目。（关于`iOSOpenDev` 和`Theos`的安装请自行百度）

新建的项目如下：

![](https://github.com/wmf00032/imagesRecource/blob/master/wechat2_images/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%873.png?raw=true)

点击`command+B` 运行报一下错误

![](https://github.com/wmf00032/imagesRecource/blob/master/wechat2_images/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%874.png?raw=true)

只要按照`.xm`文件里面的提示：

```
#error iOSOpenDev post-project creation from template requirements (remove these lines after completed) -- \
Link to libsubstrate.dylib: \
(1) go to TARGETS > Build Phases > Link Binary With Libraries and add /opt/iOSOpenDev/lib/libsubstrate.dylib \
(2) remove these lines from *.xm files (not *.mm files as they're automatically generated from *.xm files)

```

按照以上提示去操作编译就能通过了。


新建一个控制器`EncryptionViewController` 用来显示我们所有的联系人，其中`ContactsDataLogic.h、CContact.h、CBaseContact.h`使我们引用的微信中的头文件。项目结构如下：

![](https://github.com/wmf00032/imagesRecource/blob/master/wechat2_images/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%875.png?raw=true)

现在开始写我们自己的代码，`TestWeChat.xm`里面的代码如下：

```

#import "EncryptionViewController.h"

static UIBarButtonItem *leftBarItem;
%hook NewMainFrameViewController

%new
- (void)encryptionLeftItemClickBtn{
    
    UIViewController *controller = (UIViewController*)self;
    /* 进入加密名单列表 */
    EncryptionViewController *encryptionController = [[EncryptionViewController alloc] initWithStyle:UITableViewStyleGrouped];
    encryptionController.hidesBottomBarWhenPushed = YES;
    [controller.navigationController pushViewController:encryptionController animated:YES];
    [encryptionController release];
}

- (void)viewDidAppear:(BOOL)animated{
    
    %orig;
    
    if(!leftBarItem){
        UIButton *leftBtn = [[UIButton alloc] initWithFrame:CGRectMake(0, 0, 55, 44)];
        [leftBtn setTitleColor:[UIColor whiteColor] forState:UIControlStateNormal];
        leftBtn.titleLabel.font = [UIFont systemFontOfSize:13];
        [leftBtn setTitle:@"爱你加密" forState:UIControlStateNormal];
        [leftBtn addTarget:self action:@selector(encryptionLeftItemClickBtn) forControlEvents:UIControlEventTouchUpInside];
        
        leftBarItem = [[UIBarButtonItem alloc] initWithCustomView:leftBtn];
    }
    [[self navigationItem] setLeftBarButtonItem:leftBarItem];
    
}
%end

```

这里特别值得说明首页新增加的`leftBtn`放在`viewDidAppear`方法中，这是因为经过本人反复尝试发现，如果直接放在`viewDidLoad`方法中创建，微信将会在后面更新界面的时候将其冲刷掉。读者可以自己亲自试验一下，虽然这里看起来很怪！

`EncryptionViewController.m` 的实现代码如下：

```

#import "EncryptionViewController.h"
#import "ContactsDataLogic.h"
#import "CContact.h"

@interface EncryptionViewController ()

@end

@implementation EncryptionViewController
{
    NSArray *_contactsArray;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    ContactsDataLogic *contactsDataLogic = [[NSClassFromString(@"ContactsDataLogic") alloc] initWithScene:0
                                                                                                 delegate:nil];
    _contactsArray = [contactsDataLogic getAllContacts];
}


#pragma mark - Table view data source

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {

    return _contactsArray.count;
}
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath{
    return 54;
}


- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    
    UITableViewCell* cell = [tableView dequeueReusableCellWithIdentifier:@"cellIdentifier"];
    if (cell == nil) {
        cell = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleDefault reuseIdentifier:@"cellIdentifier"];
    }
    CContact *contact = _contactsArray[indexPath.row];
    cell.textLabel.text = contact.m_nsNickName;
    
    return cell;
}

@end

```

是的，代码非常少，相信大家一看就明白!

接下来按`command+B` 编译后生成 `TestWeChat.dylib` 动态库，找到动态库存放目录，如下：

<img src="https://github.com/wmf00032/imagesRecource/blob/master/wechat2_images/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%876.png?raw=true" width = "320" height = "568" alt="图片名称" align=center />

![](https://github.com/wmf00032/imagesRecource/blob/master/wechat2_images/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%877.png?raw=true)

`otool、testbs.sh、WeChat`文件是我`copy`进来的。 其中`WeChat`是二进制文件，`otool` 是是动态库插入工具（类似的工具还有`yololib、insert_dylib` 等），`testbs.sh`是我自己写的脚本，只要在终端运行：`bash testbs.sh `就会自动将动态库`TestWeChat.dylib`进行签名，然后注入`WeChat`二进制文件，再将`WeChat`二进制文件重新签名打包成  `.ipa`包，最后自动安装到手机。



安装后像这个样子：

<img src="https://github.com/wmf00032/imagesRecource/blob/master/wechat2_images/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%878.png?raw=true" width = "320" height = "568" alt="图片名称" align=center />

<img src="https://github.com/wmf00032/imagesRecource/blob/master/wechat2_images/%E6%9C%AA%E5%91%BD%E5%90%8D%E5%9B%BE%E7%89%879.png?raw=true" width = "320" height = "568" alt="图片名称" align=center />



#  一键自动化脚本
下面来解释一下上面提到的自动化脚本，先看一下`testbs.sh`文件如下：

```

#!bash

#set -e

# 注意变量赋值=左右不能有空格
## 项目路径
PROJECT_PATH=/Users/wangmengfa/Library/Developer/Xcode/DerivedData/TestWeChat-fbcbnefjbnbhkaclrzsccyotvlzq/Build/Products/Debug-iphoneos
# dylib name
DYLIB_NAME=TestWeChat.dylib
# dylib path
DYLIB_PATH=${PROJECT_PATH}/${DYLIB_NAME}
#/obj/
# 要插入的 .app 名称，列入这儿是：WeChat.app
TERGET_APP_NAME=WeChat

## 进行签名的密钥
DIST_SIGN='iPhone Distribution: BeiJing XXXX'


cd ${PROJECT_PATH}
echo "项目："
pwd


if [ -f "${DYLIB_PATH}" ];
then
echo "file exist !"
echo "this file path: ${DYLIB_PATH}"
else
echo "${DYLIB_NAME} file not exit !"
exit -1
fi


#卸载之前的dylib(先判断是否存在)

install_bylib=${TERGET_APP_NAME}.app/${DYLIB_NAME}

if [ -f "${install_bylib}" ];
then
echo "${install_bylib} aready file exist !"
echo "this file path: ${install_bylib}"

echo -e "---start----uninstall ${DYLIB_NAME}"
./optool uninstall -p "@executable_path/${DYLIB_NAME}" -t ${TERGET_APP_NAME}.app/${TERGET_APP_NAME}
echo -e "---success----卸载之前dylib\n"

fi

# copy最新的
echo "copy ${DYLIB_NAME} to ${TERGET_APP_NAME}.app"
cp -rf ${DYLIB_PATH} ${TERGET_APP_NAME}.app/
echo "copy success!"


# 动态库插入
./optool install -c load -p "@executable_path/${DYLIB_NAME}" -t ${TERGET_APP_NAME}.app/${TERGET_APP_NAME}
echo -e "---success----动态库插入\n"

# 动态库签名
codesign -f -s ${DIST_SIGN} ${TERGET_APP_NAME}.app/${DYLIB_NAME}

echo -e "---success----动态库签名完成\n"

# app签名
codesign -f -s ${DIST_SIGN} --entitlements Entitlements.plist ${TERGET_APP_NAME}.app
echo -e "---success----app签名完成\n"

# 生成ipa包
echo ${MYHOME_PATH}/${TERGET_APP_NAME}.app
xcrun -sdk iphoneos PackageApplication -v ${MYHOME_PATH}/${TERGET_APP_NAME}.app ${MYHOME_PATH}/${TERGET_APP_NAME}.ipa
echo -e "---success----生成ipa包\n"

# 安装
echo -e "---start install ipa----"
echo -e "---wait a moment----"
mobiledevice install_app ${MYHOME_PATH}/${TERGET_APP_NAME}.ipa
echo -e "---success----安装完成！\n"

```

其中`PROJECT_PATH`是当前动态库和`WeChat`二进制文件所在的目录，`DYLIB_NAME`是待注入动态库的名称，`TERGET_APP_NAME`是待注入二进制文件的名称，这里的`TERGET_APP_NAME`就是`WeChat`.

其中脚本：

```

if [ -f "${install_bylib}" ];
then
echo "${install_bylib} aready file exist !"
echo "this file path: ${install_bylib}"

echo -e "---start----uninstall ${DYLIB_NAME}"
./optool uninstall -p "@executable_path/${DYLIB_NAME}" -t ${TERGET_APP_NAME}.app/${TERGET_APP_NAME}
echo -e "---success----卸载之前dylib\n"

fi

```

在注入动态库文件之前，先检测当前二进制文件中是否已经包含了改动态库，如果已经包含了就先卸载之前的，如果没有包含就直接注入。

注入之后一定记得要将动态库和二进制文件进行签名，最后使用  `mobiledevice` 工具 将新生成的 .ipa包安装到手机上面（你也可以选择使用 `Itunes` 进行安装）。

**郑重声明：未经本人允许严禁任何个人或组织转载！**