---
title: Android音频之AudioTrack简析
tags: Android Audio AudioTrack
---

## **Android音频之AudioTrack简析**

`AudioTrack` 是 Android 系统中用于音频播放的一个核心类，它提供了对音频数据的高效播放，特别适用于播放原始 PCM 数据。与 `MediaPlayer` 相比，`AudioTrack` 更加灵活，能够提供低延迟的音频播放，广泛应用于实时音频播放、音效处理以及游戏开发等场景。

### **1. `AudioTrack` 的作用与功能**

`AudioTrack` 用于播放 PCM（脉冲编码调制）格式的音频数据。它直接将音频数据从内存写入音频硬件设备，并通过缓冲区将音频流送出。其核心功能包括：

- **音频数据播放**：可以播放从应用程序传输过来的 PCM 音频数据。
- **低延迟音频播放**：适合需要实时处理音频的场景，比如语音通信、游戏音效等。
- **音频流管理**：提供了对多个音频流的管理，包括播放、停止、暂停等。

### **2. `AudioTrack` 的工作模式**

`AudioTrack` 支持两种工作模式：`MODE_STREAM` 和 `MODE_STATIC`。

- **`MODE_STREAM`**：适用于动态、流式的音频播放。数据可以逐步写入缓冲区并播放，常用于长时间播放的音频（例如音乐、语音通话等）。
- **`MODE_STATIC`**：适用于短小的音频文件播放，如音效。所有数据在播放前一次性加载到缓冲区。

### **3. `AudioTrack` 的主要方法**

- **`play()`**：开始播放音频。
- **`stop()`**：停止播放音频。
- **`write()`**：将音频数据写入缓冲区并播放。
- **`flush()`**：清空缓冲区。
- **`release()`**：释放 `AudioTrack` 实例资源。

### **4. `AudioTrack` 的构造**

创建 `AudioTrack` 时，常用构造函数如下：
```java
AudioTrack(int streamType, int sampleRateInHz, int channelConfig, int audioFormat, int bufferSizeInBytes, int mode)
```

- `streamType`: 音频流类型，如 `AudioManager.STREAM_MUSIC`。
- `sampleRateInHz`: 音频采样率，常见值如 44100Hz 或 48000Hz。
- `channelConfig`: 音频通道配置，如单声道或立体声。
- `audioFormat`: 音频格式，如 `AudioFormat.ENCODING_PCM_16BIT`。
- `bufferSizeInBytes`: 缓冲区大小，决定了音频播放的流畅性。
- `mode`: 工作模式，`MODE_STREAM` 或 `MODE_STATIC`。

### **5. 使用示例**

#### **播放简单的 PCM 数据**

```java
import android.media.AudioFormat;
import android.media.AudioManager;
import android.media.AudioTrack;

public class AudioTrackExample {

    public void playAudio() {
        // 设置音频参数
        int sampleRate = 44100; // 采样率
        int channelConfig = AudioFormat.CHANNEL_OUT_MONO; // 单声道
        int audioFormat = AudioFormat.ENCODING_PCM_16BIT; // PCM 16 位
        int bufferSize = AudioTrack.getMinBufferSize(sampleRate, channelConfig, audioFormat);

        // 创建 AudioTrack 实例
        AudioTrack audioTrack = new AudioTrack(
            AudioManager.STREAM_MUSIC,  // 音频流类型
            sampleRate,                 // 采样率
            channelConfig,              // 通道配置
            audioFormat,                // 音频格式
            bufferSize,                 // 缓冲区大小
            AudioTrack.MODE_STREAM      // 流式播放
        );

        // 启动音频播放
        audioTrack.play();

        // 播放音频数据
        byte[] audioData = generateSampleData();  // 生成 PCM 数据
        audioTrack.write(audioData, 0, audioData.length);
    }

    // 生成简单的音频数据（如正弦波）
    private byte[] generateSampleData() {
        int duration = 3; // 持续时间 3 秒
        int numSamples = duration * 44100; // 采样数
        byte[] buffer = new byte[numSamples * 2]; // 每个样本 2 字节（16 位 PCM）

        double frequency = 440.0; // 440Hz，A4 音符
        for (int i = 0; i < numSamples; i++) {
            short sample = (short) (Math.sin(2 * Math.PI * frequency * i / 44100) * Short.MAX_VALUE);
            buffer[i * 2] = (byte) (sample & 0xFF); // 低位字节
            buffer[i * 2 + 1] = (byte) ((sample >> 8) & 0xFF); // 高位字节
        }
        return buffer;
    }
}
```

#### **代码解释：**
1. **创建 `AudioTrack` 实例**：定义音频参数（如采样率、通道配置、音频格式）并创建 `AudioTrack` 对象。选择 `MODE_STREAM` 模式用于动态播放数据。
2. **生成音频数据**：使用正弦波公式生成 PCM 音频数据（440Hz A4 音符）。
3. **播放音频**：调用 `audioTrack.play()` 启动播放，并使用 `audioTrack.write()` 写入音频数据。

---

### **6. 适用场景**

- **音效播放**：适用于游戏中的短音效播放（如按钮点击音、得分音效等），尤其在 `MODE_STATIC` 模式下。
- **实时音频播放**：适用于需要实时音频流的应用（如语音通话、在线广播、音乐流媒体等）。
- **音频处理**：用于开发音频处理应用程序，如音频合成、滤波、效果处理等。

---

### **7. 源码流程**

```java
//frameworks/base/media/java/android/media/AudioTrack.java
--> AudioTrack(AudioAttributes, AudioFormat, int,int, int, boolean, int,TunerConfiguration)
----> mConfiguredAudioAttributes = attributes;
----> Looper looper;
----> int channelIndexMask = 0;
----> channelIndexMask = format.getChannelIndexMask();
----> int channelMask = 0;
----> channelMask = format.getChannelMask();
----> int encoding = AudioFormat.ENCODING_DEFAULT;
----> encoding = format.getEncoding();
----> mOffloaded = offload;
----> mStreamType = AudioSystem.STREAM_DEFAULT;
----> audioBuffSizeCheck(bufferSizeInBytes);
----> mInitializationLooper = looper;
----> int[] sampleRate = new int[] {mSampleRate};
----> int[] session = new int[1];
----> session[0] = sessionId;
----> // native initialization
----> int initResult = native_setup(new WeakReference<AudioTrack>(this), mAttributes,
---->         sampleRate, mChannelMask, mChannelIndexMask, mAudioFormat,
---->         mNativeBufferSizeInBytes, mDataLoadMode, session, 0 /*nativeTrackInJavaObj*/,
---->         offload, encapsulationMode, tunerConfiguration,
---->         getCurrentOpPackageName());
----> // frameworks/base/core/jni/android_media_AudioTrack.cpp
----> // {"native_setup","(Ljava/lang/Object;Ljava/lang/Object;[IIIIII[IJZILjava/lang/Object;Ljava/lang/String;)I",
----> //            (void *)android_media_AudioTrack_setup}
----> mSampleRate = sampleRate[0];
----> mSessionId = session[0];
```

```c++
// frameworks/base/core/jni/android_media_AudioTrack.cpp
--> android_media_AudioTrack_setup(...)
----> sp<AudioTrack> lpTrack;
----> audio_channel_mask_t nativeChannelMask = nativeChannelMaskFromJavaChannelMasks(channelPositionMask, channelIndexMask);
----> uint32_t channelCount = audio_channel_count_from_out_mask(nativeChannelMask);
----> audio_format_t format = audioFormatToNative(audioFormat);
----> lpTrack = new AudioTrack(opPackageNameStr.c_str());
// frameworks/av/media/libaudioclient/AudioTrack.cpp
------> AudioTrack::AudioTrack(const std::string& opPackageName);
----> status = lpTrack->set(AUDIO_STREAM_DEFAULT...);
------> AudioTrack::set(audio_stream_type_t streamType...);
--------> createTrack_l();
----------> const sp<IAudioFlinger>& audioFlinger = AudioSystem::get_audio_flinger();
----------> IAudioFlinger::CreateTrackInput input;
----------> IAudioFlinger::CreateTrackOutput output;
----------> sp<IAudioTrack> track = audioFlinger->createTrack(input,output,&status);
----------> sp<IMemory> iMem = track->getCblk();
----------> void *iMemPointer = iMem->unsecurePointer();
----------> mAudioTrack = track;
----------> mCblkMemory = iMem;
----------> IPCThreadState::self()->flushCommands();
----------> audio_track_cblk_t* cblk = static_cast<audio_track_cblk_t*>(iMemPointer);
----------> mCblk = cblk;
----------> void* buffers;
----------> if (mSharedBuffer == 0) { buffers = cblk + 1;}
----------> else { buffers = mSharedBuffer->unsecurePointer();}
----------> mAudioTrack->attachAuxEffect(mAuxEffectId);
----------> if (mSharedBuffer == 0) {
---------->     mStaticProxy.clear();
---------->     mProxy = new AudioTrackClientProxy(cblk, buffers, mFrameCount, mFrameSize);
----------> } else {
---------->     mStaticProxy = new StaticAudioTrackClientProxy(cblk, buffers, mFrameCount, mFrameSize);
---------->     mProxy = mStaticProxy;
----------> }
```

```c++
// frameworks/av/services/audioflinger/AudioFlinger.cpp
sp<IAudioTrack> AudioFlinger::createTrack(const CreateTrackInput& input,CreateTrackOutput& output,status_t *status)
--> sp<PlaybackThread::Track> track;
--> sp<TrackHandle> trackHandle;
--> output.sessionId = sessionId;
--> output.outputId = AUDIO_IO_HANDLE_NONE;
--> output.selectedDeviceId = input.selectedDeviceId;
--> lStatus = AudioSystem::getOutputForAttr(&localAttr, &output.outputId, sessionId, &streamType,
-->                                         clientPid, clientUid, &input.config, input.flags,
-->                                         &output.selectedDeviceId, &portId, &secondaryOutputs);
----> status_t AudioSystem::getOutputForAttr(audio_attributes_t *attr..);
----> const sp<IAudioPolicyService>& aps = AudioSystem::get_audio_policy_service();
----> return aps->getOutputForAttr(attr, output, session, stream, pid, uid,config,flags, selectedDeviceId, portId, secondaryOutputs);
--> PlaybackThread *thread = checkPlaybackThread_l(output.outputId);
--> output.sampleRate = input.config.sample_rate;
--> output.frameCount = input.frameCount;
--> output.notificationFrameCount = input.notificationFrameCount;
--> output.flags = input.flags;
-->
--> track = thread->createTrack_l(client, streamType, localAttr, &output.sampleRate,
-->                               input.config.format, input.config.channel_mask,
-->                               &output.frameCount, &output.notificationFrameCount,
-->                               input.notificationsPerBuffer, input.speed,
-->                               input.sharedBuffer, sessionId, &output.flags,
-->                               callingPid, input.clientInfo.clientTid, clientUid,
-->                               &lStatus, portId, input.audioTrackCallback,
-->                               input.opPackageName);
--> trackHandle = new TrackHandle(track);
--> return trackHandle;
```