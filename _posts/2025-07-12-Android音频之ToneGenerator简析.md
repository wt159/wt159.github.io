---
title: Android音频之ToneGenerator简析
tags: Android Audio AudioTrack
---

`ToneGenerator` 是一个用于生成 DTMF（双音多频）信号和呼叫监督音（如忙音、回铃音）的实用工具类。它封装了音频信号生成的复杂细节，让开发者能够以简单的 API 来播放标准化的提示音。

### 一、ToneGenerator 是什么？

`ToneGenerator` 顾名思义，是一个“音调生成器”。它的主要目的是**程序化地生成特定的、标准化的音频 tones**，而不是播放预先录制好的音频文件（如 MP3、WAV）。

这些音调基于两个正弦波叠加的原理（DTMF），或者一个特定频率和节奏的正弦波（呼叫监督音）。它直接向音频输出设备（如扬声器、听筒）发送生成的 PCM 音频流。

### 二、核心特点

1.  **程序化生成**：音调是实时通过算法计算生成的，不依赖任何外部音频文件，因此应用体积更小，响应更即时。
2.  **标准化音调**：严格遵循 ITU-T（国际电信联盟）的相关建议，生成符合全球电信标准的音调，确保与其他设备的兼容性。
3.  **简单易用**：API 非常直观，通常只需一行代码即可播放一个音调。
4.  **低延迟**：由于直接驱动音频硬件，延迟非常低，适合需要即时反馈的场景（如拨号键盘）。

### 三、主要用途

1.  **DTMF 拨号音**：这是最经典的用途。当你按下手机拨号盘上的一个键（如 `1`, `2`, `#, *`），听到的声音就是由 `ToneGenerator` 生成的 DTMF 音，用于在模拟电话线上传输号码信息。
    *   例如：数字 `5` 是由 1336Hz 和 770Hz 的两个正弦波叠加而成。

2.  **呼叫进展音（Call Progress Tones）**：模拟传统电话中的各种状态提示音。
    *   **拨号音 (TONE_DIAL)**：连续的“嘟——”声，表示可以开始拨号。
    *   **回铃音 (TONE_SUP_RINGTONE)**：“嘟——嘟——”的周期性声音，表示对方电话正在响铃。
    *   **忙音 (TONE_SUP_BUSY)**：“嘟、嘟、嘟”的短促音，表示对方正在通话中或线路忙。
    *   **拥塞音 (TONE_SUP_CONGESTION)**：比忙音更慢的“嘟、嘟”声，表示网络拥塞。

3.  **应用内提示音**：虽然可以使用 SoundPool 或 MediaPlayer，但对于非常简短、无需定制、系统级的提示音（如一个简单的“嘀”声），`ToneGenerator` 是一个轻量级的选择。

### 四、工作流程与核心方法

1.  **创建实例**：
    ```java
    // 参数1: 流类型，通常使用 STREAM_DTMF，与通话音量关联
    // 参数2: 音量大小，范围是 0-100，通常直接设置为最大音量 100
    ToneGenerator tg = new ToneGenerator(AudioManager.STREAM_DTMF, 100);
    ```
    **重要**：`STREAM_DTMF` 类型的音频流会自动与通话音量轨道关联，这意味着它的音量可以通过手机的“通话音量”滑块来控制，并且通常在扬声器关闭时默认从听筒播出，适合电话场景。

2.  **播放音调**：
    ```java
    // 参数1: 音调类型，来自 ToneGenerator 的常量，如 TONE_DTMF_0, TONE_SUP_BUSY
    // 参数2: 持续时间（毫秒）。-1 表示播放直到手动停止
    tg.startTone(ToneGenerator.TONE_DTMF_0, 200); // 播放数字‘0’的音，持续200ms
    ```

3.  **停止播放**：
    *   如果设置了固定的持续时间（如上面的 200ms），音调会在时间到后自动停止。
    *   如果设置时间为 `-1`，则必须手动停止：
    ```java
    tg.stopTone(); // 停止当前正在播放的音调
    ```

4.  **释放资源**：
    `ToneGenerator` 直接访问音频硬件，是稀缺资源。**必须**在使用完毕后释放它，否则可能导致其他应用无法发声或系统资源泄漏。
    ```java
    tg.release(); // 非常重要！
    ```

### 五、一个完整的示例（模拟拨号键盘）

```java
public class DialerActivity extends AppCompatActivity {

    private ToneGenerator mToneGenerator;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_dialer);

        // 初始化ToneGenerator，音量设置为最大（100）
        mToneGenerator = new ToneGenerator(AudioManager.STREAM_DTMF, 100);
    }

    // 假设一个按钮的点击事件，按钮的tag设置为对应的DTMF音常量（如TONE_DTMF_1）
    public void onDigitButtonClicked(View view) {
        int toneType = Integer.parseInt(view.getTag().toString());
        // 播放一个短暂的DTMF音，持续120毫秒
        mToneGenerator.startTone(toneType, 120);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        // 确保释放资源
        if (mToneGenerator != null) {
            mToneGenerator.release();
            mToneGenerator = null;
        }
    }
}
```

### 六、重要注意事项和局限性

1.  **资源管理与释放**：这是最重要的一点。`ToneGenerator` 的底层实现会持有音频硬件资源。必须调用 `release()` 方法，并且最好在 `onPause()` 或 `onDestroy()` 等生命周期方法中进行。不释放会导致其他音频应用（包括MediaPlayer）无法播放声音，甚至引发系统级问题。

2.  **音量控制**：使用 `STREAM_DTMF` 意味着音量的控制依赖于用户的**通话音量**设置，而不是媒体音量或铃声音量。如果用户将通话音量调至静音，则 `ToneGenerator` 也不会发出声音。

3.  **音调定制性差**：`ToneGenerator` 只能播放其预设好的几十种音调（定义在类的常量中）。**你不能用它来生成任意频率、任意波形（如方波、锯齿波）的音调**。如果需要自定义音调，你需要使用 `AudioTrack` 并自己计算 PCM 数据。

4.  **并发播放**：通常，一个 `ToneGenerator` 实例一次只能播放一个音调。尝试在前一个音调结束前播放新的音调，行为是未定义的（可能被忽略，也可能中断前一个）。

5.  **系统资源限制**：虽然现代设备很少遇到，但系统对同时存在的 `ToneGenerator` 实例数量是有限制的。这就是为什么必须及时释放的原因。

### 七、与 AudioTrack 的对比

| 特性 | ToneGenerator | AudioTrack |
| :--- | :--- | :--- |
| **目的** | 播放**预定义**的标准音调（DTMF，呼叫音） | 播放**自定义**的 PCM 音频数据 |
| **易用性** | **高**，API 极其简单 | **低**，需要处理音频格式、数据写入、线程管理等 |
| **灵活性** | **低**，只能播放固定的音调 | **极高**，可以生成和播放任何声音 |
| **资源占用** | 占用系统音调生成资源 | 占用通用音频播放资源 |
| **适用场景** | 拨号盘、电话状态模拟、简单系统提示音 | 音乐播放器、游戏音效、任何需要自定义音频的场景 |

### 八、相关源码路径

frameworks/base/media/java/android/media/ToneGenerator.java

frameworks/base/core/jni/android_media_ToneGenerator.cpp

frameworks/av/media/libaudioclient/ToneGenerator.cpp

frameworks/av/services/audiopolicy/engine/common/src/EngineDefaultConfig.h#108

