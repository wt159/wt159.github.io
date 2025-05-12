---
title: Android编译framework和car模块
tags: android framework car app AAOS
---

## 概述

[参考链接](http://ahaoframework.tech/)

### 编译 framework 模块

```shell
source 
lunch xxx-eng
# Android10 及以前
make framework
# Android11 及以后
#make framework-minus-apex
# 相关Android.bp文件
# frameworks/base/Android.bp

# Android10 及以前
cp out/target/common/obj/JAVA_LIBRARIES/framework_intermediates/classes.jar framework.jar
# Android11 及以后
cp out/target/common/obj/JAVA_LIBRARIES/framework-minus-apex_intermediates/classes.jar framework.jar
```

### 编译 framework services 模块

```shell
source 
lunch xxx-eng
# Android10 及以前
make services
```

#### 动态更新 services 模块

```shell
adb root
adb remount
adb push system/framework/services.jar /system/framework/
adb push system/framework/framework.jar /system/framework/
adb push system/framework/car-frameworks-service.jar /system/framework/
adb push system/framework/oat/arm64/services.art /system/framework/oat/arm64/
adb push system/framework/oat/arm64/services.odex /system/framework/oat/arm64/
adb push system/framework/oat/arm64/services.vdex /system/framework/oat/arm64/
adb shell sync
```

### 编译 car 模块

```shell
source 
lunch xxx-eng
make android.car
make CarService
# Android13 及以后
make make CarServiceUpdatable

cp out/target/common/obj/JAVA_LIBRARIES/android.car_intermediates/classes.jar android.car.jar
# 相关Android.bp文件
# packages/services/Car/car-lib/Android.bp
# packages/services/Car/service/Android.bp
```

#### 动态更新 car 模块

```shell
adb root
adb remount
adb push CarService /system/priv-app/
# Android13 及以后
adb push CarServiceUpdatable /system/apex/com.android.car.framework/priv-app/CarServiceUpdatable@TQ3A.230805.001.S1/
adb shell sync
```

Android13以后把CarService模块分成了两部分，一部分是 CarService 模块，另一部分是 CarServiceUpdatableNonModule 模块。

Android13以前CarService.apk的aapt dump命令输出如下：
```shell
$ aapt dump strings out/target/product/xxx/system/priv-app/CarService/CarService.apk |grep -e service -e "com."
String #1171: Car service
String #1533: Running services
String #1763: android.car.input.service/.DefaultInputService
String #1772: car_navigation_service
String #1773: com.android.bluetooth/com.android.bluetooth.avrcpcontroller.BluetoothMediaBrowserService
String #1774: com.android.car.media
String #1775: com.android.car.messenger/.MessengerService#bind=startForeground,user=foreground,trigger=userUnlocked
String #1776: com.android.car.rotary/com.android.car.rotary.RotaryService
String #1777: com.android.car.trust.TOKEN_HANDLE
String #1778: com.android.car.user.CarUserNoticeService
String #1779: com.android.car/com.android.car.pm.ActivityBlockingActivity
String #1780: com.android.certinstaller
String #1781: com.android.server.location.ComprehensiveCountryDetector
String #1782: com.android.settings
String #1783: com.android.systemui,com.google.android.permissioncontroller/com.android.permissioncontroller.permission.ui.GrantPermissionsActivity,com.android.permissioncontroller/com.android.permissioncontroller.permission.ui.GrantPermissionsActivity,android/com.android.internal.app.ResolverActivity,com.android.mtp/com.android.mtp.ReceiverActivity,com.android.server.telecom/com.android.server.telecom.components.UserCallActivity
String #1784: com.google.android.apps.maps/com.google.android.maps.MapsActivity
String #1785: com.google.android.car.defaultstoragemonitoringcompanionapp/.ExcessiveIoIntentReceiver
String #1786: com.google.android.car.defaultstoragemonitoringcompanionapp/.MainActivity
String #1787: com.google.android.car.kitchensink/.UserNoiticeDemoUiService
String #1870: vehicle_map_service
```

Android13以后CarService.apk的aapt dump命令输出如下：
```shell
$ aapt dump strings out/target/product/xxx/system/priv-app/CarService/CarService.apk |grep -e service -e "com."
String #1: Car service
String #6: com.android.car.settings/com.android.car.settings.admin.FactoryResetActivity
String #7: com.android.car.settings/com.android.car.settings.admin.NewUserDisclaimerActivity
String #45: Bilens inputservice
String #64: VMS-klientservice
String #719: Invoerservice van auto
String #726: VMS-clientservice
String #2150: Car input service
```
```shell
$ aapt dump strings out/target/product/xxx/system/apex/com.android.car.framework/priv-app/CarServiceUpdatable@TQ3A.230805.001.S1/CarServiceUpdatable.apk |grep -e service -e "com."
String #22: Car service
String #42: Controls which service can bind to OemCarService
String #64: car_evs_service
String #65: car_navigation_service
String #66: com.android.car.cluster.home/.ClusterHomeActivity
String #67: com.android.car.media
String #68: com.android.car.messenger/.MessengerService#bind=startForeground,user=foreground,trigger=userUnlocked
String #69: com.android.car.radio/.service.RadioAppService
String #70: com.android.car.rotary/com.android.car.rotary.RotaryService
String #71: com.android.server.location.ComprehensiveCountryDetector
String #72: com.android.systemui,com.google.android.permissioncontroller/com.android.permissioncontroller.permission.ui.GrantPermissionsActivity,com.android.permissioncontroller/com.android.permissioncontroller.permission.ui.GrantPermissionsActivity,android/com.android.internal.app.ResolverActivity,com.android.mtp/com.android.mtp.ReceiverActivity,com.android.server.telecom/com.android.server.telecom.components.UserCallActivity
String #73: com.android.systemui/com.android.systemui.car.activity.ActivityBlockingActivity
String #74: com.android.systemui/com.android.systemui.car.activity.ContinuousBlankActivity
String #75: com.google.android.apps.maps/com.google.android.maps.MapsActivity
String #76: com.google.android.car.defaultstoragemonitoringcompanionapp/.ExcessiveIoIntentReceiver
String #77: com.google.android.car.defaultstoragemonitoringcompanionapp/.MainActivity
String #78: com.google.android.car.evs/com.google.android.car.evs.CarEvsCameraPreviewActivity
String #79: com.google.android.car.kitchensink/.UserNoticeDemoUiService
String #106: vehicle_map_service
```