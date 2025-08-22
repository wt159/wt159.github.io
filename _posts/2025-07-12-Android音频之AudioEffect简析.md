---
title: Android音频之AudioEffect简析
tags: Android AAOS Audio AudioEffect
---

`AudioEffect` 是 Android 音频框架中一个非常重要的组成部分，它提供了一套标准的 API，允许开发者对音频流（播放或录制）施加各种实时音效。

### 背景

* 平台: 高通8295
* 版本: Android 13
* 系统: AAOS

### 一、AudioEffect 是什么？

`AudioEffect` 是一个基类，它定义了一套用于控制音频效果的控制接口。这些效果可以附加到特定的音频轨道（如 `AudioTrack`, `AudioRecord`, `MediaPlayer` 等）上，实时地处理流过该轨道的音频数据。

它的核心思想是：**在音频数据从源（如应用、麦克风）到目的地（如扬声器、耳机）的传输路径上，插入一个处理节点**，这个节点就是音频效果器。

### 二、核心特点

1.  **实时处理**：效果处理是实时进行的，延迟极低，不会中断音频流。
2.  **链路集成**：效果器直接集成在 Android 音频流水线中，由底层原生代码处理，效率远高于在 Java 层处理 PCM 数据。
3.  **标准化接口**：为不同的音效（均衡器、重低音、虚拟化等）提供了统一的控制接口。
4.  **会话ID绑定**：效果器通过“音频会话 ID”（Audio Session ID）绑定到特定的音频流。所有具有相同会话 ID 的音频流会共享同一组效果器。

### 三、主要的子类（音效类型）

Android 提供了一系列内置的音效，它们都是 `AudioEffect` 的子类：

1.  **Equalizer (`Equalizer`)**
    *   **功能**：均衡器，用于调节不同频率段的增益（Gain）。可以提升或衰减低音、高音等。
    *   **核心方法**：
        *   `getNumberOfBands()`: 获取支持的频段数量。
        *   `getCenterFreq(band)`: 获取特定频段的中心频率。
        *   `setBandLevel(band, level)`: 设置特定频段的增益值。
        *   `getBandLevel(band)`: 获取特定频段的当前增益值。

2.  **BassBoost (`BassBoost`)**
    *   **功能**：重低音增强效果。
    *   **核心方法**：
        *   `setStrength(strength)`: 设置增强强度（0-1000）。
        *   `getStrength()`: 获取当前强度。

3.  **Virtualizer (`Virtualizer`)**
    *   **功能**：虚拟化效果，旨在通过耳机或扬声器提供更宽广的声场和环绕声体验。
    *   **核心方法**：
        *   `setStrength(strength)`: 设置虚拟化强度（0-1000）。
        *   `getStrength()`: 获取当前强度。

4.  **PresetReverb (`PresetReverb`)**
    *   **功能**：预设混响效果，模拟不同环境（如大厅、洞穴、舞台）的声学特性。
    *   **核心方法**：
        *   `setPreset(preset)`: 设置混响预设（如 `PRESET_LARGEROOM`）。
        *   `getPreset()`: 获取当前预设。

5.  **EnvironmentalReverb (`EnvironmentalReverb`) - (已弃用)**
    *   **功能**：更高级的环境混响，允许详细参数调节（房间大小、衰减、反射等），但已被更现代的架构取代。

6.  **Visualizer (`Visualizer`)**
    *   **功能**：**它不是效果器，而是一个分析器**。但它继承自 `AudioEffect`，因为它同样需要附加到音频流上。它用于获取音频流的波形或频域数据（FFT），从而实现可视化效果（如音乐播放器的频谱动画）。
    *   **核心方法**：
        *   `setDataCaptureListener(listener, rate, waveform, fft)`: 设置数据捕获监听器，定期回调波形或FFT数据。
        *   `getWaveForm(byte[] waveform)`: 获取波形数据（已弃用，推荐用监听器方式）。
        *   `getFft(byte[] fft)`: 获取FFT数据（已弃用，推荐用监听器方式）。

7.  **LoudnessEnhancer (`LoudnessEnhancer`)**
    *   **功能**：响度增强器，用于提高音频的整体响度，基于心理声学模型。

8.  **NoiseSuppressor (`NoiseSuppressor`) 和 AcousticEchoCanceller (`AcousticEchoCanceller`)**
    *   **功能**：这两个是用于**录音路径**的效果器。
    *   `NoiseSuppressor` (NS): 噪声抑制，用于消除背景噪声（如风扇声、键盘声）。
    *   `AcousticEchoCanceller` (AEC): 声学回声消除，用于消除扬声器播放的声音被麦克风再次捕获产生的回声（在语音通话中至关重要）。
    *   **注意**：这两个效果器**不一定在所有设备上都可用**，使用前必须通过 `isAvailable()` 方法进行检查。

### 四、工作流程（以 Equalizer 为例）

1.  **获取音频会话 ID**：首先，你需要一个音频流的目标会话 ID。如果你使用 `MediaPlayer`，可以通过 `mp.getAudioSessionId()` 获取。如果你自己创建 `AudioTrack`，可以在构造时指定一个非 0 的会话 ID，或者使用系统分配的。

2.  **检查可用性**（可选）：`Equalizer.getNumberOfEffects()` 可以查看设备上支持的均衡器总数。

3.  **创建效果器实例**：
    ```java
    int audioSessionId = mediaPlayer.getAudioSessionId();
    Equalizer equalizer = new Equalizer(0, audioSessionId); // 第一个0表示优先级
    equalizer.setEnabled(true); // 启用效果器
    ```

4.  **配置效果器参数**：
    ```java
    short numBands = equalizer.getNumberOfBands();
    short minLevel = equalizer.getBandLevelRange()[0]; // 最小增益值
    short maxLevel = equalizer.getBandLevelRange()[1]; // 最大增益值

    // 将第一个频段的增益设置为最大值的一半
    equalizer.setBandLevel((short)0, (short)((maxLevel - minLevel)/2));
    ```

5.  **释放资源**：当不再需要时，必须释放效果器。
    ```java
    equalizer.release();
    ```

### 五、重要概念和注意事项

1.  **音频会话 ID (Audio Session ID)**：
    *   这是效果器与音频流关联的**唯一标识符**。
    *   系统生成的音频流（如 `MediaPlayer`, `SoundPool`）会自动分配一个 ID。
    *   自己创建 `AudioTrack` 或 `AudioRecord` 时，如果传入 `0`，系统会分配一个新 ID；如果传入一个已存在的 ID，则会共享该音频上下文和效果链。

2.  **效果作用域**：
    *   附加到某个会话 ID 的效果器会影响**所有使用该会话 ID 的音频流**。这在混音时非常有用。
    *   如果你想为每个音频流单独控制效果，需要为它们创建不同的会话 ID。

3.  **可用性**：
    *   不是所有设备都支持所有音效。在使用前（尤其是 `NoiseSuppressor` 和 `AcousticEchoCanceller`），务必调用 `isAvailable()` 进行检查。
    *   可视化器 (`Visualizer`) 需要权限 `android.permission.RECORD_AUDIO`，因为它会“窃听”音频数据。

4.  **生命周期管理**：
    *   效果器的生命周期必须由应用管理。当关联的音频流被释放后，效果器不会自动释放，必须手动调用 `release()` 以避免资源泄漏。

5.  **延迟**：虽然延迟很低，但添加效果器仍然会引入少量延迟，在对延迟极其敏感的应用中需要考虑这一点。

### 六、应用场景

*   **音乐播放器**：均衡器、重低音、虚拟化、可视化频谱。
*   **视频播放器**：增强音频体验（如电影模式虚拟环绕声）。
*   **语音通话应用**：**AEC（回声消除）和 NS（降噪）是必备功能**，通常由系统在底层自动处理，但应用也可以通过此 API 进行控制或检查。
*   **音频录制应用**：在录制时添加混响、降噪等效果。
*   **游戏**：为游戏音效添加环境混响，增强沉浸感。

### 七、相关源码路径

frameworks/base/media/java/android/media/audiofx/AudioEffect.java
frameworks/base/media/jni/audioeffect/android_media_AudioEffect.cpp
frameworks/av/media/libaudioclient/AudioEffect.cpp
frameworks/av/services/audioflinger/AudioFlinger.cpp#3918
frameworks/av/media/libaudiohal/EffectsFactoryHalInterface.cpp
hardware/interfaces/audio/effect/all-versions/default/EffectsFactory.cpp
frameworks/av/media/libeffects/factory/EffectsFactory.c
frameworks/av/media/libeffects/factory/EffectsXmlConfigLoader.cpp
vendor/qcom/opensource/audio-hal-ar/primary-hal/configs/msmnile_au/audio_effects.xml
vendor/qcom/opensource/audio-hal-ar/primary-hal/audio-effects/post_proc/equalizer.c

### 八、示例

#### 应用示例

```java
package android.car.apitest.media;

import android.media.audiofx.AudioEffect;
import android.util.Log;
import java.util.UUID;

public class EffectFCEEqualizer extends AudioEffect {
    private static final String TAG = "EffectFCEEqualizer";
    public static final int PARAM_BAND_LEVEL = 0;
    public static final int PARAM_PROPERTIES = 1;
    public static final int PARAM_ALL = 2;
    public static final UUID EFFECT_TYPE_FCE_EQ_UUID = UUID.fromString("a0dac280-401c-11e3-9379-0002a5d5c51b");
    public static final UUID EFFECT_TYPE_FCE_EQ_TYPE = UUID.fromString("0bed4300-ddd6-11db-8f34-0002a5d5c51b");

    public EffectFCEEqualizer(int audioSession) throws IllegalStateException, IllegalArgumentException, UnsupportedOperationException, RuntimeException {
        super(EFFECT_TYPE_FCE_EQ_TYPE, EFFECT_TYPE_FCE_EQ_UUID, 0, audioSession);
        if (audioSession == 0) {
            Log.w("EffectFCEEqualizer", "WARNING: attaching a EffectFCEEqualizer to global output mix is deprecated!!");
        }
        this.setEnabled(true);
    }

    public EffectFCEEqualizer.Settings getAllLevels() throws IllegalStateException, IllegalArgumentException, UnsupportedOperationException {
        byte[] param = new byte[22];
        this.checkStatus(this.getParameter(2, param));
        EffectFCEEqualizer.Settings settings = new EffectFCEEqualizer.Settings();
        settings.curPreset = byteArrayToShort(param, 0);
        settings.bandLevels = new short[10];

        for(int i = 0; i < 10; ++i) {
            settings.bandLevels[i] = byteArrayToShort(param, 2 + 2 * i);
            Log.i("EffectFCEEqualizer", "getAllLevels, bandLevels[" + i + "] = " + settings.bandLevels[i]);
        }

        Log.i("EffectFCEEqualizer", "getAllLevels, curPreset = " + settings.curPreset);
        return settings;
    }

    public void setProperties(short preset_num) throws IllegalStateException, IllegalArgumentException, UnsupportedOperationException {
        int[] param = new int[1];
        short[] value = new short[1];
        param[0] = 1;
        value[0] = preset_num;
        Log.i("EffectFCEEqualizer", "setProperties, preset_num = " + preset_num);
        this.checkStatus(this.setParameter(param, value));
    }

    public void setBandLevel(short band, short level) throws IllegalStateException, IllegalArgumentException, UnsupportedOperationException {
        int[] param = new int[2];
        short[] value = new short[1];
        param[0] = 0;
        param[1] = band;
        value[0] = level;
        Log.i("EffectFCEEqualizer", "band = " + band + ", level = " + level);
        this.checkStatus(this.setParameter(param, value));
    }

    public short getBandLevel(short band) throws IllegalStateException, IllegalArgumentException, UnsupportedOperationException {
        Log.i("EffectFCEEqualizer", "band = " + band);
        short[] result = new short[1];
        EffectFCEEqualizer.Settings settings = this.getAllLevels();
        result[0] = settings.bandLevels[band];
        return result[0];
    }

    public void releaseEQ() throws IllegalStateException, IllegalArgumentException, UnsupportedOperationException {
        Log.w("EffectFCEEqualizer", "release EffectFCEEQ");
        this.release();
    }

    public static class Settings {
        public short curPreset;
        public short[] bandLevels = null;
    }
}
```

```java
private EffectFCEEqualizer mEqualizer = new EffectFCEEqualizer(0);
```