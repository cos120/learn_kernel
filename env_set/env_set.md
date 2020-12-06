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

```text
kernel hacking ->  Compile-time checks and compiler options -> Compile the kernel with debug info
kernel hacking ->  Compile-time checks and compiler options -> Provide GDB scripts for kernel debugging
```

至此就可以进行编译了，当 kernel 编译完成后， 会用到如下两个文件

```text
root of kernel
├──vmlinux 
└──arch/x86_64/boot/bzImage
```

接下来我们就可以启动 kernel 了

```bash
emu-system-x86_64 -s -S -no-kvm -kernel arch/x86/boot/bzImage -hda /dev/zero -append "root=/dev/zero console=ttyS0 nokaslr" -serial stdio -display none
```

`-s -S` 代表进入 debug 模式， 并且等待 gdb attach 到 qemu， 默认 qemu debug 端口在 1234

在另一个终端中运行 `gdb vmlinux`

```bash
(gdb) target remote :1234
Remote debugging using :1234
0x000000000000fff0 in entry_stack_storage ()
(gdb) b start_kernel
Breakpoint 1 at 0xffffffff82652b51: file init/main.c, line 576.
(gdb) c
Continuing.

Breakpoint 1, start_kernel () at init/main.c:576
576     {
(gdb) bt
#0  start_kernel () at init/main.c:576
#1  0xffffffff810000e6 in secondary_startup_64 () at arch/x86/kernel/head_64.S:241
#2  0x0000000000000000 in ?? ()
(gdb) l
571     {
572             rest_init();
573     }
574
575     asmlinkage __visible void __init start_kernel(void)
576     {
577             char *command_line;
578             char *after_dashes;
579
580             set_task_stack_end_magic(&init_task);
(gdb)
```

至此可以完成对 kernel 启动时的断点调试，但由于没有文件系统挂载，kernel 启动完就会 panic。

### buildroot

buildroot 是一个创建 rootfs 的工具，可以打包全部的 kernel 并构建 rootfs。

```bash
git clone git://git.buildroot.net/buildroot
make menuconfig
```

```text
Target Options -> Target Architecture -> X86_64
Filesystem images -> ext2/3/4 root file system -> ext4
```

以上步骤将配置目标环境为 x86 架构，文件系统为 ext4。

之后运行编译即可，编译生成的文件在 `buildroot/outputs/images/rootfs.ext4`

运行 qemu

```bash
qemu-system-x86_64 -kernel path-to-kernel/arch/x86/boot/bzImage \
-boot c -m 2049M -hda path-to-buildroot/output/images/rootfs.ext4 \
-append "root=/dev/sda rw console=ttyS0,115200 acpi=off nokaslr" \
-serial stdio -display none
```

如果没有问题，会看到很多 boot 启动的日志，进入登录终端

```text
.....
Welcome to Buildroot
buildroot login:
```

用户 root，没有密码，就可以登入 kernel 了，此时在 gdb 中可以做各种断点调试。

`但由于 buildroot 不能使用系统自带的 toolchain，因此用该种方式编译的 rootfs 可能导致编译的一些功能失效。`