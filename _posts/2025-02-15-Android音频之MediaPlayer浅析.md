---
title: Android音频之MediaPlayer浅析
tags: Android Audio MediaPlayer
---
## JAVA层

```java
public static MediaPlayer create(Context context, int resid, 
        AudioAttributes audioAttributes, int audioSessionId) {
    
    // 通过资源管理器打开原始资源文件描述符
    AssetFileDescriptor afd = context.getResources().openRawResourceFd(resid);
    if (afd == null) {
        // 资源打开失败时返回null（通常表示resid无效）
        return null;
    }

    // 创建新MediaPlayer实例
    MediaPlayer mp = new MediaPlayer();

    // 配置音频属性：使用传入参数或创建默认配置
    final AudioAttributes aa = (audioAttributes != null) 
            ? audioAttributes : new AudioAttributes.Builder().build();
    
    // 设置音频属性（替代旧setAudioStreamType方法）
    mp.setAudioAttributes(aa);
    
    // 配置音频会话ID（0=自动生成，非0=加入现有会话）
    mp.setAudioSessionId(audioSessionId);

    // 设置数据源（需传入文件描述符、起始偏移量和长度）
    // 注：此处使用原始资源的文件描述符，支持非压缩资源文件
    mp.setDataSource(afd.getFileDescriptor(), afd.getStartOffset(), afd.getLength());

    // 同步准备播放器（阻塞当前线程直到准备完成）
    mp.prepare(); // 可能抛出IllegalStateException或IOException
    
    return mp;
}
```



## 相关源码路径

**frameworks/base/media/java/android/media/MediaPlayer.java** 

**frameworks/base/media/jni/android_media_MediaPlayer.cpp** 

**frameworks/av/media/libmedia/mediaplayer.cpp** 

**frameworks/av/media/mediaserver/main_mediaserver.cpp** 

**frameworks/av/media/libmediaplayerservice/MediaPlayerService.cpp** 


## MediaPlayer类图

```plantuml
@startuml
interface IMediaPlayer {
    +{abstract} disconnect(): void
    +{abstract} setDataSource(int fd, int64_t offset, int64_t length): status_t
    +{abstract} prepare(): status_t
    +{abstract} start(): status_t
    +{abstract} stop(): status_t
    +{abstract} pause(): status_t
    +{abstract} isPlaying(): status_t
    +{abstract} setVolume(float leftVolume, float rightVolume): status_t
    +{abstract} setOutputDevice(audio_port_handle_t deviceId): status_t
}
class IMediaDeathNotifier {
    +IMediaDeathNotifier()
    +~IMediaDeathNotifier()
    +{abstract} died(): void
    +{static} getMediaPlayerService():IMediaPlayerService
    -{static} IMediaPlayerService sMediaPlayerService
    -{static} SortedVector<IMediaDeathNotifier> sObitRecipients
}
class MediaPlayer {
    +MediaPlayer()
    +~MediaPlayer()
    +died()
    +attachNewPlayer(const sp<IMediaPlayer>& player)
    +setDataSource(int fd, int64_t offset, int64_t length)
    +prepare()
    +start()
    +stop()
    +pause()
    +isPlaying()
    +setVolume(float leftVolume, float rightVolume)
    +setOutputDevice(audio_port_handle_t deviceId)
    -- private data --
    -IMediaPlayer mPlayer
}
interface IMediaPlayerService {
    +{abstract} createMediaRecorder(const String16 &opPackageName):IMediaRecorder
    +{abstract} createMetadataRetriever():IMediaMetadataRetriever
    +{abstract} create(const sp<IMediaPlayerClient>& client,...):IMediaPlayer
    +{abstract} getCodecList():IMediaCodecList
}
class MediaPlayerService {
    +create(const sp<IMediaPlayerClient>& client,...):IMediaPlayer
    +getCodecList():IMediaCodecList
    +instantiate()
    -MediaPlayerService()
    -~MediaPlayerService()
    -SortedVector<Client> mClients
    -SortedVector<MediaRecorderClient> mMediaRecordClients
}
class Client {
  +setDataSource(int fd, int64_t offset, int64_t length)
  +prepare()
  +start()
  +stop()
  +pause()
  +isPlaying()
  +setVolume(float leftVolume, float rightVolume)
  +setOutputDevice(audio_port_handle_t deviceId)
  -- private data --
  -MediaPlayerBase mPlayer
  -MediaPlayerService mService
  -AudioOutput mAudioOutput
}
class AudioOutput {
    +AudioOutput(...)
    +~AudioOutput()
    +open(uint32_t sampleRate, int channelCount, audio_channel_mask_t channelMask...)
    +start()
    +write(const void* buffer, size_t size, bool blocking = true)
    +stop()
    +flush()
    +pause()
    +close()
    +setOutputDevice(audio_port_handle_t deviceId)
    -- private data --
    -AudioTrack mTrack
}
interface AudioSink {
    +{abstract} open(uint32_t sampleRate, int channelCount, audio_channel_mask_t channelMask...)
    +{abstract} start()
    +{abstract} write(const void* buffer, size_t size, bool blocking = true)
    +{abstract} stop()
    +{abstract} flush()
    +{abstract} pause()
    +{abstract} close()
    +{abstract} setOutputDevice(audio_port_handle_t deviceId)
}
interface MediaPlayerBase {
    +{abstract} hardwareOutput(): bool
    +{abstract} setDataSource(int fd, int64_t offset, int64_t length)
    +{abstract} prepare()
    +{abstract} prepareAsync()
    +{abstract} start()
    +{abstract} stop()
    +{abstract} pause()
    +{abstract} isPlaying()
}
class MediaPlayerInterface {
    +hardwareOutput(){return false;}: bool
    +setAudioSink(const sp<AudioSink>& audioSink){mAudioSink=audioSink;}: void
    -- private data --
    -AudioSink mAudioSink
}
class NuPlayerDriver {
    +setDataSource(int fd, int64_t offset, int64_t length)
    +prepare()
    +prepareAsync()
    +start()
    +stop()
    +pause()
    +isPlaying()
    -- private data --
    -ALooper mLooper
    -MediaClock mMediaClock
    -NuPlayer mPlayer
    -AudioSink mAudioSink
}

Client <|.. IMediaPlayer
IMediaDeathNotifier <|.. MediaPlayer
MediaPlayerService <|.. IMediaPlayerService

IMediaPlayer *-- MediaPlayer
IMediaPlayerService *-- IMediaDeathNotifier
MediaPlayerBase <|.. MediaPlayerInterface
MediaPlayerInterface <|.. NuPlayerDriver
AudioSink <|.. AudioOutput
@enduml
```

## MediaPlayerService初始化流程

```plantuml
@startuml
participant init #red
Activate mediaserver
Activate MediaPlayerService
Activate MediaPlayerFactory
Activate NuPlayerFactory

init->mediaserver: main()
mediaserver->MediaPlayerService: instantiate()
MediaPlayerService->MediaPlayerService: new MediaPlayerService()
MediaPlayerService->MediaPlayerFactory: registerBuiltinFactories()
MediaPlayerFactory->NuPlayerFactory: IFactory* factory = new NuPlayerFactory()
MediaPlayerFactory->MediaPlayerFactory: registerFactory_l(factory, NU_PLAYER)
MediaPlayerService->MediaPlayerService: addService("media.player"...)

@enduml
```

## setDataSource流程

```plantuml
@startuml
actor Actor
Activate MediaPlayer
participant IMediaDeathNotifier
Activate MediaPlayerService
Activate MediaPlayerService_Client
Activate MediaPlayerFactory
Activate NuPlayerFactory
Activate NuPlayerDriver
Activate AVNuFactory
Activate NuPlayer
Activate AudioOutput

Actor->MediaPlayer: setDataSource(int fd...)
MediaPlayer->IMediaDeathNotifier: getMediaPlayerService()
IMediaDeathNotifier->IMediaDeathNotifier: getService("media.player")
MediaPlayer->MediaPlayerService: create()
MediaPlayerService->MediaPlayerService_Client: new Client()
MediaPlayer->MediaPlayerService_Client: setDataSource(int fd...)
MediaPlayerService_Client->MediaPlayerFactory : MediaPlayerFactory::getPlayerType(...)
MediaPlayerService_Client->MediaPlayerService_Client: setDataSource_pre(playerType)
MediaPlayerService_Client->MediaPlayerFactory : MediaPlayerFactory::createPlayer(playerType...)
MediaPlayerFactory->NuPlayerFactory : factory->createPlayer(...)
NuPlayerFactory->NuPlayerDriver: new NuPlayerDriver(pid)
NuPlayerDriver->AVNuFactory: AVNuFactory::get()->createNuPlayer(...)
AVNuFactory->NuPlayer: new NuPlayer(...)
NuPlayerDriver->NuPlayer: mPlayer->init(this)
NuPlayer->NuPlayer: mDriver = driver;
MediaPlayerService_Client->AudioOutput: mAudioOutput = new AudioOutput(...)
MediaPlayerService_Client->NuPlayerDriver: p->setDataSource(fd...)
NuPlayerDriver->NuPlayer: mPlayer->setDataSourceAsync(...)
NuPlayer->NuPlayer: new GenericSource(notify, mUIDValid, mUID, mMediaClock)
NuPlayer->NuPlayer: source->setDataSource(fd, offset, length)
NuPlayer->NuPlayer: onMessageReceived(const sp<AMessage> &msg)
NuPlayer->NuPlayer: mSource = static_cast<Source *>(obj.get())
MediaPlayer->MediaPlayer: attachNewPlayer(player)
MediaPlayer->MediaPlayer: mPlayer = player;
@enduml
```

## prepare流程

```plantuml
@startuml
actor Actor
Activate MediaPlayer
participant IMediaDeathNotifier
Activate MediaPlayerService
Activate MediaPlayerService_Client
Activate NuPlayerDriver
Activate AVNuFactory
Activate NuPlayer
Activate AudioOutput
Activate GenericSource

Actor->MediaPlayer: prepare()
MediaPlayer->MediaPlayer: prepareAsync_l()
MediaPlayer->MediaPlayerService_Client: setParameter()
MediaPlayerService_Client->MediaPlayerService_Client: setAudioAttributes_l()
MediaPlayerService_Client->AudioOutput: setAudioAttributes()
AudioOutput->AudioOutput: mStreamType = AudioSystem::attributesToStreamType(*attributes)
MediaPlayer->MediaPlayer: mCurrentState = MEDIA_PLAYER_PREPARING;
MediaPlayer->MediaPlayerService_Client: prepareAsync()
MediaPlayerService_Client->MediaPlayerService_Client: getPlayer()
MediaPlayerService_Client->NuPlayerDriver: prepareAsync()
NuPlayerDriver->NuPlayer: prepareAsync()
NuPlayer->NuPlayer: (new AMessage(kWhatPrepare, this))->post();
NuPlayer->NuPlayer: onMessageReceived()
NuPlayer->GenericSource: prepareAsync()
GenericSource->GenericSource: onPrepareAsync()
GenericSource->GenericSource: initFromDataSource()
@enduml
```