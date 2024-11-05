---
title: ubuntu下使用ndk交叉编译
tags: android ndk cmake
---

## 概述

在**Ubuntu**下使用**NDK**+**cmake**交叉编译.

官方文档: [https://developer.android.google.cn/ndk/guides/cmake?hl=zh-cn](https://developer.android.google.cn/ndk/guides/cmake?hl=zh-cn)

## 下载NDK

下载地址: [https://developer.android.google.cn/ndk/downloads/](https://developer.android.google.cn/ndk/downloads/)
旧版本下载地址: [https://github.com/android/ndk/wiki/Unsupported-Downloads](https://github.com/android/ndk/wiki/Unsupported-Downloads)

## 安装NDK

```shell
unzip android-ndk-r21e-linux-x86_64.zip
```

## 配置环境变量

```shell
export ANDROID_NDK_HOME=/home/xxx/tools/android-ndk-r21e
export PATH=$ANDROID_NDK_HOME:$PATH
```

## 配置cmake

```shell
sudo apt-get install cmake
```

## 编译

```shell
# arm64-v8a 
# armeabi-v7a
# x86
# x86_64
mkdir build
cd build
cmake -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK_HOME/build/cmake/android.toolchain.cmake -DCMAKE_BUILD_TYPE=Debug -DANDROID_NDK=$ANDROID_NDK_HOME -DANDROID_ABI=armeabi-v7a -DANDROID_TOOLCHAIN=clang -DANDROID_PLATFORM=android-21 -DANDROID_STL=c++_shared ..
make
```

## 编译脚本

```shell
#!/bin/bash
cd build

if [[ “$@“ =~ "-c" ]];then
    echo "----------------------------cmake clean----------------------------"
    rm -rf CMakeCache.txt
    rm -rf CMakeFiles
    rm -rf cmake_install.cmake
    rm -rf Makefile
    rm -rf CTestTestfile.cmake
    exit
fi
 
if [[ “$@“ =~ "-r" ]];then
    echo "----------------------------cmake release----------------------------"
    cmake -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK_HOME/build/cmake/android.toolchain.cmake \
        -DCMAKE_BUILD_TYPE=Release \
        -DANDROID_NDK=$ANDROID_NDK_HOME \
        -DANDROID_ABI=armeabi-v7a \
        -DANDROID_TOOLCHAIN=clang \
        -DANDROID_PLATFORM=android-21 \
        -DANDROID_STL=c++_shared \
        ..
else      
    echo "----------------------------cmake debug----------------------------"
    cmake -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK_HOME/build/cmake/android.toolchain.cmake \
        -DCMAKE_BUILD_TYPE=Debug \
        -DANDROID_NDK=$ANDROID_NDK_HOME \
        -DANDROID_ABI=armeabi-v7a \
        -DANDROID_TOOLCHAIN=clang \
        -DANDROID_PLATFORM=android-21 \
        -DANDROID_STL=c++_shared \
        ..
fi

make

```

## 常见问题

### 1. cmake版本不对

```shell
CMake Error at /home/weitaiping/study/android-ndk-r21e/build/cmake/android.toolchain.cmake:35 (cmake_minimum_required):
  CMake 3.6.0 or higher is required.  You are running version 3.5.1
```
解决办法: 
* 更新cmake版本到3.6.0以上
or
* 修改`$ANDROID_NDK_HOME/build/cmake/android.toolchain.cmake`文件, 将`cmake_minimum_required(VERSION 3.6.0)`改为`cmake_minimum_required(VERSION 3.5.1)`