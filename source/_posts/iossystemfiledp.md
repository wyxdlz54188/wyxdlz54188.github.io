---
title: iOS系统文件目录解析
date: 2025-01-05 2:07:02
tags: iOS玩机
categories: iOS社区
---

1、【/Applications】 常用软件的安装目录

2. 【/private /var/ mobile/Media /iphone video Recorder】录像文件存放目录

3、【/private /var/ mobile/Media /DCIM】相机拍摄的照片文件存放目录

4、【/private/var/ mobile /Media/iTunes_Control/Music】iTunes上传的多媒体文件（例如MP3、MP4等）存放目录，文件没有被修改，但是文件名字被修改了，直接下载到电脑即可读取。

5、【/private /var/root/Media/EBooks】电子书存放目录

6、【/Library/Ringtones】系统自带的来电铃声存放目录（用iTunes将文件转换为ACC文件，把后缀名改成.m4r,用iPhone_PC_Suite传到/Library/Ringtones即可）

7、【/private/var/ mobile /Library/AddressBook】系统电话本的存放目录。

8、【/private /var/ mobile/Media /iphone Recorder】录音文件存放目录

9、【/Applications/Preferences.app/zh_CN.lproj】软件Preferences.app的中文汉化文件存放目录

10、【/Library/Wallpaper】系统墙纸的存放目录

11、【/System/Library/Audio/UISounds】系统声音文件的存放目录

12、【/private/var/root/Media/PXL】上传安装程序建立的一个数据库

13、【/bin】和linux系统差不多，是系统执行指令的存放目录。

14、【/private/var/ mobile /Library/SMS】系统短信的存放目录

15、【/private/var/run】系统进程运行的临时目录（查看系统启动的所有进程）

16、【/private/var/logs/CrashReporter】系统错误记录报告、

17. 这个电池图标存放自用的主题目录下，/var/stash/Themes.xxxxx/自用主题目录/Bundles/com.apple.springboard/目录内上传BatteryBG_1-17.png图片即可，如无com.apple.springboard目录，请自建。（Themes.xxxxx每个人都是不一样的，故用xxxxx表示）也可以直接替换/System/Library/CoreServices/SpringBoard.app图标替换路径
WB相关主题直接放在Library/Themes里面注意修改权限

18、充电图标：System/Library/CoreServices/SpringBoard.app/BatteryBG_1.png 一直到 BatteryBG_17.png ，Batteryfill.png

19、手机信号图标：/System/Library/CoreServices/SpringBoard.app/下Default_0_Bars.png一直到Default_5_Bars.png 和FSO_0_Bars.png--FSO_5_Bars.png

20、Wifi信号图标

:/System/Library/CoreServices/SpringBoard.app/Default_0_AirPort.png

Default_3_AirPort.png和FSO_0_AirPort.png---FSO_3_AirPort.png

21、Edge信号图标:/System/Library/CoreServices/SpringBoard.app/ Default_EDGE_ON.png和FSO_EDGE_ON.png 2图标为Edge信号图标

22、日期美化图标：/System/Library/CoreServices/SpringBoard.app|FSO_LockIcon.png

23、待机播放器图标：/System/Library/CoreServices/SpringBoard.app|nexttrack.png , pause.png , play.png, prevtrack.png 4个图标为待机播放器图标

24、IPOD播放信号 /System/Library/CoreServices/SpringBoard.app/FSO_Play.png ,Default_Play.png

25、闹钟信号 /System/Library/CoreServices/SpringBoard.app/Default_AlarmClock.png ,FSO_AlarmClock.png

26、震动图标

/System/Library//CoreServices/SpringBoard.app/silent.png ,hud.png ,ring.png

27、滑块图标：

/System/Library/PrivateFrameworks/TelephonyUI.framework 目录下

Bottombarknobgray.png（待机解锁滑块图标）

bottombarknobgreen.png（待机状态下移动滑动来接听 滑块图标)

Bottombarknobred.png(关机滑块 图标）

28、待机时间字体：/System/Library/Fonts/Cache/LockClock.ttf

29、待机时间背景：system/library/Frameworks/UIKit.framework/Other.artwork

30、滑块文字变为闪光字: /System/Library/PrivateFrameworks/TelephonyUI.framework/bottombarlocktextmask.png

32、解锁滑条路径: /System/Library/PrivateFrameworks/TelephonyUI.framework//topbarbkgnd.png ,bottombarbkgndlock.png

33、 修复系统菜单 删除文件/private/var/mobile/Library/Caches/com.apple.mobile.installation.plist，然后重新启动iPhone
