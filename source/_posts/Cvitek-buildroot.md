---
title: buildroot 打包 rootfs
top_img: transparent
date: 2023-11-16 23:45:42
updated: 2023-12-9 23:45:42
tags:
  - Linux
  - Buildroot
  - Rootfs
  - Cvitek
categories: Linux
keywords:
description:
---

## Cvitek buildroot 打包 rootfs 流程介绍

> 基于 [https://github.com/sophgo/cvi_mmf_sdk/tree/v4.1.0](https://github.com/sophgo/cvi_mmf_sdk/tree/v4.1.0) 的实现。

## 概述

在 [cvi_mmf_sdk](https://github.com/sophgo/cvi_mmf_sdk) 中，支持两种生成 rootfs 的方式：

- 基于 ramdisk 目录下的文件系统。
- 基于 buildroot 打包生成文件系统。

前者仅保留了常用的一些工具，因此生成的文件系统较小，不过新增工具比较麻烦。后者方便用户自己控制需要打包哪些工具。

**默认使用第一种方式来生成文件系统**，可通过在板子的配置文件中加上 `CONFIG_BUILDROOT_FS=y` （如 [cv1812cp_sophpi_duo_sd_defconfig#L30](https://github.com/sophgo/cvi_mmf_sdk/blob/v4.1.0/build/boards/cv181x/cv1812cp_sophpi_duo_sd/cv1812cp_sophpi_duo_sd_defconfig#L30) 中的例子），以启用 `buildroot` 来生成文件系统。

## rootfs 打包流程说明

在 `source cvisetup.sh` 之后，就可以使用 `pack_rootfs` 命令来打包文件系统了。它本质上是定义在 [common_functions.sh](https://github.com/sophgo/cvi_mmf_sdk/blob/v4.1.0/build/common_functions.sh#L107) 中的一个 shell 函数，在设置必要的环境变量后，通过 `make rootfs` 完成打包的相关处理。主要有3个步骤：

```makefile
rootfs-prepare:
	# 这里将所需要的打包文件都复制到 $(OUTPUT_DIR)/rootfs 这个目录中

rootfs-pack:rootfs-prepare
rootfs-pack:
	# 完成一些清理工作，去掉无用的文件或者 strip ko文件等
	# 制作文件系统

rootfs:rootfs-pack
rootfs:
	# 文件系统在 rootfs-pack 中已制作完成，
	# 这里仅对该文件再进行封装一个指定的 Header，烧录到 flash 时会检查该 Header。
	# 若无Header，或Header信息不正确，就会跳过该文件的烧录。Header本身并不会烧录到 flash。
	# 如果是 SD 卡启动的方式，则不需要封装。
ifneq ($(STORAGE_TYPE), sd)
	$(call raw2cimg ,rootfs.$(STORAGE_TYPE))
endif
```

上面是基于 `ramdisk` 目录制作文件系统的流程，基于 `buildroot` 的流程也是类似的：

```makefile
br-rootfs-prepare:
	# 将必要的文件复制到 buildroot 目录下的 overlay 中，
	# buildroot 制作文件系统时会将该目录下的文件也包含进去。

br-rootfs-pack:
	# 制作 ext4 的文件系统

# 在 Makefile 中，会检查 CONFIG_BUILDROOT_FS 配置是否启用
# 若启用了，则通过 br-rootfs-pack 调用 buildroot 来制作文件系统。
ifeq ($(CONFIG_BUILDROOT_FS),y)
rootfs:br-rootfs-prepare
rootfs:br-rootfs-pack
else
rootfs:rootfs-pack
rootfs:
	$(call print_target)
ifneq ($(STORAGE_TYPE), sd)
	$(call raw2cimg ,rootfs.$(STORAGE_TYPE))
endif
endif
```

有了基本流程的概念后，下面展开分析基于 `buildroot` 制作文件系统的流程。

### br-rootfs-prepare

`br-rootfs-prepare` 的主要工作是将需要打包到文件系统中的文件复制到指定的目录下，以便 buildroot 在制作文件系统时，能将这些文件一同打包。一般这些文件包括：

- `build_kernel` 得到的 `.ko` 文件；
- `build_middleware` 得到的动态库文件；
- 存放在 `middlleware` 中以 `ko` 或 `so` 形式发布的一些模块，如：[middleware/v2/cv180x/ko](https://github.com/sophgo/cvi_mmf_sdk/tree/v4.1.0/middleware/v2/cv180x/ko)。

在 `cvisetup.sh` 中将需要的路径导出为环境变量，主要有：

```bash
# buildroot 仓库所在路径
export BR_DIR="$TOP_DIR"/buildroot-2021.05

# overlay 路径，用于存储需要打包到文件系统中的文件
export BR_OVERLAY_DIR=${BR_DIR}/board/cvitek/${CHIP_ARCH}/overlay

# buildroot 的配置文件名称，即：
# cvitek_CV180X_musl_riscv64_defconfig  cvitek_CV181X_musl_riscv64_defconfig
export BR_BOARD=cvitek_${CHIP_ARCH}_${SDK_VER}
export BR_DEFCONFIG=${BR_BOARD}_defconfig

# buildroot 生成的文件系统的保存路径
export BR_ROOTFS_DIR="$OUTPUT_DIR"/tmp-rootfs
```

上述环境变量定义了文件复制的目标位置，在 `br-rootfs-prepare` 只进行复制即可！

```makefile
br-rootfs-prepare:
	# 复制 ko 以及相关 lib 到 buildroot 相关路径
	${Q}mkdir -p $(BR_OVERLAY_DIR)/mnt/system
	${Q}cp -arf ${SYSTEM_OUT_DIR}/* $(BR_OVERLAY_DIR)/mnt/system/
	# 对 ko 以及相关动态库 进行 strip，以缩小文件系统大小
	${Q}find $(BR_OVERLAY_DIR) -name "*.ko" -type f -printf 'striping %p\n' -exec $(CROSS_COMPILE_KERNEL)strip --strip-unneeded {} \;
	${Q}find $(BR_OVERLAY_DIR) -name "*.so*" -type f -printf 'striping %p\n' -exec $(CROSS_COMPILE_KERNEL)strip --strip-all {} \;
	${Q}find $(BR_OVERLAY_DIR) -executable -type f ! -name "*.sh" ! -path "*etc*" ! -path "*.ko" -printf 'striping %p\n' -exec $(CROSS_COMPILE_SDK)strip --strip-all {} 2>/dev/null \;
```

### br-rootfs-pack

`br-rootfs-pack` 的主要工作是指定 buildroot 的配置文件，完成 buildroot 的编译工作，以得到需要的文件系统。

```makefile
br-rootfs-pack:export TARGET_OUTPUT_DIR=$(BR_DIR)/output/$(BR_BOARD)
br-rootfs-pack:
	# 指定配置文件 和 编译工具链路径
	${Q}$(MAKE) -C $(BR_DIR) $(BR_DEFCONFIG) BR2_TOOLCHAIN_EXTERNAL_PATH=$(CROSS_COMPILE_PATH)
	# 编译得到文件系统，位于 $(TARGET_OUTPUT_DIR)/images/rootfs.ext4
	${Q}$(MAKE) -j${NPROC} -C $(BR_DIR)
	# ${Q}rm -rf $(BR_ROOTFS_DIR)/*
	# 将文件系统从 buildroot 路径，复制到 SDK 统一的输出路径 $(OUTPUT_DIR)
 	${Q}cp $(TARGET_OUTPUT_DIR)/images/rootfs.ext4 $(OUTPUT_DIR)/rawimages/rootfs_ext4.$(STORAGE_TYPE)
 	# 封装 Header，这是烧录到 flash 的必要步骤
	$(call raw2cimg ,rootfs_ext4.$(STORAGE_TYPE))
```

至此，就得到了 buildroot 编译的文件系统，不过需要注意的是：**该文件系统中仅包含了 `ko` 和 动态库，ramdisk 中的一些开机自动运行的脚本并未包含进文件系统中，因此没有自动加载 ko 等流程**

## buildroot 配置文件说明

目前 `buildroot/configs` 目录下提供了 `cv180x, cv181x` 的基础配置文件，用户可在该配置文件的基础上进行修改，部分参数与上面介绍的编译流程或编译工具链有关，不可随意更改。

```bash
# 1. 工具链相关的配置。
BR2_TOOLCHAIN_*
BR2_TOOLCHAIN_EXTERNAL_PREFIX="riscv64-unknown-linux-musl"
BR2_TARGET_LDFLAGS="-mcpu=c906fdv -march=rv64imafdcv0p7xthead -mcmodel=medany -mabi=lp64d"

# 2. overlay 路径
BR2_ROOTFS_OVERLAY="board/cvitek/CV181X/overlay"
```

### 网络问题

```bash
# 国内镜像
BR2_BACKUP_SITE="http://sources.buildroot.net"
BR2_KERNEL_MIRROR="https://mirrors.aliyun.com/linux-kernel"
BR2_GNU_MIRROR="https://mirrors.aliyun.com/gnu"
BR2_LUAROCKS_MIRROR="https://luarocks.cn"
BR2_CPAN_MIRROR="https://mirrors.aliyun.com/CPAN"
```

有些仓库设置上面的代理也没用😑，就需要**手动下载后，放到 `buildroot-2021.05/dl` 对应模块的文件夹下**。

### 打包 public 工具问题

在使用 ramdisk 方式打包文件系统时，有从好几个地方复制文件。一般来说，既然选择用 buildroot 了，一些三方工具都直接用 buildroot 编译的就好，不需要拷贝。**不过像 S11defer_init 这个文件，又必须从 ramdisk 拷贝。**

```makefile
# 从 ramdisk/rootfs/public 下拷贝选中的工具
define TARGET_PACKAGE_INSTALL_CMD
	@echo 'TARGET PACKAGE OUTPUT DIR=$(OUTPUT_DIR)/rootfs';\
	$(foreach t,$(TARGET_PACKAGES),\
		${Q}cd $(TOP_DIR)/ramdisk/rootfs/public/$(t)/$(packages_arch)/ && \
		${Q}find . \( ! -type d ! -name "*.a" ! -path "*include*" ! -name ".gitkeep" \) \
			-printf 'Copy Package file $(TOP_DIR)/ramdisk/rootfs/public/$(t)/$(packages_arch)/%p\n' \
			-exec cp -a --remove-destination --parents '{}' $(OUTPUT_DIR)/rootfs/ \; ; )
endef

rootfs-prepare:$(OUTPUT_DIR)/rootfs
	# Copy rootfs
ifeq ($(CONFIG_FASTBOOT),y)
	${Q}cp -a --remove-destination $(RAMDISK_PATH)/rootfs/$(ROOTFS_BASE)_shrink/* $(OUTPUT_DIR)/rootfs
else
	${Q}cp -a --remove-destination $(RAMDISK_PATH)/rootfs/$(ROOTFS_BASE)/* $(OUTPUT_DIR)/rootfs
endif

	# Copy arch overlay rootfs
ifneq ("$(wildcard $(SDK_VER_FOLDER_PATH))", "")
	${Q}cp -r $(SDK_VER_FOLDER_PATH)/* $(OUTPUT_DIR)/rootfs
endif

	# Copy chip overlay rootfs
ifneq ("$(wildcard $(CHIP_FOLDER_PATH))", "")
	${Q}cp -r $(CHIP_FOLDER_PATH)/* $(OUTPUT_DIR)/rootfs
endif

	# Copy project overlay rootfs
ifneq ("$(wildcard $(CUST_FOLDER_PATH))", "")
	${Q}cp -r $(CUST_FOLDER_PATH)/* $(OUTPUT_DIR)/rootfs
endif

	# 打包 menuconfig 中配置的一些三方工具。比如 ntp/dropbear/python3 等
	$(call TARGET_PACKAGE_INSTALL_CMD)
	# 设置其他
	${Q}${BUILD_PATH}/boards/default/rootfs_script/prepare_rootfs.sh $(OUTPUT_DIR)/rootfs
	# Generate S10_automount
	${Q}python3 $(COMMON_TOOLS_PATH)/image_tool/create_automount.py $(FLASH_PARTITION_XML) $(OUTPUT_DIR)/rootfs/etc/init.d/
	# Generate /etc/fw_env.config
	${Q}python3 $(COMMON_TOOLS_PATH)/image_tool/mkcvipart.py $(FLASH_PARTITION_XML) $(OUTPUT_DIR)/rootfs/etc/ --fw_env
ifneq ($(CONFIG_FASTBOOT),y)
	if [ -f $(ROOTFS_DIR)/etc/init.d/fastboot ]; then ${Q}rm -rf $(ROOTFS_DIR)/etc/init.d/fastboot ; fi ;
	if [ -f $(ROOTFS_DIR)/etc/init.d/S11defer_init ]; then ${Q}rm -rf $(ROOTFS_DIR)/etc/init.d/S11defer_init; fi ;
endif
```

上面从几个路径拷了文件，在使用 buildroot 时，可以选择只从 Project 路径拷贝。

```makefile
CUST_FOLDER_NAME = cv1813ha_wevb_0007a_emmc
CHIP_FOLDER_PATH = /data/song.yu/youdao2/v4.2.0-intl/ramdisk/rootfs/overlay/cv1813ha
SDK_VER_FOLDER_PATH = /data/song.yu/youdao2/v4.2.0-intl/ramdisk/rootfs/overlay/cv181x_32bit
CUST_FOLDER_PATH = /data/song.yu/youdao2/v4.2.0-intl/ramdisk/rootfs/overlay/cv1813ha_wevb_0007a_emmc
```

将必需的文件放在 `ramdisk/rootfs/overlay/cv1813ha_wevb_0007a_emmc` 目录下就行。

```makefile
br-rootfs-prepare:
	$(call print_target)

	# Copy project overlay rootfs, like ramdisk/rootfs/overlay/cv1813ha_wevb_0007a_emmc
ifneq ("$(wildcard $(CUST_FOLDER_PATH))", "")
	${Q}cp -r $(CUST_FOLDER_PATH)/* $(BR_OVERLAY_DIR)
endif

	# Generate /etc/inid.d/S10automount
	${Q}python3 $(COMMON_TOOLS_PATH)/image_tool/create_automount.py $(FLASH_PARTITION_XML) $(BR_OVERLAY_DIR)/etc/init.d/
	# Generate /etc/fw_env.config
	${Q}python3 $(COMMON_TOOLS_PATH)/image_tool/mkcvipart.py $(FLASH_PARTITION_XML) $(BR_OVERLAY_DIR)/etc/ --fw_env

ifneq ($(CONFIG_FASTBOOT),y)
	if [ -f $(ROOTFS_DIR)/etc/init.d/fastboot ]; then ${Q}rm -rf $(ROOTFS_DIR)/etc/init.d/fastboot ; fi ;
	if [ -f $(ROOTFS_DIR)/etc/init.d/S11defer_init ]; then ${Q}rm -rf $(ROOTFS_DIR)/etc/init.d/S11defer_init; fi ;
endif

	# copy ko and mmf libs
	${Q}mkdir -p $(BR_OVERLAY_DIR)/mnt/system
	${Q}cp -arf ${SYSTEM_OUT_DIR}/* $(BR_OVERLAY_DIR)/mnt/system/
	# strip
	${Q}find $(BR_OVERLAY_DIR) -name "*.ko" -type f -printf 'striping %p\n' -exec $(CROSS_COMPILE_KERNEL)strip --strip-unneeded {} \;
	${Q}find $(BR_OVERLAY_DIR) -name "*.so*" -type f -printf 'striping %p\n' -exec $(CROSS_COMPILE_KERNEL)strip --strip-all {} \;
	${Q}find $(BR_OVERLAY_DIR) -executable -type f ! -name "*.sh" ! -path "*etc*" ! -path "*.ko" -printf 'striping %p\n' -exec $(CROSS_COMPILE_SDK)strip --strip-all {} 2>/dev/null \;
```

如果非要 rootfs/public 下的工具，也在加上这个，不建议！还是直接修改 buildroot 配置更好。

```makefile
define TARGET_PACKAGE_INSTALL_CMD_BR
	@echo 'TARGET PACKAGE OUTPUT DIR=$(BR_OVERLAY_DIR)';\
	$(foreach t,$(TARGET_PACKAGES),\
		${Q}cd $(TOP_DIR)/ramdisk/rootfs/public/$(t)/$(packages_arch)/ && \
		${Q}find . \( ! -type d ! -name "*.a" ! -path "*include*" ! -name ".gitkeep" \) \
			-printf 'Copy Package file $(TOP_DIR)/ramdisk/rootfs/public/$(t)/$(packages_arch)/%p\n' \
			-exec cp -a --remove-destination --parents '{}' $(BR_OVERLAY_DIR)/ \; ; )
endef
```

### 其他

- ❗❗注意：buildroot 目录下的 output/xxx 中的文件，并不会自动删除。也就是说，overlay中第一次添加了某个文件，后面不需要这个文件了，仅从overlay目录删除还不够，必须从 output/xxx/target 中删除，而且有时候奇奇怪怪的编译报错，直接清空 output/xxx 文件夹也能解决。

  ❗也不能简单地直接 **只** 删除output/xxx/target  中的文件，有一些文件、文件夹是在 buildroot 某个模块的编译过程中放进去的，只删除目标文件，而不清楚编译中间文件 output/xxx/build，那些文件也不会重新生成。

  如果在流程中自动清空 output/xxx 文件夹，会导致编译时间变得很长。

- 如何去掉开机必须输入用户名、密码？
  ```
  修改 getty 服务
  如果你使用的是 getty 来管理终端登录，可以在 /etc/inittab 文件中设置自动登录。
  
  编辑 /etc/inittab：
  打开 /etc/inittab 文件（如果存在），找到类似于以下的行：
  
  1:2345:respawn:/sbin/getty 38400 tty1
  修改为：
  
  1:2345:respawn:/sbin/getty --noclear tty1
  创建自动登录的脚本：
  在 /etc/init.d/ 下创建一个脚本，命名为 autologin，内容如下：
  
  #!/bin/sh
  exec /bin/login -f your_username
  替换 your_username 为你的实际用户名。
  
  使脚本可执行：
  chmod +x /etc/init.d/autologin
  ```

- 不显示当前所在路径，增加 `/etc/profile` 文件。`\u: user \h: host`

  ```bash
  #/etc/profile
  export LD_LIBRARY_PATH="/lib:/lib/3rd:/usr/lib:/mnt/system/lib:/mnt/system/usr/lib:/mnt/system/usr/lib/3rd:/mnt/data/lib"
  export PATH="/usr/local/bin:/usr/bin:/bin:/usr/local/sbin:/usr/sbin:/sbin:/mnt/system/usr/bin:/mnt/system/usr/sbin:/mnt/data/bin:/mnt/data/sbin"

  if [ "$PS1" ]; then
  	if [ "`id -u`" -eq 0 ]; then
  		export PS1='# '
  	else
  		export PS1='$ '
  	fi
  fi

  export PAGER='/bin/more '
  export EDITOR='/bin/vi'

  # Source configuration files from /etc/profile.d
  for i in /etc/profile.d/*.sh ; do
  	if [ -r "$i" ]; then
  		. $i
  	fi
  	unset i
  done

  export HOSTNAME="$(hostname)"
  export OLDPWD=/root

  # \u: user \h: host
  if [ '$USER' == 'root' ]; then
      export PS1='[\u@\h]\w\# '
  else
      export PS1='[\u@\h]\w\$ '
  fi

  alias ll='ls -alF'
  alias la='ls -A'
  alias l='ls -CF'
  ```

  修改之后，就显示 `[root@cvitek]~# `

- 使用 erofs 只读文件系统后，/var/run 中也只读，导致一些动态生成的文件（如 PID 文件、socket 文件等）无法写入。增加 `/etc/fstab 文件。
  ```
  ramdisk/rootfs/common_arm/etc$ cat fstab
  # <file system> <mount pt>      <type>  <options>       <dump>  <pass>
  /dev/root       /               ext2    rw,noauto       0       1
  proc            /proc           proc    defaults        0       0
  devpts          /dev/pts        devpts  defaults,gid=5,mode=620,ptmxmode=0666   0       0
  tmpfs           /dev/shm        tmpfs   mode=0777       0       0
  tmpfs           /tmp            tmpfs   mode=1777       0       0
  tmpfs           /run            tmpfs   mode=0755,nosuid,nodev  0       0
  tmpfs           /var/run        tmpfs   mode=0755,nosuid,nodev  0       0
  tmpfs           /var/lock       tmpfs   mode=0755,nosuid,nodev  0       0
  tmpfs           /var/empty      tmpfs   mode=0755,nosuid,nodev  0       0
  sysfs           /sys            sysfs   defaults        0       0
  nodev           /sys/kernel/debug debugfs   defaults    0       0
  ```
