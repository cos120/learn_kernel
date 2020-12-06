# learn_kernel

## 写在前面

学习 linux kernel 实在是太难了，这个系统太大了，而且 kernel 的迭代速度又很快， 因此打算用 qemu debug + custom ko 的方式来学习，在此记录一下学习过程。

## Debug 环境配置

本人学习过程均使用 qemu 模拟内核的方式进行学习, 因此第一步便是 [Debug Env](env_set/env_set.md) 的配置.

## VFS 学习

在配置好 Debug 环境后， 因为我这次针对 VFS 进行学习，在此用 [黄老师](https://github.com/jserv/simplefs) 的 simplefs 进行实验，通过加载一个文件系统的 module，来学习如何实现一个文件系统。所有的记录在 [VFS](VFS/simplefs.md)