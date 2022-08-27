---
title: qemu搭建Linux学习环境
tags: tools linux Linux内核 qemu
---

# 背景
最近在学习Linux的内核驱动，就想在Linux环境下搭建一个运行Linux的环境。了解到qemu可以满足我的需求，就折腾了一下。
[参考1](https://blog.csdn.net/weixin_38227420/article/details/88402738?spm=1001.2101.3001.6650.2&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-2-88402738-blog-53783221.t0_layer_searchtargeting_sa&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-2-88402738-blog-53783221.t0_layer_searchtargeting_sa&utm_relevant_index=3)
[参考2](https://zhuanlan.zhihu.com/p/340362172)

# 环境
* VM虚拟机
* Ubuntu 20.04

# 安装

## 相关工具链
命令行终端下输入：
```shell
sudo apt-get update
sudo apt-get install -y git gnupg flex bison gperf build-essential \
zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 \
lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z1-dev ccache \
libgl1-mesa-dev libxml2-utils xsltproc unzip u-boot-tools qemu
```
## arm交叉编译工具
### 方法一(对版本没有要求)
使用apt直接安装
```shell
sudo apt-get install gcc-arm-linux-gnueabi -y
```
### 方法二(自己选择版本)
1. 下载编译链，执行命令
```shell
wget https://releases.linaro.org/components/toolchain/binaries/latest-7/arm-linux-gnueabihf/gcc-linaro-7.4.1-2019.02-x86_64_arm-linux-gnueabihf.tar.xz
```
关于编译链的资料参考：
[arm交叉编译器gnueabi、none-eabi、arm-eabi、gnueabihf的区别](https://www.cnblogs.com/linuxbo/p/4297680.html)

2。 解压源码
```shell
tar -xjf gcc-linaro-7.4.1-2019.02-x86_64_arm-linux-gnueabihf.tar.xz
```
3. 添加环境变量，使你的编译链全局可用
```shell
gedit /etc/profile
```
在最后一行加入：
```shell
export PATH=$PATH:/qemu-lab/gcc-linaro-7.4.1-2019.02-x86_64_arm-linux-gnueabihf/bin
```
该路径是你编译链加压后的编译工具所在的路径

4. 保存退出后使该变量生效
```shell
source /etc/profile
```
## 下载，编译Linux内核
1. 下载
[Linux内核下载地址](https://mirrors.edge.kernel.org/pub/linux/kernel/)
```shell
wget https://mirrors.tuna.tsinghua.edu.cn/kernel/v5.x/linux-5.10.99.tar.xz
```