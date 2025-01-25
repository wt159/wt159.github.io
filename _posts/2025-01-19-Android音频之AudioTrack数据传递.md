---
title: Android音频之AudioTrack数据传递
tags: Android Audio AudioTrack audio_track_cblk_t
---

## **Android音频之AudioTrack数据传递**
# Android音频之AudioTrack数据传递

## 概述

Android中的AudioTrack是一个用于设备声音播放的基础类。它通过与AudioFlinger互动，将应用端数据传送到音频硬件进行播放。本文将从数据传递的角度，解析AudioTrack如何实现数据之间的流转。

## 主要组件

### AudioTrackClientProxy

`AudioTrackClientProxy`是用户端用于与服务端通信的软件代理。它通过共享内存区，将应用端的音频数据上传到AudioFlinger。主要函数包括：

- `obtainBuffer()`: 获取一块可用的内存空间，以写入音频数据。
- `releaseBuffer()`: 释放内存块，提示数据已经准备好。

### AudioTrackServerProxy

`AudioTrackServerProxy`是服务端用于接收音频数据的代理。它和`AudioTrackClientProxy`共享同一块内存，通过读取已写入的数据，将它播放到硬件上。主要函数包括：

- `obtainBuffer()`: 获取已写入数据的内存块，以准备播放。
- `releaseBuffer()`: 提示内存块已经播放完毕，为新数据清理空间。

## AudioTrack数据传递流程

1. **audioserver**

```c++
AudioFlinger::PlaybackThread::threadLoop();
--> for (int64_t loopCount = 0; !exitPending(); ++loopCount) {
----> processConfigEvents_l();
----> saveOutputTracks();
----> if ((mActiveTracks.isEmpty() && systemTime() > mStandbyTimeNs) || isSuspended()) {
---->     if (shouldStandby_l()) {threadLoop_standby();}
---->     if (mActiveTracks.isEmpty() && mConfigEvents.isEmpty()) {
---->         clearOutputTracks();
---->         mWaitWorkCV.wait(mLock); //阻塞等待
---->     }
----> }
----> mMixerStatus = prepareTracks_l(&tracksToRemove);
------> audio_track_cblk_t* cblk = track->cblk();
------> const int trackId = track->id();
------> mAudioMixer->create(trackId,track->mChannelMask,track->mFormat,track->mSessionId);
------> size_t desiredFrames;
------> const uint32_t sampleRate = track->mAudioTrackServerProxy->getSampleRate();
------> AudioPlaybackRate playbackRate = track->mAudioTrackServerProxy->getPlaybackRate();
------> size_t framesReady = track->framesReady();
------> if ((framesReady >= minFrames) && track->isReady() &&!track->isPaused() && !track->isTerminated()){
------>     mixedTracks++;
------>     if (track->mainBuffer() != mSinkBuffer && track->mainBuffer() != mMixerBuffer) {
------>         if (mEffectBufferEnabled) {mEffectBufferValid = true; /* Later can set directly.*/}
------>         chain = getEffectChain_l(track->sessionId());
------>         if (chain != 0) {tracksWithEffect++;}
------>     }
------>     track->setFinalVolume((vrf + vlf) / 2.f);
------>         mAudioMixer->setBufferProvider(trackId, track);
------>     mAudioMixer->enable(trackId);
------>         const std::shared_ptr<TrackBase> &track = mTracks[name];
------>         if (!track->enabled) {
------>             track->enabled = true;
------>             invalidate();
------>         }
------>     mAudioMixer->setParameter(trackId, param, AudioMixer::VOLUME0, &vlf);
------>     mAudioMixer->setParameter(trackId, param, AudioMixer::VOLUME1, &vrf);
------>     mAudioMixer->setParameter(trackId, param, AudioMixer::AUXLEVEL, &vaf);
------>     mAudioMixer->setParameter(trackId,AudioMixer::TRACK,AudioMixer::FORMAT, (void *)track->format());
------>     mAudioMixer->setParameter(trackId,AudioMixer::TRACK,AudioMixer::CHANNEL_MASK, (void *)(uintptr_t)track->channelMask());
------>     mAudioMixer->setParameter(trackId,AudioMixer::TRACK,AudioMixer::MIXER_CHANNEL_MASK,(void *)(uintptr_t)------> (mChannelMask | mHapticChannelMask));
------>     if (mMixerBufferEnabled && (track->mainBuffer() == mSinkBuffer || track->mainBuffer() == mMixerBuffer)) {
------>         mAudioMixer->setParameter(trackId,AudioMixer::TRACK,AudioMixer::MIXER_FORMAT, (void *)mMixerBufferFormat);
------>         mAudioMixer->setParameter(trackId,AudioMixer::TRACK,AudioMixer::MAIN_BUFFER, (void *)mMixerBuffer);
------>         mMixerBufferValid = true;
------>     }
------>     track->mRetryCount = kMaxTrackRetries;
------>     uint32_t maxSampleRate = mSampleRate * AUDIO_RESAMPLER_DOWN_RATIO_MAX;
------>     uint32_t reqSampleRate = proxy->getSampleRate();
------>     if (reqSampleRate == 0) {reqSampleRate = mSampleRate;}
------>     else if (reqSampleRate > maxSampleRate) {reqSampleRate = maxSampleRate; }
------>     mAudioMixer->setParameter(trackId,AudioMixer::RESAMPLE,AudioMixer::SAMPLE_RATE,(void *)(uintptr_t)reqSampleRate);
------>     AudioPlaybackRate playbackRate = proxy->getPlaybackRate();
------> } else {
------> 
------> }
----> mActiveTracks.updatePowerState(this);
----> activeTracks.insert(activeTracks.end(), mActiveTracks.begin(), mActiveTracks.end());
-> }
-> AudioFlinger::PlaybackThread::threadLoop_write();
---> ssize_t framesWritten = mNormalSink->write((char *)mSinkBuffer + offset, count);
---> bytesWritten = framesWritten * mFrameSize;
---> mNumWrites++;
---> mInWrite = false;
---> if (mStandby) { mStandby = false; }
---> return bytesWritten;
```

1. **初始化**
   - 应用端创建`AudioTrack`实例，并通过JNI连接到AudioFlinger。
   - AudioFlinger创建对应的音频通道，并创建共享内存。

2. **数据写入和读取**
   - 应用端通过`AudioTrackClientProxy`将音频数据写入共享内存。
   - AudioFlinger通过`AudioTrackServerProxy`读取数据，将它传送到硬件。

3. **播放过程中的同步**
   - `AudioTrackClientProxy`和`AudioTrackServerProxy`通过互相调用`obtainBuffer`和`releaseBuffer`，确保数据不会重叠或者丢失。

## 示例代码

以下代码例给出了如何通过`AudioTrackClientProxy`和`AudioTrackServerProxy`进行数据传递。

```cpp
#include <iostream>
#include <media/AudioTrackShared.h>
#include <memory>
#include <utils/Log.h>
#include <thread>
#include <unistd.h>
#include <cassert>

using namespace android;
#define TAG "AudioTrackProxy"
constexpr size_t mFrameCount = 960; // 缓冲区帧数
constexpr size_t kFrameCount = mFrameCount * 10; // 缓冲区帧数
constexpr size_t mFrameSize = sizeof(int16_t); // 每帧大小
int kMaxNum = 4;

class AudioTrackServerProxyDerived : public AudioTrackServerProxy {
public:
    AudioTrackServerProxyDerived(audio_track_cblk_t* cblk, void *buffers, size_t frameCount,
            size_t frameSize, bool clientInServer, uint32_t sampleRate) : AudioTrackServerProxy( cblk, buffers, frameCount,
             frameSize,  clientInServer,  sampleRate) {}
    // 提供公共析构函数
    ~AudioTrackServerProxyDerived() override {
        // 可选：自定义清理代码
    }
};

void* producer(std::shared_ptr<AudioTrackClientProxy> proxy, int16_t *sineWave, size_t sineSize)
{
        // Client 写入数据
        (void)sineSize;
        ALOGI("Client writing data...");
        char *sinePtr = (char*)sineWave;
        size_t userSize = mFrameCount * mFrameSize;
        size_t toWrite = 0;
        size_t written = 0;
        size_t num = 0;
        Proxy::Buffer buffer;
        status_t status = NO_ERROR;
        
        while(num++ < kMaxNum) {
            userSize = mFrameCount * mFrameSize;
            written = 0;
            while(userSize >= mFrameSize) {
                buffer.mFrameCount = userSize / mFrameSize;
                status = proxy->obtainBuffer(&buffer, &ClientProxy::kNonBlocking, NULL);
                if (status < 0) {
                    ALOGE("obtainBuffer failed, ret:%d", status);
                    continue;
                }
                ALOGD("Client obtain buffer %zu frames", buffer.mFrameCount);
                toWrite = buffer.mFrameCount * mFrameSize;
                // buffer.mRaw = sinePtr + written;
                memcpy(buffer.mRaw, sinePtr + written, toWrite);
                userSize -= toWrite;
                written += toWrite;
                proxy->releaseBuffer(&buffer);
                ALOGI("Client wrote %zu bytes. userSize:%zu, written:%zu, num:%zu", toWrite, userSize, written, num);
            }
            // usleep(5000);
        }
        return NULL;
}
void *consumer(std::shared_ptr<AudioTrackServerProxyDerived> proxy, int16_t *sineWave, size_t sineSize)
{
    // Server 读取数据
    (void)sineSize;
    int16_t readBuffer[mFrameCount*5] = { 0 };
    char * bufPtr = (char*)readBuffer;
    size_t toRead = 0;
    size_t reads = mFrameCount *mFrameSize;
    size_t num = 0;
    ALOGI("Server reading data...");
    while(num < kMaxNum) {
        // usleep(5000);
        
        size_t read = proxy->framesReadySafe();
        if (read < mFrameCount) {
            ALOGE("Server ready %zu frames.", read);
            continue;
        } else {
            ALOGI("Server ready %zu frames.", read);
        }

        ServerProxy::Buffer buf;
        memset(readBuffer, 0, mFrameCount*mFrameSize);
        toRead = 0;
        reads = mFrameCount *mFrameSize;
        while(reads >= mFrameSize) {
            buf.mFrameCount = reads / mFrameSize;
            status_t status = proxy->obtainBuffer(&buf);
            if (status == NO_ERROR) {
                toRead = buf.mFrameCount*mFrameSize;
                memcpy(bufPtr + (mFrameCount *mFrameSize) - reads, buf.mRaw, toRead);
                reads -= toRead;
                ALOGD("read buffer suc. count:%zu, size:%zu, toRead:%zu, reads:%zu, num:%zu", buf.mFrameCount, mFrameSize, toRead, reads, num);
                proxy->releaseBuffer(&buf);
            } else {
                ALOGE("obtain buffer failed,  ret:%d", status);
            }
            
        }
        num++;
        // 验证数据一致性
        bool dataMatches = true;
        for (size_t i = 0; i < mFrameCount; ++i) {
            if (readBuffer[i] != sineWave[i]) {
                dataMatches = false;
                break;
            }
        }

        if (dataMatches) {
            std::cout << "Data matches between Server and Client!" << std::endl;
        } else {
            std::cerr << "Data mismatch!" << std::endl;
            abort();
        }
    }
    return NULL;
}

int main(int argc, char *argv[])
{
    if (argc > 1) {
        kMaxNum = atoi(argv[1]);
    }
    std::cout << "kMaxNum: " << kMaxNum << std::endl;
    // 分配共享内存，用于 Server 和 Client 之间的数据交互
    size_t sharedBufferSize = kFrameCount * mFrameSize + sizeof(audio_track_cblk_t);
    void* sharedMemory = malloc(sharedBufferSize);

    if (!sharedMemory) {
        std::cerr << "Failed to allocate shared memory" << std::endl;
        return -1;
    }

    // 初始化 audio_track_cblk_t（控制块）
    audio_track_cblk_t* cblk = new (sharedMemory) audio_track_cblk_t();

    // 缓冲区
    void* buffers = cblk + 1;
    // 创建 Server 和 Client Proxy
    std::shared_ptr<AudioTrackServerProxyDerived> serverProxy(new AudioTrackServerProxyDerived(cblk, buffers, kFrameCount, mFrameSize, false, 48000));
    std::shared_ptr<AudioTrackClientProxy> clientProxy(new AudioTrackClientProxy(cblk, buffers, kFrameCount, mFrameSize));

    // 示例数据：生成一个简单的正弦波数据
    int16_t sineWave[mFrameCount];
    for (size_t i = 0; i < mFrameCount; ++i) {
        sineWave[i] = static_cast<int16_t>(32767 * sin(2 * M_PI * i / mFrameCount));
    }

    // start
    {
        serverProxy->start();
        ServerProxy::Buffer buffer;
        buffer.mFrameCount = 1;
        (void)serverProxy->obtainBuffer(&buffer, true);
    }

    std::thread client(producer, clientProxy, sineWave, mFrameCount);
    std::thread server(consumer, serverProxy, sineWave, mFrameCount);

    client.join();
    server.join();

    // 清理
    free(sharedMemory);
    fgetc(stdin);
    return 0;
}

```

```shell
cc_binary {
    name: "audiotrack_proxy_example", // 编译生成的可执行文件名
    srcs: [
        "AudioTrackSharedExample.cpp", // 源文件
        "AudioTrackShared.cpp",
    ],
    include_dirs: [
	"frameworks/av/media/libavextensions",
        "frameworks/av/media/libnbaio/include_mono/",
        "frameworks/av/include/private",
        "system/media/audio_utils/include",
    ],
    local_include_dirs: [
        "include/media", "aidl"
    ],
    header_libs: [
        "libaudioclient_headers",
        "libbase_headers",
        "libmedia_headers",
    ],
    shared_libs: [
        "libaudioclient", // 依赖的库：用于 FIFO 缓冲区
        "libmedia",        // 依赖的库：用于 AudioTrackShared 相关操作
        "liblog",          // 日志库
        "libcutils",       // 常用的系统工具库
        "libutils",
        "libaudioutils",
    ],
    cflags: [
        "-std=c++17", // 使用 C++17 标准
        "-Wall",      // 开启所有警告
        "-DLOG_NDEBUG=0",
    ],
    stem: "AudioTrackProxyExample", // 指定生成的可执行文件名称
}
```
