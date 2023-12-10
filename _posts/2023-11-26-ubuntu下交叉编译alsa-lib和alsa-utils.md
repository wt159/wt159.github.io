---
title: ubuntu下交叉编译alsa-lib和alsa-utils
tags: ubuntu alsa-lib alsa-utils
mermaid: true
---

## 环境

* Ubuntu 20.04.2 LTS

* alsa-lib-1.2.2

* alsa-utils-1.2.2

## 下载源码

[官网https://www.alsa-project.org](https://www.alsa-project.org/main/index.php/Download)

## 解压

```shell
tar -xvf alsa-lib-1.2.2.tar.bz2
tar -xvf alsa-utils-1.2.2.tar.bz2
```

## 安装依赖

```shell
sudo apt install -y build-essential gcc g++ cmake libxkbcommon-x11-dev libgl1-mesa-dev libglu1-mesa-dev libfontconfig1-dev libmysqlclient-dev libxcb-xfixes0-dev libxcb-util-dev
```

## 编译源码

### alsa-lib

```shell
cd alsa-lib-1.2.2
mkdir _install
sudo mkdir /usr/share/arm-alsa -p
./configure --host=arm-linux-gnueabihf --enable-shared --disable-python --prefix=$PWD/_install --with-configdir=/usr/share/arm-alsa
make
sudo make install
```

#### 详解

```shell
--host=arm-linux-gnueabihf #指定主机
# 如果arm-linux-gnueabihf编译器没有添加到全局变量，则需要指定路径
CC=/usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf-gcc
STRIP=/usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf-strip
--enable-shared     #使能动态库
--disable-python    #失能python
--prefix=/path      #动态库安装目录
--with-configdir=/usr/share/arm-alsa #配置文件目录(程序运行绝对路径)
```

#### 报错

```shell
make[2]: Entering directory '/home/cosmos/workspace/3rdparty/alsa-lib-1.2.2/src/topology'
 /usr/bin/mkdir -p '/home/cosmos/workspace/imx6ull_alientek/nfs/rootfs/usr/lib'
 /bin/bash ../../libtool   --mode=install /usr/bin/install -c   libatopology.la '/home/cosmos/workspace/imx6ull_alientek/nfs/rootfs/usr/lib'
libtool: warning: relinking 'libatopology.la'
libtool: install: (cd /home/cosmos/workspace/3rdparty/alsa-lib-1.2.2/src/topology; /bin/bash "/home/cosmos/workspace/3rdparty/alsa-lib-1.2.2/libtool"  --silent --tag CC --mode=relink arm-linux-gnueabihf-gcc -g -O2 -version-info 2:0:0 -Wl,--version-script=../Versions -Wl,-z,defs -o libatopology.la -rpath /home/cosmos/workspace/imx6ull_alientek/nfs/rootfs/usr/lib parser.lo builder.lo ctl.lo dapm.lo pcm.lo data.lo text.lo channel.lo ops.lo elem.lo save.lo decoder.lo log.lo ../libasound.la )
/home/cosmos/workspace/3rdparty/alsa-lib-1.2.2/libtool: line 10533: arm-linux-gnueabihf-gcc: command not found
libtool:   error: error: relink 'libatopology.la' with the above command before installing it
make[2]: *** [Makefile:408: install-libLTLIBRARIES] Error 1
make[2]: Leaving directory '/home/cosmos/workspace/3rdparty/alsa-lib-1.2.2/src/topology'
make[1]: *** [Makefile:595: install-am] Error 2
make[1]: Leaving directory '/home/cosmos/workspace/3rdparty/alsa-lib-1.2.2/src/topology'
make: *** [Makefile:404: install-recursive] Error 1
```

解决办法：

```shell
# 切换到管理员账号
sudo -s
source /etc/profile
```

#### 安装

```shell
sudo cp -rf /usr/share/arm-alsa xxx/nfs/rootfs/usr/share/
cp _install/bin/* xxx/nfs/rootfs/usr/bin
cp _install/lib/*so* xxx/nfs/rootfs/usr/lib/
```

### alsa-utils

```shell
cd alsa-utils-1.2.2
mkdir _install
./configure --host=arm-linux-gnueabihf --prefix=$PWD/_install --with-alsa-inc-prefix=$PWD/../alsa-lib-1.2.2/_install/include --with-alsa-prefix=$PWD/../alsa-lib-1.2.2/_install/lib --disable-alsamixer --disable-xmlto
make
sudo make install
```

#### 报错

```shell
mv: cannot stat 't-ja.gmo': No such file or directory
make[2]: *** [Makefile:41: ru.gmo] Error 1

mv: cannot stat 't-ru.gmo': No such file or directory
make[2]: *** [Makefile:41: ru.gmo] Error 1
```

解决办法：

```shell
touch alsaconf/po/t-ja.gmo
touch alsaconf/po/t-ru.gmo
```

#### 安装

```shell
cp _install/bin/*  xxx/nfs/rootfs/usr/bin/
cp _install/sbin/* xxx/nfs/rootfs/usr/sbin/
```
