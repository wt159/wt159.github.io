---
title: Android音频之audioserver
tags: Android Audio AAOS audioserver
---

# Android 音频之 audioserver

## 概述
`audioserver` 是 Android 系统中负责管理音频功能的核心服务。它处理音频设备的输入输出、音频路由、音量控制、音频焦点管理等任务，是 Android 音频系统的关键组件之一。

## 组成模块
- **AudioFlinger**：核心模块，负责音频数据的混合和输出,以及音效处理。
- **AudioPolicyService**：负责音频策略管理（如设备选择、路由规则）。

## 启动过程
`audioserver` 在 Android 系统启动时由 `init` 进程启动。它的启动脚本通常位于：
```
/system/etc/init/audioserver.rc
```
在启动脚本中，`audioserver` 会加载所需的共享库并启动相关服务。

## 源码位置

> frameworks/av/media/audioserver/audioserver.rc
> frameworks/av/media/audioserver/main_audioserver.cpp

## 源码流程

```c++
--> main()
----> if (doLog && (childPid = fork()) != 0) {
----> MediaLogService::instantiate();
----> } else {
----> AudioFlinger::instantiate();
------> return sm->addService(String16("media.audio_flinger"), new AudioFlinger(), allowIsolated,dumpFlags);
--------> new AudioFlinger();
--------> AudioFlinger::onFirstRef();
----> AudioPolicyService::instantiate();
------> return sm->addService(String16("media.audio_policy"), new AudioPolicyService(), allowIsolated,dumpFlags);
------> new AudioPolicyService();
------> AudioPolicyService::onFirstRef();
----> }
```

`AudioFlinger` 和 `AudioPolicyService`直接相互调用，是靠 `libaudioclient.so`的`AudioSystem.cpp`中 `AudioSystem::get_audio_policy_service()` 和 `AudioSystem::get_audio_flinger()`函数,具体实现如下：

```c++
// client singleton for AudioPolicyService binder interface
sp<IAudioPolicyService> AudioSystem::gAudioPolicyService;
sp<AudioSystem::AudioPolicyServiceClient> AudioSystem::gAudioPolicyServiceClient;
// establish binder interface to AudioPolicy service
const sp<IAudioPolicyService> AudioSystem::get_audio_policy_service()
{
    sp<IAudioPolicyService> ap;
    sp<AudioPolicyServiceClient> apc;
    {
        Mutex::Autolock _l(gLockAPS);
        if (gAudioPolicyService == 0) {
            sp<IServiceManager> sm = defaultServiceManager();
            sp<IBinder> binder = sm->getService(String16("media.audio_policy"));
            if (gAudioPolicyServiceClient == NULL) {
                gAudioPolicyServiceClient = new AudioPolicyServiceClient();
            }
            binder->linkToDeath(gAudioPolicyServiceClient);
            gAudioPolicyService = interface_cast<IAudioPolicyService>(binder);
            LOG_ALWAYS_FATAL_IF(gAudioPolicyService == 0);
            apc = gAudioPolicyServiceClient;
            // Make sure callbacks can be received by gAudioPolicyServiceClient
            ProcessState::self()->startThreadPool();
        }
        ap = gAudioPolicyService;
    }
    if (apc != 0) {
        int64_t token = IPCThreadState::self()->clearCallingIdentity();
        ap->registerClient(apc);
        ap->setAudioPortCallbacksEnabled(apc->isAudioPortCbEnabled());
        ap->setAudioVolumeGroupCallbacksEnabled(apc->isAudioVolumeGroupCbEnabled());
        IPCThreadState::self()->restoreCallingIdentity(token);
    }
    return ap;
}
```

```c++
// client singleton for AudioFlinger binder interface
sp<IAudioFlinger> AudioSystem::gAudioFlinger;
sp<AudioSystem::AudioFlingerClient> AudioSystem::gAudioFlingerClient;
// establish binder interface to AudioFlinger service
const sp<IAudioFlinger> AudioSystem::get_audio_flinger()
{
    sp<IAudioFlinger> af;
    sp<AudioFlingerClient> afc;
    bool reportNoError = false;
    {
        Mutex::Autolock _l(gLock);
        if (gAudioFlinger == 0) {
            sp<IServiceManager> sm = defaultServiceManager();
            sp<IBinder> binder;
            do {
                binder = sm->getService(String16("media.audio_flinger"));
                ...
            } while (true);
            if (gAudioFlingerClient == NULL) {
                gAudioFlingerClient = new AudioFlingerClient();
            } else {
                reportNoError = true;
            }
            binder->linkToDeath(gAudioFlingerClient);
            gAudioFlinger = interface_cast<IAudioFlinger>(binder);
            LOG_ALWAYS_FATAL_IF(gAudioFlinger == 0);
            afc = gAudioFlingerClient;
            // Make sure callbacks can be received by gAudioFlingerClient
            ProcessState::self()->startThreadPool();
        }
        af = gAudioFlinger;
    }
    if (afc != 0) {
        int64_t token = IPCThreadState::self()->clearCallingIdentity();
        af->registerClient(afc);
        IPCThreadState::self()->restoreCallingIdentity(token);
    }
    if (reportNoError) reportError(NO_ERROR);
    return af;
}
```

## 日志与调试
### 查看日志
`audioserver` 的日志可以通过 `logcat` 查看。使用以下命令过滤 `audioserver` 的日志：
```bash
adb logcat -s audioserver
```
常见的日志标签包括：
- `AudioFlinger`：音频混合和输出的日志。
- `AudioPolicyService`：音频策略管理的日志。
- `AudioEffect`：音频效果处理的日志。

### 调试命令
- **查看音频设备状态**：
  ```bash
  adb shell dumpsys media.audio_flinger
  ```
- **查看音频策略**：
  ```bash
  adb shell dumpsys media.audio_policy
  ```
- **查看音频焦点状态**：
  ```bash
  adb shell dumpsys audio
  ```

## AudioFlinger

## 概述
`AudioFlinger` 是 Android 音频系统的核心组件之一，负责音频数据的混合和输出。它是 `audioserver` 的一部分，处理音频流的生命周期、音频数据的混合、音频设备的输出等任务。

## 主要功能
1. **音频流管理**：管理音频流的创建、销毁和生命周期。
2. **音频数据混合**：将多个音频流的数据混合成一个输出流。
3. **音频设备输出**：将混合后的音频数据输出到音频设备（如扬声器、耳机等）。
4. **音频效果处理**：应用音频效果（如均衡器、重低音等）。
5. **音频缓冲区管理**：管理音频数据的缓冲区，确保音频数据的连续性和低延迟。

## 源码位置

> frameworks/av/services/audioflinger/AudioFlinger.h
> frameworks/av/services/audioflinger/AudioFlinger.cpp

## 源码流程

### 1. **初始化**
`AudioFlinger` 在 `audioserver` 启动时初始化, 它创建了音频HAL的工厂接口`mDevicesFactoryHal`，也创建了音效的工厂接口`mEffectsFactoryHal`。


主要流程如下：

```c++
----> AudioFlinger::instantiate();
------> return sm->addService(String16("media.audio_flinger"), new AudioFlinger(), allowIsolated,dumpFlags);
--------> new AudioFlinger();
----------> mDevicesFactoryHal = DevicesFactoryHalInterface::create();
// frameworks/av/media/libaudiohal/DevicesFactoryHalInterface.cpp
------------> createPreferredImpl<DevicesFactoryHalInterface>("android.hardware.audio", "IDevicesFactory")
--------------> createHalService(*version, interface, &rawInterface)
// frameworks/av/media/libaudiohal/FactoryHalHidl.cpp
----------------> const std::string libName = "libaudiohal@" + version + ".so";
----------------> const std::string factoryFunctionName = "create" + interface;
----------------> handle = dlopen(libName.c_str(), dlMode);
----------------> *(void **)(&factoryFunction) = dlsym(handle, factoryFunctionName.c_str());
----------------> *rawInterface = (*factoryFunction)();
// frameworks/av/media/libaudiohal/impl/DevicesFactoryHalHybrid.cpp
------------------> void* createIDevicesFactory();
------------------> return service ? new CPP_VERSION::DevicesFactoryHalHybrid(service) : nullptr;
--------------------> mLocalFactory(new DevicesFactoryHalLocal()),
--------------------> mHidlFactory(new DevicesFactoryHalHidl(hidlFactory))
----------> mEffectsFactoryHal = EffectsFactoryHalInterface::create();
------------> createPreferredImpl<EffectsFactoryHalInterface>("android.hardware.audio.effect", "IEffectsFactory");
// frameworks/av/media/libaudiohal/impl/EffectsFactoryHalHidl.cpp
--------------> return service ? new effect::CPP_VERSION::EffectsFactoryHalHidl(service) : nullptr;
----------> TimeCheck::setAudioHalPids(halPids);
--------> AudioFlinger::onFirstRef();
----------> property_get("ro.audio.flinger_standbytime_ms", val_str, NULL);
----------> mStandbyTimeInNsecs = milliseconds(int_val);
----------> mStandbyTimeInNsecs = kDefaultStandbyTimeInNsecs;
----------> mDevicesFactoryHalCallback = new DevicesFactoryHalCallbackImpl;
----------> mDevicesFactoryHal->setCallbackOnce(mDevicesFactoryHalCallback);
```

---

## AudioPolicyService

## 概述
`AudioPolicyService` 是 Android 音频系统的核心组件之一，负责音频策略的管理。它是 `AudioServer` 的一部分，处理音频设备的选择、音频路由、音量控制、音频焦点管理等任务。

## 主要功能
1. **音频设备选择**：根据当前场景（如通话、媒体播放、导航）选择合适的音频设备。
2. **音频路由**：动态调整音频路由，确保音频数据正确输出到目标设备。
3. **音量控制**：管理系统音量、应用音量以及音量曲线。
4. **音频焦点管理**：处理多个应用之间的音频焦点竞争（如音乐播放和导航提示音）。
5. **音频策略管理**：根据设备状态（如插入耳机、连接蓝牙）调整音频策略。

## 源码位置

> frameworks/av/services/audiopolicy/AudioPolicyService.h
> frameworks/av/services/audiopolicy/AudioPolicyService.cpp

## 源码流程

### 1. **初始化**
`AudioPolicyService` 在 `AudioServer` 启动时初始化。

主要流程如下：

```c++
--> AudioPolicyService::instantiate()
----> new AudioPolicyService()
----> AudioPolicyService::onFirstRef()
------> mAudioCommandThread = new AudioCommandThread(String8("ApmAudio"), this);
------> mOutputCommandThread = new AudioCommandThread(String8("ApmOutput"), this);
------> mAudioPolicyClient = new AudioPolicyClient(this);
------> mAudioPolicyManager = createAudioPolicyManager(mAudioPolicyClient);
--------> AudioPolicyManager *apm = new AudioPolicyManager(clientInterface);
----------> loadConfig();
--------> apm->initialize();
---------> mEngine = engLib->createEngine();
---------> onNewAudioModulesAvailableInt(nullptr /*newDevices*/);
---------> property_set("log.tag." LOG_TAG, "D");
---------> updateDevicesAndOutputs();
------> mAudioPolicyEffects = new AudioPolicyEffects();
------> mSensorPrivacyPolicy = new SensorPrivacyPolicy(this);
```

### 初始化流程详细介绍

#### 配置文件获取流程

获取配置文件`audio_policy_configuration.xml`路径,会依次查找.路径如下:

* `/odm/etc`
* `/vendor/etc/audio`
* `/vendor/etc`
* `/system/etc`

配置文件中默认会有四个module,具体如下:

* `<module name="primary" halVersion="3.0">`
* `<module name="a2dp" halVersion="2.0">`
* `<module name="usb" halVersion="2.0">`
* `<module name="r_submix" halVersion="2.0">`

```c++
// frameworks/av/services/audiopolicy/managerdefault/AudioPolicyManager.cpp
--> AudioPolicyManager *apm = new AudioPolicyManager(clientInterface);
----> mpClientInterface(clientInterface),
----> AudioPolicyConfig mConfig(mHwModulesAll, mOutputDevicesAll, mInputDevicesAll, mDefaultOutputDevice),
----> loadConfig();
------> deserializeAudioPolicyXmlConfig(AudioPolicyConfig &config);
--------> fileNames.push_back("audio_policy_configuration.xml");
--------> audio_get_configuration_paths();
----------> return std::vector<std::string>({"/odm/etc", "/vendor/etc/audio","/vendor/etc", "/system/etc"});
--------> deserializeAudioPolicyFile(audioPolicyXmlConfigFile, &config);
```

#### 配置文件解析流程

```c++
--> deserializeAudioPolicyFile(audioPolicyXmlConfigFile, &config);
----> PolicySerializer::deserialize(fileName, config);
------> auto doc = make_xmlUnique(xmlParseFile(configFile));
------> ModuleTraits::Collection modules;
------> deserializeCollection<ModuleTraits>(root, &modules, config);
--------> ModuleTraits::deserialize(const xmlNode *cur, PtrSerializingCtx ctx);
--------> Element module = new HwModule(name.c_str(), versionMajor, versionMinor);
--------> MixPortTraits::Collection mixPorts;
--------> status_t status = deserializeCollection<MixPortTraits>(cur, &mixPorts, NULL);
--------> module->setProfiles(mixPorts);
--------> DevicePortTraits::Collection devicePorts;
--------> status = deserializeCollection<DevicePortTraits>(cur, &devicePorts, NULL);
--------> module->setDeclaredDevices(devicePorts);
--------> RouteTraits::Collection routes;
--------> status = deserializeCollection<RouteTraits>(cur, &routes, module.get());
--------> module->setRoutes(routes);
--------> sp<DeviceDescriptor> device = module->getDeclaredDevices().getDeviceFromTagName(std::string(attachedDevice.get()));
--------> ctx->addDevice(device);
--------> ctx->setDefaultOutputDevice(device);
------> config->setHwModules(modules);
------> GlobalConfigTraits::deserialize(root, config);
------> SurroundSoundTraits::deserialize(root, config);
--> config.setSource(audioPolicyXmlConfigFile);
```

#### Engine加载

```c++
--> apm->initialize();
----> // libaudiopolicyengineconfigurable.so
----> auto engLib = EngineLibrary::load("libaudiopolicyengine" + getConfig().getEngineLibraryNameSuffix() + ".so");
------> mLibraryHandle = dlopen(libraryPath.c_str(), 0);
------> mCreateEngineInstance = (EngineInterface* (*)())dlsym(mLibraryHandle, "createEngineInstance");
----> mEngine = engLib->createEngine();
------> EngineInstance(mCreateEngineInstance(),[lib = shared_from_this(), destroy = mDestroyEngineInstance] (EngineInterface* e) {destroy(e);});
// frameworks/av/services/audiopolicy/engineconfigurable/src/EngineInstance.cpp
------> extern "C" EngineInterface* createEngineInstance();
--------> audio_policy::EngineInstance::getInstance()->queryInterface<EngineInterface>();
----------> EngineInterface *EngineInstance::queryInterface();
------------> EngineInterface *Engine::queryInterface();
----> mEngine->setObserver(this);
```

#### module加载

这部分负责遍历配置文件解析出来的`Module`，然后通过`IBinder`调用到`AudioFlinger`里面，，.下面有几个重要的点：

* `mHwModulesAll`的赋值，是在 `AudioPolicyManager` 构造函数时作为引用传给了`AudioPolicyConfig mConfig`,在解析完时通过`config->setHwModules(modules);`赋值的。
* ` mDevicesFactoryHal->openDevice(name, &dev);` 这个调用是调用到了音频HAL的 `adev_open` 函数, 并创建了 `AudioHwDevice`。
* `outHwDev->openOutputStream()` 这个调用是调用的音频HAL的 `adev_open_output_stream` 函数, 并创建了播放线程`PlaybackThread`，然后在线程里面创建了音频混音器`AudioMixer`和输出槽`AudioStreamOutSink`.
* `inHwHal->openOutputStream()` 这个调用是调用的音频HAL的 `adev_open_input_stream` 函数, 并创建了采集线程`RecordThread`,然后在线程里面创建了输入源`AudioStreamInSource`。

```c++
--> apm->initialize();
----> onNewAudioModulesAvailableInt(nullptr /*newDevices*/);
------> for (const auto& hwModule : mHwModulesAll) {
--------> mpClientInterface->loadHwModule(hwModule->getName())
----------> sp<IAudioFlinger> af = AudioSystem::get_audio_flinger();
----------> af->loadHwModule(name);
------------> AudioFlinger::loadHwModule(const char *name);
------------> loadHwModule_l(name);
--------------> sp<DeviceHalInterface> dev;
--------------> mDevicesFactoryHal->openDevice(name, &dev);
--------------> audio_module_handle_t handle = (audio_module_handle_t) nextUniqueId(AUDIO_UNIQUE_ID_USE_MODULE);
--------------> AudioHwDevice *audioDevice = new AudioHwDevice(handle, name, dev, flags);
--------------> mAudioHwDevs.add(handle, audioDevice);
--------------> return handle;
--------> hwModule->setHandle(handle);
--------> mHwModules.push_back(hwModule);
--------> for (const auto& outProfile : hwModule->getOutputProfiles()) {
----------> sp<SwAudioOutputDescriptor> outputDesc = new SwAudioOutputDescriptor(outProfile,mpClientInterface);
----------> audio_io_handle_t output = AUDIO_IO_HANDLE_NONE;
----------> outputDesc->open(nullptr, DeviceVector(supportedDevice),AUDIO_STREAM_DEFAULT,AUDIO_OUTPUT_FLAG_NONE, &output);
------------> mClientInterface->openOutput(mProfile->getModuleHandle(),output,&lConfig,device,&mLatency,mFlags);
--------------> AudioFlinger::openOutput(...);
--------------> AudioHwDevice *outHwDev = findSuitableHwDev_l(module, deviceType);
--------------> outHwDev->openOutputStream(&outputStream,*output,deviceType,flags,config,address.string());
--------------> sp<PlaybackThread> thread;
--------------> // MmapPlaybackThread OffloadThread DirectOutputThread
--------------> thread = new MixerThread(this, outputStream, *output, mSystemReady);
----------------> AudioFlinger::MixerThread::MixerThread();
----------------> mAudioMixer = new AudioMixer(mNormalFrameCount, mSampleRate);
----------------> mOutputSink = new AudioStreamOutSink(output->stream);
--------------> mPlaybackThreads.add(*output, thread);
--------------> mPatchPanel.notifyStreamOpened(outHwDev, *output);
----------> addOutput(output, outputDesc);
------------> mOutputs.add(output, outputDesc);
------------> applyStreamVolumes(outputDesc, DeviceTypeSet(), 0 /* delayMs */, true /* force */);
------------> updateMono(output); // update mono status when adding to output list
------------> selectOutputForMusicEffects();
--------------> mEffects.moveEffects(AUDIO_SESSION_OUTPUT_MIX, mMusicEffectOutput, output);
--------------> mpClientInterface->moveEffects(AUDIO_SESSION_OUTPUT_MIX, mMusicEffectOutput, output);
----------------> AudioFlinger::moveEffects(audio_session_t sessionId, audio_io_handle_t srcOutput,audio_io_handle_t dstOutput);
------------------> PlaybackThread *srcThread = checkPlaybackThread_l(srcOutput);
------------------> PlaybackThread *dstThread = checkPlaybackThread_l(dstOutput);
------------------> moveEffectChain_l(sessionId, srcThread, dstThread);
--------------> mMusicEffectOutput = output;
------------> nextAudioPortGeneration();
----------> setOutputDevices(outputDesc,DeviceVector(supportedDevice),true,0,NULL);
------------> PatchBuilder patchBuilder;
------------> patchBuilder.addSource(outputDesc);
------------> installPatch(__func__, patchHandle, outputDesc.get(), patchBuilder.patch(),muteWaitMs == 0 ? (delayMs + (outputDesc->latency() / 2)) : delayMs);
------------> applyStreamVolumes(outputDesc, filteredDevices.types(), delayMs);
--------> }
--------> for (const auto& inProfile : hwModule->getInputProfiles()) {
----------> sp<AudioInputDescriptor> inputDesc =new AudioInputDescriptor(inProfile, mpClientInterface);
----------> audio_io_handle_t input = AUDIO_IO_HANDLE_NONE;
----------> inputDesc->open(nullptr,availProfileDevices.itemAt(0),AUDIO_SOURCE_MIC,AUDIO_INPUT_FLAG_NONE,&input);
--------------> AudioFlinger::openInput(...);
--------------> AudioHwDevice *inHwDev = findSuitableHwDev_l(module, deviceType);
--------------> sp<DeviceHalInterface> inHwHal = inHwDev->hwDevice();
--------------> inHwHal->openInputStream(*input, devices, &halconfig, flags, address.string(), source,outputDevice, outputDeviceAddress, &inStream);
--------------> AudioStreamIn *inputStream = new AudioStreamIn(inHwDev, inStream, flags);
--------------> sp<RecordThread> thread = new RecordThread(this, inputStream, *input, mSystemReady);
----------------> mInputSource = new AudioStreamInSource(input->stream);
--------------> mRecordThreads.add(*input, thread);
--------> }
------> }
```

---
