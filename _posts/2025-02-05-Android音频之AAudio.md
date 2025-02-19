---
title: Android音频之AAudio
tags: Android Audio AAudio
---

# Android音频之AAudio

AAudio 是 Android 提供的原生音频 API，专为低延迟、高性能音频应用而设计。自 Android O（8.0）起引入以来，AAudio 逐步取代了 OpenSL ES，成为构建高质量音频应用的首选接口。本文将介绍 AAudio 的基本概念、核心功能、开发入门及调试技巧，并结合 Android NDK 指南和 Android 源码文档进行说明。

---

## 1. 概述

AAudio 提供了一个 C 风格的音频 API，允许开发者直接访问底层音频硬件。相比传统的 Java 层音频 API（如 AudioTrack、MediaPlayer），AAudio 在延迟、资源开销和实时性方面有显著优势，非常适合专业音频、音乐制作、语音通信和游戏等场景。

主要特点包括：
- **低延迟**：专为低延迟音频传输设计，能够满足实时音频处理的要求。
- **高性能**：直接与音频硬件交互，减少了不必要的抽象层，提高了整体性能。
- **简单易用**：采用 C 接口，使得基于 C/C++ 的音频应用可以直接调用，实现跨平台兼容性。

---

## 2. 核心概念与组件

AAudio 的开发主要围绕以下几个核心组件和概念展开：

### 2.1 AAudioStream & AAudioStreamBuilder
- **AAudioStream**：代表一个音频流，可以用于播放或录制音频数据。通过该流，应用程序可以写入或读取 PCM 数据。
- **AAudioStreamBuilder**：用于构建 AAudioStream 的对象，开发者可以通过设置采样率、通道数、音频格式、共享模式（共享或独占）等参数来定制音频流。

### 2.2 回调机制
- **流回调 (Stream Callback)**：通过注册回调函数，应用可以在音频数据准备就绪时获得通知，从而实现实时音频处理和低延迟播放。回调函数会在独立的线程上运行，确保音频流的连续性。

### 2.3 模式选择
- **共享模式 vs 独占模式**：  
  - **共享模式**允许多个应用共享同一个音频硬件资源，适用于一般多媒体播放场景。  
  - **独占模式**则能提供更低延迟的性能，通常用于实时性要求较高的专业应用。

---

## 3. Getting Started

根据 Android NDK 开发指南的说明，使用 AAudio 的基本步骤如下：

1. **创建 AAudioStreamBuilder**  
   利用 `AAudioStreamBuilder` 创建流对象，并设置所需的参数：
   ```c
   AAudioStreamBuilder *builder = nullptr;
    aaudio_result_t result = AAudio_createStreamBuilder(&builder);
    if (result != AAUDIO_OK) {
        LOGE("Error creating stream builder: %s", AAudio_convertResultToText(result));
    }
   
   AAudioStreamBuilder_setFormat(builder, AAUDIO_FORMAT_PCM_I16);
   AAudioStreamBuilder_setChannelCount(builder, 2); // stereo
   AAudioStreamBuilder_setSampleRate(builder, 48000); // 48KHz
   AAudioStreamBuilder_setErrorCallback(builder, ::errorCallback, this);
   AAudioStreamBuilder_setDeviceId(builder, playbackDeviceId_);  // 选择设备 可以通过 Java AudioManager获取设备Id
   AAudioStreamBuilder_setDirection(builder, AAUDIO_DIRECTION_OUTPUT);  // 选择 输入 or 输出
   AAudioStreamBuilder_setSharingMode(builder, AAUDIO_SHARING_MODE_SHARED);// 设置模式：共享模式或独占模式
   // 默认设备Id是 0; AAUDIO_UNSPECIFIED
   // 默认方向是输出; AAUDIO_DIRECTION_OUTPUT
   // 默认模式是共享模式; AAUDIO_SHARING_MODE_SHARED
   ```

2. **打开音频流**  
   使用 builder 打开 AAudio 流：
   ```c
   AAudioStream* stream = NULL;
   AAudioStreamBuilder_openStream(builder, &stream);
   
   ```

3. **启动播放/录制**  
   对于播放音频：
   ```c
   AAudioStream_requestStart(stream);
   // 写入 PCM 数据……
   AAudioStream_write(stream, audioBuffer, bufferSize, timeoutNanos);
   
   ```
   对于录制音频，同理调用 `AAudioStream_read` 获取数据。

4. **关闭流并清理**  
   在结束后关闭流并删除：
   ```c
   AAudioStream_requestStop(stream);
   AAudioStream_close(stream);
   AAudioStreamBuilder_delete(builder);
   ```

---

## 4. 核心优势与使用场景

### 4.1 低延迟优势
AAudio 能够显著降低音频播放和录制的延迟，适用于实时音频处理、在线通话、音乐制作等需要即时响应的场景。

### 4.2 高性能
通过直接操作底层硬件，AAudio 降低了系统调用的开销，在高负载环境下仍能保持稳定性能，适合对资源敏感的应用。

### 4.3 专业音频应用
AAudio 支持独占模式、流回调和高级音频流配置，使得开发者能够构建高保真、低延迟的专业音频应用，如实时混音、音频效果处理等。

---

## 5. 其它

相关属性,`AAudio`根据这两个属性来判断系统是否支持`MMap`和`Legacy`.只不过车载音频这两个属性的值一般为1(`AAUDIO_POLICY_NEVER`)，都不支持`MMap`和`Legacy`。
```c
/**
 * Read system property.
 * @return AAUDIO_UNSPECIFIED, AAUDIO_POLICY_NEVER or AAUDIO_POLICY_AUTO or AAUDIO_POLICY_ALWAYS
 */
int32_t AAudioProperty_getMMapPolicy();
#define AAUDIO_PROP_MMAP_POLICY           "aaudio.mmap_policy"

/**
 * Read system property.
 * @return AAUDIO_UNSPECIFIED, AAUDIO_POLICY_NEVER or AAUDIO_POLICY_AUTO or AAUDIO_POLICY_ALWAYS
 */
int32_t AAudioProperty_getMMapExclusivePolicy();
#define AAUDIO_PROP_MMAP_EXCLUSIVE_POLICY "aaudio.mmap_exclusive_policy"
```
---

**参考资料**  
- [AAudio Getting Started - Android NDK](https://developer.android.google.cn/ndk/guides/audio/aaudio/aaudio?hl=zh-cn#getting-started) 
- [Android AAudio 文档](https://source.android.google.cn/docs/core/audio/aaudio?hl=zh-cn) 
- [Android AAudio 源码](http://androidxref.com/9.0.0_r3/xref/frameworks/av/media/libaaudio/include/aaudio/AAudio.h) 
