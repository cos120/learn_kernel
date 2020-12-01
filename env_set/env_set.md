# Debug env 搭建

## 环境构建

### 基本环境

我使用的 ubuntu 18.04 版本，目前需要的基础环境为

```bash
apt install apt install build-essential kernel-package fakeroot libncurses5-dev libssl-dev ccache flex bison libelf-dev qemu qemu-system git
```

### kernel 

使用的 kernel upstream 为

```
git clone https://kernel.googlesource.com/pub/scm/linux/kernel/git/next/linux-next.git
```

PS： 在此最好记录一下自己使用的 kernel 版本， 之后会继续用到，这里我用 `v5.4` 这个版本。

之后编译 kernel

```bash
make ARCH=x86_64 x86_64_defconfig
make ARCH=x86_64 menuconfig
```

既然是学习 kernel， 那肯定需要编译带 debug 的 kernel，在 menu 里面选择


在 kernel hacking ->  Compile-time checks and compiler options
选择 Compile the kernel with debug info & Provide GDB scripts for kernel debugging

至此就可以进行编译了。

### buildroot

```bash
git clone git://git.buildroot.net/buildroot
make menuconfig
```

在 Target Options -> Target Architecture 中选择 x86_64 架构
同时在 Filesystem images -> ext2/3/4 root file system 中选择 ext4
将 kernel 的 upstream 与上面步骤的地址设置成一样的，并且分支也要一样

因为我这次针对 VFS 进行学习，在此用 [黄老师](https://github.com/jserv/simplefs) 的 simplefs 进行实验，因此需要额外配置其他的 ko。

配置共分为以下步骤

首先生成 simplefs 的 menuconfig，注意 `BR2_PACKAGE_` 这个前缀是必须的。

```bash
cd buildroot
mkdir package/simplefs
cat << EOF > package/simplefs/Config.in
config BR2_PACKAGE_SIMPLEFS
       bool "simpefs"
       help
               Simplefs ko.
EOF
```

之后编写 `package/simplefs/simplefs.mk`

`SIMPLEFS_SITE` 为外部依赖的项目路径， 注意填写 `SIMPLEFS_SITE` 时 cwd 为 buildroot，其中，我的目录结构如下，因此为 `../simplefs`

```text
.
├── [Dec  1 20:52]  buildroot
├── [Dec  1 20:46]  linux-next
└── [Dec  1 20:10]  simplefs                                                               
```
```shell
cat << EOF > package/simplefs/simplefs.mk
SIMPLEFS_VERSION = 1.0
SIMPLEFS_SITE = ../simplefs
SIMPLEFS_SITE_METHOD = local
define KERNEL_MODULE_BUILD_CMDS
        $(MAKE) -C '$(@D)' LINUX_DIR='$(LINUX_DIR)' CC='$(TARGET_CC)' LD='$(TARGET_LD)' modules
endef 
$(eval $(kernel-module))
$(eval $(generic-package))
EOF
```


对 simplefs 里面的修改如下

```git
diff --git a/Makefile b/Makefile
index 3598587..43b3e42 100644
--- a/Makefile
+++ b/Makefile
@@ -1,12 +1,13 @@
 obj-m += simplefs.o
+ccflags-y := -DDEBUG -g -std=gnu99 -Wno-declaration-after-statement
 simplefs-objs := fs.o super.o inode.o file.o dir.o extent.o
-KDIR ?= /lib/modules/$(shell uname -r)/build
 MKFS = mkfs.simplefs
 all: $(MKFS)
-       make -C $(KDIR) M=$(PWD) modules
+       $(MAKE) -C '$(LINUX_DIR)' M='$(PWD)' modules
+
 IMAGE ?= test.img
 IMAGESIZE ?= 50
```

运行 make menuconfig， 如果没有任何错误的话，在 `Target packages`最下面会出现 simplefs 选项，开启它

![](https://raw.githubusercontent.com/cos120/learn_kernel/main/env_set/simplefs_menu.png)

最后运行编译，时间会很长

```shell
make -j $(($(nproc) - 1))
```

## GDB 配置

TBD
