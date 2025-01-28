---
title: theos无根插件适配
tags: iOS玩机
categories: iOS社区
date: 2025-01-25 21:52:56
top:
---
为了适配 Rootless 模式的 Theos 插件，你需要在`Makefile`中进行一些调整，以确保插件能够在无根越狱环境中正常运行。以下是适配 Rootless 模式的`Makefile`示例代码和说明：


适配 Rootless 模式的`Makefile`示例

```makefile
## 设置 Rootless 模式
ROOTLESS = 1
ifeq ($(ROOTLESS),1)
THEOS_PACKAGE_SCHEME = rootless
endif

## 根据是否启用 Rootless 模式设置目标系统版本
ifeq ($(THEOS_PACKAGE_SCHEME), rootless)
TARGET = iphone:clang:latest:15.0
else
TARGET = iphone:clang:latest:12.0
endif

## 包含 Theos 的通用配置
include $(THEOS)/makefiles/common.mk

## 定义插件名称和文件
TWEAK_NAME = YourTweakName
YourTweakName_FILES = Tweak.xm
YourTweakName_CFLAGS = -fobjc-arc
YourTweakName_LIBRARIES = substrate
YourTweakName_FRAMEWORKS = UIKit

## 设置安装路径
YourTweakName_INSTALL_PATH = /var/jb/usr/lib

## 包含 Theos 的插件构建规则
include $(THEOS_MAKE_PATH)/tweak.mk
```



代码说明

1. Rootless 模式判断：

• 通过设置`ROOTLESS`变量来启用 Rootless 模式。

• 如果启用 Rootless 模式，则设置`THEOS_PACKAGE_SCHEME = rootless`。


2. 目标系统版本：

• 根据是否启用 Rootless 模式，设置目标系统版本。例如，Rootless 模式下目标版本为 iOS 15.0。


3. 安装路径：

• 在 Rootless 模式下，插件的安装路径通常为`/var/jb/usr/lib`。


4. 其他配置：

• 包含 Theos 的通用配置文件和插件构建规则。


注意事项

• 路径适配：如果插件中涉及路径调用，确保路径适配 Rootless 环境。例如，`/var/jb/`是 Rootless 环境下的常用路径。

• 依赖库适配：确保依赖库路径正确，例如使用`@rpath`来链接动态库。

• 测试：在 Rootless 环境下测试插件，确保其功能正常。

通过以上配置，你的插件应该能够在 Rootless 模式下正常编译和运行。如果仍有问题，可以参考具体的错误日志进行分析。