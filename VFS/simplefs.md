## simplefs debug 环境

因为我这次针对 VFS 进行学习，在此用 [黄老师](https://github.com/jserv/simplefs) 的 simplefs 进行实验，因此需要额外配置其他的 ko。

目前已经配置好了基本的 kernel debug 环境，但由于 buildroot 的 toolchain 与系统不能一致，因此使用 buildroot 来统一构建 kernel + rootfs

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
└── [Dec  1 20:10]  simplefs                                                               
```
```shell
cat << EOF > package/simplefs/simplefs.mk
SIMPLEFS_VERSION = 1.0                                                                                                                 
SIMPLEFS_SITE = ../simplefs
SIMPLEFS_SITE_METHOD = local
define SIMPLEFS_MODULE_BUILD_CMDS
        $(MAKE) all -C '$(@D)' LINUX_DIR='$(LINUX_DIR)' CC='$(TARGET_CC)' LD='$(TARGET_LD)' modules
endef

$(eval $(kernel-module))

define MKFS_BUILD_CMDS
       $(MAKE) mkfs.simplefs CC="$(TARGET_CC)" LD="$(TARGET_LD)" -C $(@D)
endef

define MKFS_INSTALL_TARGET_CMDS
                $(INSTALL) -D -m 0755 $(@D)/mkfs.simplefs $(TARGET_DIR)/usr/bin
endef
SIMPLEFS_POST_BUILD_HOOKS += MKFS_BUILD_CMDS
SIMPLEFS_POST_BUILD_HOOKS += MKFS_INSTALL_TARGET_CMDS

$(eval $(generic-package))
EOF
```


对 simplefs 里面的修改如下，将 KDIR 改为 LINUX_DIR，buildroot 在编译时会进行替换

```git
diff --git a/Makefile b/Makefile
index 3598587..10d7098 100644
--- a/Makefile
+++ b/Makefile
@@ -1,12 +1,12 @@
 obj-m += simplefs.o
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

除此之外， 还需要一些额外的选项，主要用于开启 debug symbol 和对 loopback device 的支持
buildroot 的选项
```bash
make menuconfig

Kernrl -> Deconfig_name(设置为 x86_64)
Build Options -> build package with debug symbol
Build Options -> strip target binaries (去掉)
Build Options -> gcc optimization level (optimize for debugging)
File System -> cloop root filesystem for the target device
File System -> exact size(设置大小到 200M)
Target packages -> Filesystem and flash utilities -> e2fsprogs
```
linux kernel 的选项，除了开启 kernel debug，还需要开启 loop device 的支持
```bash
make linux-menuconfig
Device Drivers -> Block devices ->  Loopback device support
```

最后运行编译，时间会很长

```shell
make -j $(($(nproc) - 1))
```

去 buildroot 的根目录运行 qemu

```bash
qemu-system-x86_64 -s -kernel output/images/bzImage \
-boot c -m 2049M -hda output/images/rootfs.ext4 \
-append "root=/dev/sda rw console=ttyS0,115200 acpi=off nokaslr" \
-serial stdio -display none
```

登录后先测试一下 simplefs

```bash
# modprobe simplefs
[   31.341419] simplefs: loading out-of-tree module taints kernel.
[   31.361416] simplefs: module loaded
# dd if=/dev/zero of=test.img count=50 bs=1M
50+0 records in
50+0 records out
# mkfs.simplefs test.img
Superblock: (4096)
        magic=0xdeadce
        nr_blocks=12800
        nr_inodes=12824 (istore=229 blocks)
        nr_ifree_blocks=1
        nr_bfree_blocks=1
        nr_free_inodes=12823
        nr_free_blocks=12567
Inode store: wrote 229 blocks
        inode size = 72 B
Ifree blocks: wrote 1 blocks
Bfree blocks: wrote 1 blocks                                                                                                           
# mkdir test
# mount -t simplefs test.img test
[   81.642772] simplefs: '/dev/loop0' mount success
# echo hello > test/a
# cat test/a
hello 
```

至此我们已经可以在 kernel 里面加载 simplefs 的 module，并且可以成功 mount 上，下一步就是进行 debug 调试了

## GDB 配置断点

buildroot 产出的 kernel vmlinux 在 `buildroot/output/build/linux-5.9.11` 下，其中 5.9.11 为你编译时的 kernel 版本。按照之前的步骤 attach 到 qemu 的进程，注意这个一但 attach 后 qemu 内运行的内核会卡住，需要在 gdb 中让 kernel 继续运行

此时， gdb 并没有加载 simplefs 的 symbol，gdb 提供的 python 脚本我没有成功，这里需要手动加载一下，首先去 qemu

```bash
# cat /sys/module/simplefs/sections/.text
0xffffffffc0000000
```

代表内核里 simplefs 的 text 段的地址为 `0xffffffffc0000000`，回到 gdb 中，先用 ctrl-C 中断 gdb，加载 simplefs 的地址，之后 gdb 的断点命令可以补全 simplefs 相关的函数，这里我们断在 `simplefs_new_inode` 函数，因此任何创建命令都会断下

```bash
^C
Program received signal SIGINT, Interrupt.
default_idle () at arch/x86/kernel/process.c:688
688             safe_halt();
(gdb) add-symbol-file /root/buildroot/output/target/lib/modules/5.9.11/extra/simplefs.ko 0xffffffffc0000000
add symbol table from file "/root/buildroot/output/target/lib/modules/5.9.11/extra/simplefs.ko" at
        .text_addr = 0xffffffffc0000000
(y or n) y
Reading symbols from /root/buildroot/output/target/lib/modules/5.9.11/extra/simplefs.ko...done.
(gdb)
(gdb) b simplefs
simplefs.h                    simplefs_fill_super.cold      simplefs_mkdir                simplefs_symlink
simplefs.mod.c                simplefs_get_link             simplefs_mount                simplefs_symlink.cold
simplefs_alloc_inode          simplefs_iget                 simplefs_mount.cold           simplefs_sync_fs
simplefs_create               simplefs_init                 simplefs_new_inode            simplefs_unlink
simplefs_destroy_inode        simplefs_init_inode_cache     simplefs_new_inode.cold       simplefs_write_begin
simplefs_destroy_inode_cache  simplefs_iterate              simplefs_put_super            simplefs_write_begin.cold
simplefs_exit                 simplefs_kill_sb              simplefs_readpage             simplefs_write_end
simplefs_ext_search           simplefs_link                 simplefs_rename               simplefs_write_end.cold
simplefs_file_get_block       simplefs_link.cold            simplefs_rmdir                simplefs_write_inode
simplefs_fill_super           simplefs_lookup               simplefs_statfs               simplefs_writepage
(gdb) b simplefs_new_inode
Breakpoint 1 at 0x91: simplefs_new_inode. (2 locations)
(gdb) c
Continuing. 
```

回到 qemu 中，可以观察到进程卡了

```bash
# echo hello > test/b
```

回到 gdb， 可以观察到断点，之后可以进行调试了，但是 kernel 默认是 O2 优化，一些常量会被优化掉，但是不影响观察内核运行

```bash
Breakpoint 1, simplefs_new_inode () at /root/buildroot/output/build/simplefs-1.0/./inode.c:152
152         if (!S_ISDIR(mode) && !S_ISREG(mode) && !S_ISLNK(mode)) {
(gdb) bt
#0  simplefs_new_inode () at /root/buildroot/output/build/simplefs-1.0/./inode.c:152
#1  0xffffffffc0001779 in simplefs_create (dir=0xffff88807c53c028, dentry=0xffff8880790190c0, mode=<optimized out>,
    excl=<optimized out>) at /root/buildroot/output/build/simplefs-1.0/./inode.c:261
#2  0xffffffff811cf5c5 in lookup_open (got_write=<optimized out>, op=<optimized out>, file=<optimized out>, nd=<optimized out>)
    at fs/namei.c:3103
#3  open_last_lookups (op=<optimized out>, file=<optimized out>, nd=<optimized out>) at fs/namei.c:3177
#4  path_openat (nd=0xffffc90000223de0, op=0xffffc90000223eec, flags=<optimized out>) at fs/namei.c:3365
#5  0xffffffff811d10d9 in do_filp_open (dfd=<optimized out>, pathname=<optimized out>, op=0xffffc90000223eec) at fs/namei.c:3395
#6  0xffffffff811bb664 in do_sys_openat2 (dfd=-100, filename=<optimized out>, how=0xffffc90000223f28) at fs/open.c:1168
#7  0xffffffff811bc8df in do_sys_open (dfd=<optimized out>, filename=<optimized out>, flags=<optimized out>, mode=<optimized out>)
    at fs/open.c:1184
#8  0xffffffff81aa97d3 in do_syscall_64 (nr=<optimized out>, regs=0xffffc90000223f58) at arch/x86/entry/common.c:46
#9  0xffffffff81c0007c in entry_SYSCALL_64 () at arch/x86/entry/entry_64.S:118
#10 0x000000000131d420 in ?? ()
#11 0x0000000000000001 in fixed_percpu_data ()
#12 0x000000000131d458 in ?? ()
#13 0x0000000001320710 in ?? ()
#14 0x0000000000000001 in fixed_percpu_data ()
#15 0x000000000131d398 in ?? ()
#16 0x0000000000000246 in ?? ()
#17 0x0000000000000001 in fixed_percpu_data ()
#18 0x0000000000000000 in ?? ()
(gdb) l
147         struct simplefs_sb_info *sbi;
148         uint32_t ino, bno;
149         int ret;
150
151         /* Check mode before doing anything to avoid undoing everything */
152         if (!S_ISDIR(mode) && !S_ISREG(mode) && !S_ISLNK(mode)) {
153             pr_err(
154                 "File type not supported (only directory, regular file and symlink "
155                 "supported)\n");
156             return ERR_PTR(-EINVAL);
(gdb) 

```

