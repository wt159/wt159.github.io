---
title: Android13添加AudioControl接口
tags: android13 android AudioControl
---

## 概述

`AudioControl` 模块是 Android Automotive 平台的一部分，它提供了对车载音频系统的控制接口。该模块负责管理音频路由、音量调节、音频焦点管理等功能，确保在车载环境中音频体验的流畅性和一致性。`AudioControl` 模块通常通过硬件抽象层（HAL）实现，与底层音频硬件通信，以执行各种音频操作。

### 模块架构

`AudioControl` 模块是基于 Android HAL 的一部分，其架构设计如下：

- **应用层（Application Layer）**：车载应用程序通过 CarAudioManager 发起音频请求，例如播放媒体、语音助手等。
- **框架层（Framework Layer）**：Android 框架通过 Car Audio Service 接受音频请求并将其传递到 Audio HAL。
- **硬件抽象层（HAL）**：`AudioControl` 模块作为 HAL 的一部分，实现了 `IAudioControl` 接口，负责将来自框架的音频请求翻译成硬件可执行的操作。
- **硬件层（Hardware Layer）**：音频硬件接收并执行 HAL 提供的指令，实现实际的音频输出和控制。

### `IAudioControl` 接口

`IAudioControl` 接口是 `AudioControl` 模块的核心接口，它定义了一系列方法来控制和管理车载音频。该接口通常通过 AIDL（Android Interface Definition Language）或 HIDL（HAL Interface Definition Language）定义。以下是一些常见的方法：

- **`onAudioFocusChange(int usage, int zoneId, int focusChange)`**：处理音频焦点的变化。音频焦点用于管理不同音频源之间的优先级和混音策略。
- **`onDevicesToMuteChange(MutingInfo[] mutingInfos)`**：静音或取消静音特定音频设备，例如在导航提示或警报音播放时静音媒体音频。
- **`onDevicesToDuckChange(DuckingInfo[] duckingInfos)`**：在某些特定事件发生时（如导航提示、电话来电等），对特定音频设备（如音乐播放设备）进行音量降低的操作。
- **`setBalanceTowardRight(float balance)`**：设置左右声道的平衡。
- **`setFadeTowardFront(float fade)`**：设置前后声场的淡入淡出。
  
这些接口方法通常是 `oneway` 的，这意味着它们是异步的，不会阻塞调用线程。具体的实现由 HAL 驱动和车载音频硬件共同完成。

### 功能和特点

`AudioControl` 模块的功能特点包括：

- **音频路由控制**：根据音频类型和车载环境动态路由音频信号到正确的输出设备，例如将导航提示音发送到驾驶座的扬声器，而不是后座。
- **音频焦点管理**：管理多个音频源的焦点请求和冲突，以确保重要的音频内容不会被其他音频打断。
- **设备静音和音量控制**：提供对车载音频设备的静音和音量控制，以适应不同的驾驶场景和用户偏好。
- **自定义音频策略**：支持车载厂商和应用开发者定义和实现自定义音频策略，以满足特定车辆或市场的需求。

## 目录结构

```shell
hardware/interfaces/automotive/audiocontrol/
├── 1.0
├── 2.0
└── aidl
    ├── aidl_api
    │   └── android.hardware.automotive.audiocontrol
    │       ├── 1
    │       ├── 2
    │       └── current
    ├── android
    ├── Android.bp
    ├── default
    └── vts
```

 `Android13` 使用的是 `AIDL`, 没有使用 `1.0` 和 `2.0` 版本的 `HIDL` 。

 - **AIDL 文件定义**：`IAudioControl.aidl` 文件定义了 HAL 接口的规范，位于 `hardware/interfaces/automotive/audiocontrol/aidl/android/hardware/automotive/audiocontrol/` 目录下。

## 添加接口

```diff
diff --git a/automotive/audiocontrol/aidl/android/hardware/automotive/audiocontrol/IAudioControl.aidl b/automotive/audiocontrol/aidl/android/hardware/automotive/audiocontrol/IAudioControl.aidl
index 0ffcd5e..8a5c294 100644
--- a/automotive/audiocontrol/aidl/android/hardware/automotive/audiocontrol/IAudioControl.aidl
+++ b/automotive/audiocontrol/aidl/android/hardware/automotive/audiocontrol/IAudioControl.aidl
@@ -183,4 +183,5 @@ interface IAudioControl {
      *                 interface.
      */
     oneway void registerGainCallback(in IAudioGainCallback callback);
+    oneway void setBalance(in int value);
 }
diff --git a/automotive/audiocontrol/aidl/default/AudioControl.cpp b/automotive/audiocontrol/aidl/default/AudioControl.cpp
index a121f8b..36074d0 100644
--- a/automotive/audiocontrol/aidl/default/AudioControl.cpp
+++ b/automotive/audiocontrol/aidl/default/AudioControl.cpp
@@ -214,6 +214,11 @@ binder_status_t AudioControl::dump(int fd, const char** args, uint32_t numArgs)
     }
 }
 
+ndk::ScopedAStatus AudioControl::setBalance(int in_value) {
+    LOG(DEBUG) << ": " << __func__ << "in_value: " << in_value;
+    // TODO： Implement setBalance
+    return ndk::ScopedAStatus::ok();
+}
+
 binder_status_t AudioControl::dumpsys(int fd) {
     if (mFocusListener == nullptr) {
         dprintf(fd, "No focus listener registered\n");
diff --git a/automotive/audiocontrol/aidl/default/AudioControl.h b/automotive/audiocontrol/aidl/default/AudioControl.h
index 16b80e8..bfe2780 100644
--- a/automotive/audiocontrol/aidl/default/AudioControl.h
+++ b/automotive/audiocontrol/aidl/default/AudioControl.h
@@ -54,6 +54,8 @@ class AudioControl : public BnAudioControl {
 
     binder_status_t dump(int fd, const char** args, uint32_t numArgs) override;
 
+    ndk::ScopedAStatus setBalance(int in_value) override;
+
   private:
     // This focus listener will only be used by this HAL instance to communicate with
     // a single instance of CarAudioService. As such, it doesn't have explicit serialization.
```

## 更新&编译

```shell
m android.hardware.automotive.audiocontrol-update-api
cd hardware/interfaces/automotive/audiocontrol/aidl/aidl_api/android.hardware.automotive.audiocontrol
cp -rf current/* 2/
cd -
mmm hardware/interfaces/automotive/audiocontrol/aidl
```

### 报错

```shell
[  0% 9/1305] Verify hardware/interfaces/automotive/audiocontrol/aidl/aidl_api/android.hardware.automotive.audiocontrol/2 files have not been modified
FAILED: out/soong/.intermediates/hardware/interfaces/automotive/audiocontrol/aidl/android.hardware.automotive.audiocontrol-api/checkhash_2.timestamp
if [ $(cd 'hardware/interfaces/automotive/audiocontrol/aidl/aidl_api/android.hardware.automotive.audiocontrol/2' && { find ./ -name "*.aidl" -print0 | LC_ALL=C sort -z | xargs -0 sha1sum && echo 1; } | sha1sum | cut -d " " -f 1) = $(tail -1 'hardware/interfaces/automotive/audiocontrol/aidl/aidl_api/android.hardware.automotive.audiocontrol/2/.hash') ]; then touch out/soong/.intermediates/hardware/interfaces/automotive/audiocontrol/aidl/android.hardware.automotive.audiocontrol-api/checkhash_2.timestamp; else cat 'system/tools/aidl/build/message_check_integrity.txt' && exit 1; fi
###############################################################################
# ERROR: Modification detected of stable AIDL API file                        #
###############################################################################
Above AIDL file(s) has changed, resulting in a different hash. Hash values may
be checked at runtime to verify interface stability. If a device is shipped
with this change by ignoring this message, it has a high risk of breaking later
when a module using the interface is updated, e.g., Mainline modules.
13:59:48 ninja failed with: exit status 1

#### failed to build some targets (18 seconds) ####
```

### 解决方法

获取当前 `api` 的 `hash` 并 更新到 `.hash` 文件，然后重新编译

```shell
cd 'hardware/interfaces/automotive/audiocontrol/aidl/aidl_api/android.hardware.automotive.audiocontrol/2' && { find ./ -name "*.aidl" -print0 | LC_ALL=C sort -z | xargs -0 sha1sum && echo 1; } | sha1sum | cut -d " " -f 1 > .hash
cd -
mmm hardware/interfaces/automotive/audiocontrol/aidl
```
