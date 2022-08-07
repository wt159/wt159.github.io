---
title: cmake之install
tags: tools cmake
mermaid: true
---

# make install打包安装

install用于指定在安装时运行的规则。它可以用来安装很多内容，可以包括目标二进制、动态库、静态库以及文件、目录、脚本等：

```makefile
install(TARGETS <target>... [...])
install({FILES | PROGRAMS} <file>... [...])
install(DIRECTORY <dir>... [...])
install(SCRIPT <file> [...])
install(CODE <code> [...])
install(EXPORT <export-name> [...])
```

# 指定安装位置

`CMAKE_INSTALL_PREFIX`为cmake内部变量，用于指定cmake执行install的时候，安装的路径前缀

## 1. 编译时确定

`cmake -DCMAKE_INSTALL_PREFIX=/usr/local ../`

## 2. cmake项目文件中确定

```
set(CMAKE_INSTALL_PREFIX /usr/local)
```
