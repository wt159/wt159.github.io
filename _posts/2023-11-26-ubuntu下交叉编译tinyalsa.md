---
title: ubuntu下交叉编译tinyalsa
tags: ubuntu tinyalsa
mermaid: true
---

## 环境

* Ubuntu 20.04.2 LTS

* tinyalsa-2.0.0

## 下载源码

[Github https://github.com/tinyalsa/tinyalsa](https://github.com/tinyalsa/tinyalsa/releases)

## 解压

```shell
tar -xvf tinyalsa-2.0.0.tar.bz2
```

## 编译源码

### 添加交叉编译配置

```shell
# vim toolchain_arm_linux_gcc.cmake
# 设置C和C++编译器路径
set(CMAKE_C_COMPILER "/usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf-gcc")
set(CMAKE_CXX_COMPILER "/usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf-g++")

# 设置编译选项
set(CMAKE_C_FLAGS "-Wall -Wextra -Werror -Wfatal-errors")
set(CMAKE_CXX_FLAGS "-Wall -Wextra -Werror -Wfatal-errors")
```

```shell
mkdir _install
mkdir arm_build
cd arm_build
cmake .. --no-warn-unused-cli -DCMAKE_BUILD_TYPE:STRING=Debug -DCMAKE_TOOLCHAIN_FILE=../toolchain_arm_linux_gcc.cmake -DCMAKE_INSTALL_PREFIX=$PWD/../_install
cmake --build .
cmake --install .
```

#### 报错

```shell
/home/cosmos/workspace/3rdparty/tinyalsa-2.0.0/src/pcm.c: In function ‘pcm_hw_mmap_status’:
/home/cosmos/workspace/3rdparty/tinyalsa-2.0.0/src/pcm.c:626:29: error: overflow in implicit constant conversion [-Werror=overflow]
                             SNDRV_PCM_MMAP_OFFSET_STATUS);
                             ^
compilation terminated due to -Wfatal-errors.

/home/cosmos/workspace/3rdparty/tinyalsa-2.0.0/src/pcm.c: In function ‘pcm_hw_mmap_status’:
/home/cosmos/workspace/3rdparty/tinyalsa-2.0.0/src/pcm.c:633:42: error: overflow in implicit constant conversion [-Werror=overflow]
                              MAP_SHARED, SNDRV_PCM_MMAP_OFFSET_CONTROL);
                                          ^
compilation terminated due to -Wfatal-errors.
```

解决办法：

```shell
# 添加一个强制转换
(off_t)SNDRV_PCM_MMAP_OFFSET_STATUS
(off_t)SNDRV_PCM_MMAP_OFFSET_CONTROL
```
