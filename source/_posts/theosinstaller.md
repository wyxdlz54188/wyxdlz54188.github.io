---
title: Theos的安装及theosinstaller使用教程
date: 2025-01-04 11:07:02
tags: Theos开发
categories: iOS社区
---

## 前言
准备工具：
一台可以使用(巨魔/越狱)的🍎苹果设备&&越狱最好,巨魔可越狱者尽量越狱,不可越狱者安装bootstarp最新版本

### 安装
那么想必大家已经准备好了吧！
打开sileo越狱商店添加插件源
添加李子源到[sileo](sileo://source/https://wyxdlz54188.github.io/repo/)
李子源
``` bash
 https://wyxdlz54188.github.io/repo/
```
1.打开插件源安装theos依赖及theosinstaller
2.安装***Newterm3*** 
无根越狱者用插件源里的插件半越狱/roothide隐根 者安装roothide源的Newterm3插件
[theosinstaller下载](https://wyxdlz54188.github.io/repo/debs/theosinstaller.deb)
[Newterm3无根专用](https://wyxdlz54188.github.io/repo/debs/ws.hbang.newterm3_3.0_iphoneos-arm64.deb)
[Newterm3汉化](https://wyxdlz54188.github.io/repo/debs/lizi.newterm.cn_1.0_iphoneos-arm64.deb)
3.打开Newterm3输入
``` bash
 sh theosinstaller.sh
```
即可自动安装开发Theos环境，安装过程中会卡住请耐心等待15~20分钟后退出应用后台重新打开即可安装成功
### 验证安装
1.打开Newterm3输入
``` bash
 $THEOS/usr/bin/nic.pl
```
如显示(1) (2) (3)的选项即安装成功,反之filza打开越狱目录找到tmp文件夹,搜索sdks打开iPhoneossdks的目录,复制里面的sdk到越狱目录/Users/theos/sdks文件夹内

打赏一下李子呗
<img src="/img/wechat.png" width="50%">