---
title: ubuntu搭建vscode阅读android_car_java环境
tags: ubuntu vscode android java audio AAOS
---

# ubuntu搭建vscode阅读android_car_java环境

## 操作环境

* `Ubuntu-16.04.1`

* `VSCode 1.92.2`

## 插件安装

`vscode` 安装插件 `Extension Pack for Java`

## 环境配置

```shell
# 根据实际环境修改源码路径
cd android_code_path
cd packages/services/Car
mkdir mylib
ln -s ./../../../../frameworks/ mylib/frameworks

mkdir mylib/my_framework_core
mkdir mylib/my_framework_core/android
cd mylib/my_framework_core/android
ln -s ../../frameworks/base/core/java/android/content/ content
ln -s ../../frameworks/base/core/java/android/provider/ provider
ln -s ../../frameworks/base/core/java/android/util/ util
ln -s ../../frameworks/base/core/java/android/os/ os
cd -
```

```shell
# 根据实际环境修改版本
cd ~/.gradle/wrapper/dists/gradle-8.9-bin/90cnw93cvbtalezasaz0blq0a
wget https://mirrors.cloud.tencent.com/gradle/gradle-8.9-bin.zip
rm -rf gradle-8.9-bin.zip.*
cd -
```

## vscode

### 打开源码

> vscode 打开源码目录 `packages/services/Car`

### 配置 `settings.json`

> `ctrl + shift + p` 输入 `首选项: 打开工作区设置(JSON)`

```json
{
    "java.project.sourcePaths": [
        "service/src",
        "service-builtin/src",
        "car-lib/src",
        "car-builtin-lib/src",
        "mylib/frameworks/libs/modules-utils/java",
        "mylib/frameworks/base/media/java",
        "mylib/frameworks/base/services/core/java",
        //"mylib/frameworks/base/core/java", //这个里面的源码太多了，需要编译很久，按照自己的需求来确定是否打开
        "mylib/my_framework_core"
    ],
    "java.jdt.ls.vmargs": "-XX:+UseParallelGC -XX:GCTimeRatio=4 -XX:AdaptiveSizePolicyWeight=90 -Dsun.zip.disableMemoryMapping=true -Xmx8G -Xms100m -Xlog:disable",
}
```
