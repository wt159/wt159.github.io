---
title: ffmpeg结构体分析之AVFrame
tags: ffmpeg video
mermaid: true
---

# 概述

> 基于ffmpeg 4.3.2

AVFrame结构体一般用于存储原始数据（即非压缩数据，例如对视频来说是YUV，RGB，对音频来说是PCM），此外还包含了一些相关的信息。比如说，解码的时候存储了宏块类型表，QP表，运动矢量表等数据。编码的时候也存储了相关的数据。

[参考1-雷神](https://blog.csdn.net/leixiaohua1020/article/details/14214577) [参考2](https://www.cnblogs.com/leisure_chn/p/10404502.html)

# 定义

```c
typedef struct AVFrame {
#define AV_NUM_DATA_POINTERS 8
    /**
     * 指针指向图像/通道平面。
     * 这可能与第一个被分配的字节不同
     *
     * 一些解码器访问0,0以外的区域--宽度、高度，请
     * 见avcodec_align_dimensions2()。一些滤波器和swscale可以读取
     * 多达16个字节的平面，如果要使用这些滤波器。
     * 则必须分配额外的16个字节。
     *
     * 注意：除了hwaccel格式，格式不需要的指针
     * 的指针必须被设置为NULL。
     */
    uint8_t *data[AV_NUM_DATA_POINTERS];

    /**
     * 对于视频来说，每条图片线的大小（以字节为单位）。
     * 对于音频，每个平面的大小以字节为单位。
     *
     * 对于音频来说，只有linesize[0]可以被设置。对于平面的音频，每个通道
     * 平面必须是相同的尺寸。
     *
     * 对于视频来说，行的大小应该是CPU排列的倍数。
     * 偏好，对于现代桌面CPU来说，这是16或32。
     * 有些代码需要这样的对齐方式，其他代码在没有正确对齐的情况下可能会更慢。
     * 正确的对齐方式，而对于其他的代码则没有区别。
     *
     * @注意 行的大小可能大于可用数据的大小 -- 可能有额外的填充。
     *由于性能原因，可能会有额外的填充存在。
     */
    int linesize[AV_NUM_DATA_POINTERS];

    /**
     * 指向数据平面/通道的指针。
     *
     * 对于视频，这应该简单地指向data[]。
     *
     * 对于平面音频，每个通道都有一个单独的数据指针，并且
     * linesize[0] 包含每个通道缓冲区的大小。
     * 对于打包的音频，只有一个数据指针，linesize[0] *包含所有通道的总大小。
     * 包含所有通道的缓冲区的总大小。
     *
     * 注意：data和extension_data都应该始终设置在一个有效的帧中。
     * 但对于有更多通道的平面音频，可以用data来装。
     * 必须使用extend_data，以便访问所有的通道。
     */
    uint8_t **extended_data;

    /**
     * @名称 视频尺寸
     * 仅限视频帧。视频帧的编码尺寸（单位：像素）。
     *即包含一些明确定义的值的矩形的大小。
     *
     * 注释 拟用于显示/呈现的帧的部分被进一步
     *由@ref裁剪 "裁剪矩形 "限制。
     * @{
     */
    int width, height;
    /**
     * @}
     */

    /**
     * 该帧所描述的音频样本数（每通道）。
     */
    int nb_samples;

    /**
     * 帧的格式，如果未知或未设置则为-1
     *值对应于视频帧的enum AVPixelFormat。
     * enum AVSampleFormat用于音频)
     */
    int format;

    /**
     * 1 -> 关键帧, 0-> not
     */
    int key_frame;

    /**
     * 帧的图片类型。
     */
    enum AVPictureType pict_type;

    /**
     * 视频帧的采样宽高比, 0/1 if unknown/unspecified.
     */
    AVRational sample_aspect_ratio;

    /**
     * 以Time_base为单位的展示时间戳（框架应该显示给用户的时间）。
     */
    int64_t pts;

#if FF_API_PKT_PTS
    /**
     * 从被解码的AVPacket中复制的PTS，以产生这个帧。
     * @deprecated use the pts field instead.
     */
    attribute_deprecated
    int64_t pkt_pts;
#endif

    /**
     * 从触发返回该帧的AVPacket中复制的DTS。(如果没有使用帧线程)
     * 这也是这个AVFrame的演示时间，它是由以下数据计算出来的
     * 只有AVPacket.dts值，没有pts值。
     */
    int64_t pkt_dts;

    /**
     * 按比特流顺序显示的图像编号
     */
    int coded_picture_number;
    /**
     * 按显示顺序显示的图片编号
     */
    int display_picture_number;

    /**
     * quality (between 1 (good) and FF_LAMBDA_MAX (bad))
     */
    int quality;

    /**
     * for some private data of the user
     */
    void *opaque;

#if FF_API_ERROR_FRAME
    /**
     * @deprecated unused
     */
    attribute_deprecated
    uint64_t error[AV_NUM_DATA_POINTERS];
#endif

    /**
     * When decoding, this signals how much the picture must be delayed.
     * extra_delay = repeat_pict / (2*fps)
     */
    int repeat_pict;

    /**
     * The content of the picture is interlaced.
     */
    int interlaced_frame;

    /**
     * If the content is interlaced, is top field displayed first.
     */
    int top_field_first;

    /**
     * Tell user application that palette has changed from previous frame.
     */
    int palette_has_changed;

    /**
     * reordered opaque 64 bits (generally an integer or a double precision float
     * PTS but can be anything).
     * The user sets AVCodecContext.reordered_opaque to represent the input at
     * that time,
     * the decoder reorders values as needed and sets AVFrame.reordered_opaque
     * to exactly one of the values provided by the user through AVCodecContext.reordered_opaque
     */
    int64_t reordered_opaque;

    /**
     * Sample rate of the audio data. 音频数据采样率
     */
    int sample_rate;

    /**
     * Channel layout of the audio data.音频数据的声道布局
     */
    uint64_t channel_layout;

    /**
     * AVBuffer引用支持该帧的数据。如果这个数组的所有元素
     * 的所有元素都是NULL，那么这个帧就没有被引用。这个数组
     * 必须是连续填充的 -- 如果buf[i]不是NULL，那么buf[j]也必须不是NULL。
     * 对于所有的j < i，也必须是非NULL。
     *
     * 每个数据平面最多可以有一个AVBuffer，所以对于视频来说，这个数组
     * 总是包含所有的引用。对于平面音频来说，有多于
     * AV_NUM_DATA_POINTERS通道，可能会有更多的缓冲区无法容纳在
     * 这个数组。然后，额外的AVBufferRef指针被存储在
     * extended_buf数组。
     */
    AVBufferRef *buf[AV_NUM_DATA_POINTERS];

    /**
     * 对于需要多于AV_NUM_DATA_POINTERS的平面音频
     * 的AVBufferRef指针，这个数组将保存所有不能放入AVFrame.buf的引用。
     *不能装入AVFrame.buf。
     *
     * 注意，这与AVFrame.extended_data不同，后者总是
     * 包含所有的指针。这个数组只包含额外的指针。
     *不能放入AVFrame.buf中。
     *
     * 这个数组总是由构造框架的人使用av_malloc()分配。
     * 框架。它在av_frame_unref()中被释放。
     */
    AVBufferRef **extended_buf;
    /**
     * Number of elements in extended_buf.
     */
    int        nb_extended_buf;

    AVFrameSideData **side_data;
    int            nb_side_data;

/**
 * @defgroup lavu_frame_flags AV_FRAME_FLAGS
 * @ingroup lavu_frame
 * Flags describing additional frame properties.
 *
 * @{
 */

/**
 * The frame data may be corrupted, e.g. due to decoding errors.
 */
#define AV_FRAME_FLAG_CORRUPT       (1 << 0)
/**
 * A flag to mark the frames which need to be decoded, but shouldn't be output.
 */
#define AV_FRAME_FLAG_DISCARD   (1 << 2)
/**
 * @}
 */

    /**
     * Frame flags, a combination of @ref lavu_frame_flags
     */
    int flags;

    /**
     * MPEG vs JPEG YUV range.
     * - encoding: Set by user
     * - decoding: Set by libavcodec
     */
    enum AVColorRange color_range;

    enum AVColorPrimaries color_primaries;

    enum AVColorTransferCharacteristic color_trc;

    /**
     * YUV colorspace type.
     * - encoding: Set by user
     * - decoding: Set by libavcodec
     */
    enum AVColorSpace colorspace;

    enum AVChromaLocation chroma_location;

    /**
     * frame timestamp estimated using various heuristics, in stream time base
     * - encoding: unused
     * - decoding: set by libavcodec, read by user.
     */
    int64_t best_effort_timestamp;

    /**
     * reordered pos from the last AVPacket that has been input into the decoder
     * - encoding: unused
     * - decoding: Read by user.
     */
    int64_t pkt_pos;

    /**
     * duration of the corresponding packet, expressed in
     * AVStream->time_base units, 0 if unknown.
     * - encoding: unused
     * - decoding: Read by user.
     */
    int64_t pkt_duration;

    /**
     * metadata.
     * - encoding: Set by user.
     * - decoding: Set by libavcodec.
     */
    AVDictionary *metadata;

    /**
     * decode error flags of the frame, set to a combination of
     * FF_DECODE_ERROR_xxx flags if the decoder produced a frame, but there
     * were errors during the decoding.
     * - encoding: unused
     * - decoding: set by libavcodec, read by user.
     */
    int decode_error_flags;
#define FF_DECODE_ERROR_INVALID_BITSTREAM   1
#define FF_DECODE_ERROR_MISSING_REFERENCE   2
#define FF_DECODE_ERROR_CONCEALMENT_ACTIVE  4
#define FF_DECODE_ERROR_DECODE_SLICES       8

    /**
     * number of audio channels, only used for audio.音频数据通道
     * - encoding: unused
     * - decoding: Read by user.
     */
    int channels;

    /**
     * 含有压缩后的数据包的相应大小。
     * 帧。
     * 如果未知，它将被设置为一个负值。
     * - encoding: unused
     * - decoding: set by libavcodec, read by user.
     */
    int pkt_size;

#if FF_API_FRAME_QP
    /**
     * QP table
     */
    attribute_deprecated
    int8_t *qscale_table;
    /**
     * QP store stride
     */
    attribute_deprecated
    int qstride;

    attribute_deprecated
    int qscale_type;

    attribute_deprecated
    AVBufferRef *qp_table_buf;
#endif
    /**
     * For hwaccel-format frames, this should be a reference to the
     * AVHWFramesContext describing the frame.
     */
    AVBufferRef *hw_frames_ctx;

    /**
     * AVBufferRef for free use by the API user. FFmpeg will never check the
     * contents of the buffer ref. FFmpeg calls av_buffer_unref() on it when
     * the frame is unreferenced. av_frame_copy_props() calls create a new
     * reference with av_buffer_ref() for the target frame's opaque_ref field.
     *
     * This is unrelated to the opaque field, although it serves a similar
     * purpose.
     */
    AVBufferRef *opaque_ref;

    /**
     * @anchor cropping
     * @name Cropping
     * Video frames only. The number of pixels to discard from the the
     * top/bottom/left/right border of the frame to obtain the sub-rectangle of
     * the frame intended for presentation.
     * @{
     */
    size_t crop_top;
    size_t crop_bottom;
    size_t crop_left;
    size_t crop_right;
    /**
     * @}
     */

    /**
     * AVBufferRef供单个libav*库内部使用。
     * 不得用于在库之间传输数据。
     * 当帧的所有权离开各自的库时，必须为NULL。
     *
     * FFmpeg库以外的代码不应该检查或改变缓冲区的内容。
     *
     * FFmpeg在帧未被引用时调用av_buffer_unref()。
     * av_frame_copy_props()调用av_buffer_ref()创建一个新的引用。
     * 为目标帧的private_ref字段创建一个新的引用。
     */
    AVBufferRef *private_ref;
} AVFrame;
```

# 详解

## 通用

### 数据缓冲区指针

对于packed格式的数据（例如RGB24），会存到data[0]里面。

对于planar格式的数据（例如YUV420P），则会分开成data[0]，data[1]，data[2]...（YUV420P中data[0]存Y，data[1]存U，data[2]存V）

```c
uint8_t *data[AV_NUM_DATA_POINTERS];
```

### 数据缓冲区大小

```c
int linesize[AV_NUM_DATA_POINTERS];
```

### 编码后的数据包大小

```c
/**
     * 含有压缩后的数据包的相应大小。
     * 帧。
     * 如果未知，它将被设置为一个负值。
     * - encoding: unused
     * - decoding: set by libavcodec, read by user.
     */
    int pkt_size;
```

### 格式

```c
/**
     * 帧的格式，如果未知或未设置则为-1
     *值对应于视频帧的enum AVPixelFormat。
     * 音频的enum AVSampleFormat)
     */
    int format;
```

## 音频

### 样本数

```c
    /**
     * 该帧所描述的音频样本数（每通道）。
     */
    int nb_samples;
```

### 采样率

```c
    /**
     * Sample rate of the audio data. 音频数据采样率
     */
    int sample_rate;
```

### 声道数

```c
    /**
     * number of audio channels, only used for audio.音频数据通道
     * - encoding: unused
     * - decoding: Read by user.
     */
    int channels;
```

### 声道布局

```c
    /**
     * Channel layout of the audio data.音频数据的声道布局
     */
    uint64_t channel_layout;
```

## 视频

### 视频帧宽和高（1920x1080,1280x720...）

```c
int width, height;
```

### 关键帧

```c
/**
 * 1 -> 关键帧, 0-> not
 */
int key_frame;
```

### 帧类型

```c
/**
 * 帧的图片类型。
 */
enum AVPictureType pict_type;
```

包含以下类型：

```
enum AVPictureType {
    AV_PICTURE_TYPE_NONE = 0, ///< Undefined
    AV_PICTURE_TYPE_I,     ///< Intra
    AV_PICTURE_TYPE_P,     ///< Predicted
    AV_PICTURE_TYPE_B,     ///< Bi-dir predicted
    AV_PICTURE_TYPE_S,     ///< S(GMC)-VOP MPEG-4
    AV_PICTURE_TYPE_SI,    ///< Switching Intra
    AV_PICTURE_TYPE_SP,    ///< Switching Predicted
    AV_PICTURE_TYPE_BI,    ///< BI type
};
```

### 宽高比

```c
/**
 * 视频帧的采样宽高比, 0/1 if unknown/unspecified.
 */
AVRational sample_aspect_ratio;
```

详细格式如下：

```
/**
 * Rational number (pair of numerator and denominator).
 */
typedef struct AVRational{
    int num; ///< Numerator
    int den; ///< Denominator
} AVRational;
```

### 显示时间戳

```c
/**
 * 以Time_base为单位的展示时间戳（框架应该显示给用户的时间）。
 */
int64_t pts;
```

### 编码帧序号

```c
/**
 * 按比特流顺序显示的图像编号
 */
int coded_picture_number;
```

int display_picture_number：

### 显示帧序号

```c
/**
 * 按显示顺序显示的图片编号
 */
int display_picture_number;
```

### QP表

```c
int8_t *qscale_table;
```

### 是否是隔行扫描

```c
int interlaced_frame;
```
