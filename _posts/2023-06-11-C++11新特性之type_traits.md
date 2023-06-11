---
title: C++11新特性之type_traits
tags: c++ C++11
mermaid: true
---

## 概述

`C++11`中的`type_traits`是一个模板元编程库，它提供了一些模板类和模板函数，用于在编译期间进行类型信息的查询和转换。它可以帮助我们编写更加通用和可靠的代码，避免了很多运行时类型检查和转换的开销。

## 中文官方网址

[zh.cppreference](https://zh.cppreference.com/w/cpp/header/type_traits)

## 头文件

```cpp
#include <type_traits>
```

## 基本类型特性

以下模板类和函数用于判断基本类型的特性：

| 模板类/函数 | 描述 |
| --- | --- |
| `std::is_void` | 是否为`void`类型 |
| `std::is_integral` | 是否为整数类型 |
| `std::is_floating_point` | 是否为浮点数类型 |
| `std::is_array` | 是否为数组类型 |
| `std::is_pointer` | 是否为指针类型 |
| `std::is_lvalue_reference` | 是否为左值引用类型 |
| `std::is_rvalue_reference` | 是否为右值引用类型 |
| `std::is_member_object_pointer` | 是否为成员变量指针类型 |
| `std::is_member_function_pointer` | 是否为成员函数指针类型 |
| `std::is_enum` | 是否为枚举类型 |
| `std::is_union` | 是否为联合类型 |
| `std::is_class` | 是否为类类型 |
| `std::is_function` | 是否为函数类型 |

以下模板类和函数用于判断类型的属性：

| 模板类/函数 | 描述 |
| --- | --- |
| `std::is_reference` | 是否为引用类型 |
| `std::is_arithmetic` | 是否为算术类型 |
| `std::is_fundamental` | 是否为基本类型 |
| `std::is_object` | 是否为对象类型 |
| `std::is_scalar` | 是否为标量类型 |
| `std::is_compound` | 是否为复合类型 |
| `std::is_member_pointer` | 是否为成员指针类型 |

以下模板类和函数用于类型转换：

| 模板类/函数 | 描述 |
| --- | --- |
| `std::remove_const` | 去除类型的`const`修饰符 |
| `std::remove_volatile` | 去除类型的`volatile`修饰符 |
| `std::remove_cv` | 去除类型的`const`和`volatile`修饰符 |
| `std::add_const` | 添加`const`修饰符 |
| `std::add_volatile` | 添加`volatile`修饰符 |
| `std::add_cv` | 添加`const`和`volatile`修饰符 |
| `std::add_lvalue_reference` | 添加左值引用修饰符 |
| `std::add_rvalue_reference` | 添加右值引用修饰符 |
| `std::remove_reference` | 去除引用修饰符 |
| `std::decay` | 将类型转换为对应的函数参数类型 |

以下模板类和函数用于条件判断：

| 模板类/函数 | 描述 |
| --- | --- |
| `std::conditional` | 根据条件选择不同的类型 |
| `std::enable_if` | 根据条件判断是否启用某个函数模板 |
| `std::is_same` | 判断两个类型是否相同 |
| `std::is_convertible` | 判断一个类型是否可以隐式转换为另一个类型 |
| `std::is_base_of` ` | 判断一个类是否是另一个类的基类 |
| `std::is_assignable` | 判断一个类型是否可以被赋值为另一个类型 |
| `std::is_constructible` | 判断一个类型是否可以被构造为另一个类型 |
| `std::is_trivially_assignable` | 判断一个类型是否可以被快速赋值为另一个类型 |
| `std::is_trivially_constructible` | 判断一个类型是否可以被快速构造为另一个类型 |
| `std::is_nothrow_assignable` | 判断一个类型是否可以在不抛出异常的情况下被赋值为另一个类型 |
| `std::is_nothrow_constructible` | 判断一个类型是否可以在不抛出异常的情况下被构造为另一个类型 |

以下模板类和函数用于获取类型属性：

| 模板类/函数 | 描述 |
| --- | --- |
| `std::aligned_storage` | 获取指定大小和对齐方式的内存块类型 |
| `std::aligned_union` | 获取指定类型中占用空间最大的类型 |
| `std::decay` | 将类型转换为对应的函数参数类型 |
| `std::common_type` | 获取多个类型的公共类型 |
| `std::underlying_type` | 获取枚举类型的底层类型 |
| `std::result_of` | 获取可调用对象的返回类型 |

以下模板类和函数用于类型转换和比较：

| 模板类/函数 | 描述 |
| --- | --- |
| `std::enable_if` | 根据条件判断是否启用某个函数模板 |
| `std::conditional` | 根据条件选择不同的类型 |
| `std::common_type` | 获取多个类型的公共类型 |
| `std::declval` | 获取指定类型的右值引用 |
| `std::is_convertible` | 判断一个类型是否可以隐式转换为另一个类型 |
| `std::is_same` | 判断两个类型是否相同 |
| `std::is_base_of` | 判断一个类是否是另一个类的基类 |
| `std::is_assignable` | 判断一个类型是否可以被赋值为另一个类型 |
| `std::is_constructible` | 判断一个类型是否可以被构造为另一个类型 |
| `std::is_trivially_assignable` | 判断一个类型是否可以被快速赋值为另一个类型 |
| `std::is_trivially_constructible` | 判断一个类型是否可以被快速构造为另一个类型 |
| `std::is_nothrow_assignable` | 判断一个类型是否可以在不抛出异常的情况下被赋值为另一个类型 |
| `std::is_nothrow_constructible` | 判断一个类型是否可以在不抛出异常的情况下被构造为另一个类型 |
| `std::rank` | 获取数组类型的维度 |
| `std::extent` | 获取数组类型的指定维度大小 |
