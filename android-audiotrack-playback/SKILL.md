---
name: "android-audiotrack-playback"
description: "Documents the Android AudioTrack playback pipeline from Java to speaker: track creation, shared memory cblk allocation, ClientProxy/ServerProxy futex sync, start threshold, MixerThread threadLoop mixing, AIDL HAL FMQ transfer, and underrun diagnosis. Invoke when analyzing playback glitches, start latency, no-sound, buffer sizing, or track creation failures."
---

# Android AudioTrack 播放流程详解

本技能沿着 `new AudioTrack()` 一路追到扬声器，把播放链路的五个阶段拆开：创建 Track → 分配共享内存 → 写数据 → start 入队 → MixerThread 混音 → HAL 出硬件。每段都标出「谁在等谁」和「出错会长什么样」。

适用场景：播放无声 / 起播延迟异常 / 卡顿 underrun / 缓冲区该开多大 / Track 创建失败 / 想搞清楚一次 `write()` 到底拷贝了几次。

## 一、全景：四行代码 ↔ 五个阶段

应用层写的四行代码，底下对应五个阶段：

```java
AudioTrack track = new AudioTrack.Builder()          // ① 创建
        .setAudioAttributes(...).setAudioFormat(...)
        .setBufferSizeInBytes(bytes).build();
track.write(pcm, 0, pcm.length);                     // ② 写数据
track.play();                                        // ③ 起播
// ④ 混音、⑤ 出硬件：App 不需要写，AudioFlinger 后台自动
```

| 阶段 | App 看到的 | 底下真正发生什么 | 关键类 |
|:---:|-----------|----------------|--------|
| ① 创建 | `new AudioTrack()` | Binder 到 audioserver，路由选 output，分配共享内存 | `AudioFlinger::createTrack` → `TrackBase` |
| ② 写 | `write()` | memcpy 进共享内存环形缓冲，推 `user` 指针 | `AudioTrackClientProxy` |
| ③ 起播 | `play()` | Track 挂进 `mActiveTracks`，唤醒 MixerThread | `Track::start` |
| ④ 混音 | 无感 | 周期读共享内存、重采样、多路相加 | `MixerThread::threadLoop` |
| ⑤ 出硬件 | 无感 | memcpy 进 FMQ → HAL → tinyalsa → DMA | `StreamHalAidl` / `IStreamOut` |

**一句话记住架构**：一个 Java `AudioTrack` 对应一个 AudioFlinger `Track`，一个 Track 持有一块共享内存；App 往共享内存写，AudioFlinger 从同一块内存读，两边靠读写指针 + futex 协调，**音频数据不走 Binder**。

## 二、阶段一：AudioTrack 创建

### 2.1 Java 层：Builder 与两种构造方式

```java
// 推荐写法：Builder
new AudioTrack.Builder()
    .setAudioAttributes(new AudioAttributes.Builder()
        .setUsage(AudioAttributes.USAGE_MEDIA)
        .setContentType(AudioAttributes.CONTENT_TYPE_MUSIC)
        .build())
    .setAudioFormat(new AudioFormat.Builder()
        .setEncoding(AudioFormat.ENCODING_PCM_16BIT)
        .setSampleRate(48000)
        .setChannelMask(AudioFormat.CHANNEL_OUT_STEREO)
        .build())
    .setBufferSizeInBytes(bufferSize)
    .setTransferMode(AudioTrack.MODE_STREAM)
    .build();
```

Builder 不显式设置的默认值（很容易踩坑）：

| 属性 | 缺省值 |
|-----|--------|
| AudioAttributes | `USAGE_MEDIA` |
| AudioFormat | 立体声 / `PCM_16BIT` / 采样率由设备决定 |
| bufferSizeInBytes | 流模式下自动取最小值 |
| transferMode | `MODE_STREAM` |
| sessionId | 自动生成新 ID |

**旧构造器（已废弃但仍兼容）会做一次隐式翻译**：`AudioTrack(int streamType, ...)` → `setLegacyStreamType(streamType)` → `usageForStreamType()` 的 switch 映射（`STREAM_MUSIC → USAGE_MEDIA`、`STREAM_VOICE_CALL → USAGE_VOICE_COMMUNICATION`、`STREAM_ALARM → USAGE_ALARM`…）。

⚠️ **Android 15 的一个行为变化**：从 streamType 推导 AudioAttributes 时，**不再从策略里读 contentType**，且 `STREAM_MUSIC` 的 contentType 保持 `CONTENT_TYPE_UNKNOWN`（commit `2eb20eb`，理由：streamType 不足以表达 contentType）。以前写死 `CONTENT_TYPE_MUSIC` 的代码，在 dumpsys 里会看到 `content=CONTENT_TYPE_UNKNOWN`，**这不是 bug**。

### 2.2 JNI 层：参数过河

`frameworks/base/core/jni/android_media_AudioTrack.cpp` 里的 `android_media_AudioTrack_setup()`：

- `AudioAttributes` → `audio_attributes_t*`（身份：usage / contentType / flags / tags）
- `int[]` → 逐项拆成 sampleRate / channelMask / format / sessionId
- `Parcel` → `AttributionSourceState`（调用者身份，用于权限与归因）
- 按 `MODE_STREAM` / `MODE_STATIC` 二选一：`MODE_STREAM` 传 `0`（共享内存为 null，交给服务端分配）且 transferType 用 `TRANSFER_SYNC`；`MODE_STATIC` 走 `allocSharedMem()` 自己分配

### 2.3 Native 层：`AudioTrack::set()` 里的那处硬改写

`frameworks/av/media/libaudioclient/AudioTrack.cpp`，`AudioTrack::set()`：

```cpp
if ((flags & AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) || !audio_is_linear_pcm(format)) {
    flags = (flags | AUDIO_OUTPUT_FLAG_DIRECT) & ~AUDIO_OUTPUT_FLAG_FAST;
}
```

**这一句是「为什么我的 Track 没走 Fast 通道」的第一现场**：只要格式不是线性 PCM，或者带 offload 标志，系统会强制加上 `DIRECT` 并且**清掉 `FAST`**。你在 App 里请求低延迟，底层直接被降级，而且不会报错。

紧接着 `createTrack_l()` 会再补一刀：只有 `fastAllowed`（当前是前台 / 有可感知的音频活动）才保留 `FAST` 标志，否则同样被清掉。

然后构造 AIDL 请求并跨进程：

```
CreateTrackRequest {
    attr, config{format, channelMask, sampleRate}, flags,
    frameCount, notificationFrameCount, notificationsPerBuffer,
    speed, sharedBuffer, sessionId, selectedDeviceId,
    audioTrackCallback, clientInfo{clientPid, clientTid},
    attributionSource, ...
}
```

### 2.4 AudioFlinger 侧：`createTrack()` 的 12 步

`frameworks/av/services/audioflinger/AudioFlinger.cpp`，`AudioFlinger::createTrack()`：

```
 1. 解包 CreateTrackRequest（AIDL → 内部结构）
 2. AudioSystem::getOutputForAttr()
      → 路由决策，拿到 output.outputId / selectedDeviceId / portId
      → 同时返回 isSpatialized / isBitPerfect（Android 15 已进入主流程）
 3. 多音区场景：算出 secondaryOutputs
 4. 分配 portId（AUDIO_PORT_HANDLE_NONE 时）
 5. checkPlaybackThread_l(output.outputId) → 拿到已存在的播放线程
 6. registerPid(callingPid) → Client（按进程复用，wp 弱引用防环）
 7. 检查同 sessionId 的效果链是否在别的线程上（要在后面搬过来）
 8. thread->createTrack_l(...)  ★ 真正创建 Track
 9. 回填线程参数：afFrameCount / afSampleRate / afChannelMask / afFormat / afLatencyMs
10. updateSecondaryOutputsForTrack_l()  → 连接辅助输出
11. moveEffectChain_ll()                → 把效果链搬到本线程
12. output.audioTrack = IAfTrack::createIAudioTrackAdapter(track)  → 返回给客户端
```

**第 5 步是关键认知**：AudioFlinger 并不会为每个 Track 新开线程。线程是 `openOutput_l()` 时就按 output 建好的（`MixerThread` / `DirectOutputThread` / `OffloadThread` / `MmapPlaybackThread`），createTrack 只是把 Track 挂到一个**已经在跑的线程**上。

`outputId → 线程` 的对应关系（某 Tensor 机型 dump 实例，**ID 每台机器不同，仅供理解规律**）：

| flags | device | outputId | 线程类型 |
|-------|--------|:--------:|---------|
| `PRIMARY \| FAST` | SPEAKER | 13 | MixerThread |
| `DEEP_BUFFER` | SPEAKER | 21 | MixerThread |
| `DIRECT \| COMPRESS_OFFLOAD \| NON_BLOCKING` | SPEAKER | 29 | OffloadThread |
| `NONE` | SPEAKER | 37 | MixerThread |
| `FAST \| RAW` | SPEAKER | 45 | MixerThread |
| `DIRECT \| MMAP_NOIRQ` | SPEAKER | 53 | MmapPlaybackThread |
| `INCALL_MUSIC` | TELEPHONY_TX | 61 | MixerThread |
| `NONE` | TELEPHONY_TX | 69 | MixerThread |
| `NONE` | REMOTE_SUBMIX | 77 | MixerThread |

### 2.5 `TrackBase` 构造：共享内存真正落地的地方

前面 12 步都只是把 `sharedBuffer` 这个参数往下传。**真正分配共享内存的是 `TrackBase` 构造函数**（`frameworks/av/services/audioflinger/Tracks.cpp`，约 84 行起）。

```cpp
// 第 1 步：算大小
size_t size = sizeof(audio_track_cblk_t);        // 控制块
if (buffer == NULL && alloc == ALLOC_CBLK) {     // STREAM + 普通混音
    size += bufferSize;                          // 控制块和数据区拼成一块
}

// 第 2 步：从 Client 的内存分配器分配（按进程管理）
mCblkMemory = client->allocator().allocate(roundup(size), ...);
mCblk = static_cast<audio_track_cblk_t*>(mCblkMemory->unsecurePointer());

// 第 3 步：数据区紧跟控制块
case ALLOC_CBLK:
    if (buffer == NULL) {                                  // STREAM
        mBuffer = (char*)mCblk + sizeof(audio_track_cblk_t);
    } else {                                               // STATIC
        mBuffer = buffer;                                  // 用 App 传下来的
    }
```

内存布局：

```
┌────────────────────── 一块跨进程共享内存 ──────────────────────┐
│  audio_track_cblk_t（控制块）  │      PCM 数据环形缓冲区          │
│  读写指针 / futex / 标志位     │  ← mBuffer 指向这里             │
└───────────────────────────────────────────────────────────────┘
        ↑ mCblk                                         bufferSize
```

**控制块和数据放一起是刻意的**：两个进程 mmap 同一块内存，看到的就是同一份指针，不需要第二次映射，也不需要额外同步元数据。

**`MODE_STREAM` vs `MODE_STATIC` 的本质区别就是谁分配内存**：

| 维度 | MODE_STREAM | MODE_STATIC |
|-----|-------------|-------------|
| 共享内存 | AudioFlinger 分配 | App 侧 `allocSharedMem()` 分配后传下来 |
| 大小 | 几十毫秒容量（按延迟算） | 整段音频长度 |
| 写入时机 | 播放中持续 `write()` | `play()` 之前一次写完 |
| 阻塞 | 满了会 futex 睡眠 | 不阻塞，直接 memcpy |
| underrun | 存在（App 供数据不及时） | **不存在**（数据全在内存里） |
| 循环播放 | App 要反复写 | `setLoopPoints()` 由 AF 内部循环，**零 CPU** |
| 典型用途 | 音乐 / 视频 / 通话 | 按键音 / 通知音 / 游戏音效 |

> ⚠️ 常见误解：`MODE_STATIC` 不是「把数据预加载进 ring buffer」。服务端的读取逻辑完全不同——它知道整块数据都已就位，不需要等 App 供数据，也不统计 underrun。

## 三、阶段二：共享内存与两个 Proxy

### 3.1 `audio_track_cblk_t`：跨进程的「控制面板」

定义在 `frameworks/av/media/libaudioclient/include/media/AudioTrackShared.h`（约 207 行）。

| 字段 | 作用 |
|-----|------|
| `mServer` / `mFront` / `mRear` | 服务端位置、客户端写指针、客户端提交指针（单位：帧，会环绕） |
| `mFutex` | 等待/唤醒原语（普通播放用，非 fast 模式） |
| `mFrameCount` | 环形缓冲区总容量（帧） |
| `mSampleRate` / `mFrameSize` | 采样率、每帧字节数 |
| `mVolumeLR` | 打包的左右声道音量（Q4.28 定点） |
| `mMinimum` | 混音前最低需要的帧数 |
| `mFlags` | `CBLK_UNDERRUN` / `CBLK_FORCEREADY` / `CBLK_INVALID` / `CBLK_DISABLED` / `CBLK_STREAM_END_DONE` |

**为什么用原子操作 + futex 而不是 mutex**：跨进程 mutex 每次加解锁都可能进内核（robust futex），而环形缓冲天然是「单写者 + 单读者」，用原子操作更新指针就够了；只有真正需要等待时才 futex 睡下去。这是实时音频路径上「锁越少越稳」的直接体现。

### 3.2 ClientProxy 与 ServerProxy

直接操作指针 + futex 太容易出错，所以封装成两个代理，**同一块内存、同一对指针，一个只写一个只读**：

| 角色 | 类 | 位置 | 职责 |
|-----|----|------|------|
| 写（生产者） | `AudioTrackClientProxy` | App 进程 | `obtainBuffer()` 取空位、`releaseBuffer()` 推写指针、满了 futex 等 |
| 读（消费者） | `AudioTrackServerProxy` | audioserver | `obtainBuffer()` 读数据、`releaseBuffer()` 推读指针、统计 underrun |

Track 侧读数据的入口是 `Track::getNextBuffer()`（`Tracks.cpp`），它内部调 `mServerProxy->obtainBuffer(&buf)`，把 `buf.raw` 和 `buf.mFrameCount` 交给 AudioMixer。

### 3.3 underrun 的精确判定

```cpp
// Track::getNextBuffer() —— frameworks/av/services/audioflinger/Tracks.cpp
status_t status = mServerProxy->obtainBuffer(&buf);
buffer->frameCount = buf.mFrameCount;
buffer->raw = buf.mRaw;

if (buf.mFrameCount == 0 && !isStopping() && !isStopped()
        && !isPaused() && !isOffloaded()) {
    ALOGV("%s(%d): underrun, framesReady(%zu) < framesDesired(%zu)", ...);
    mAudioTrackServerProxy->tallyUnderrunFrames(desiredFrames);
} else {
    mAudioTrackServerProxy->tallyUnderrunFrames(0);   // 传 0 表示重置连续计数
}
```

三点值得注意：

1. **只有「正在播」的 Track 才计数**——暂停/停止/offload 期间取不到数据是正常的，不算 underrun。
2. `tallyUnderrunFrames(desiredFrames)` 累加的是**帧数**，dumpsys 里的 `Underruns` 是次数。
3. 典型日志长这样：`AF::Track: getNextBuffer(229): underrun, framesReady(0) < framesDesired(358)` —— 358 是这一周期线程期望的帧数，0 是 App 实际备好的量。

### 3.4 `write()` 的五步与两种模式

```
1. Java write(byte[]/short[]) → JNI 拿到 native AudioTrack*
2. writeToTrack() 分流
3. obtainBuffer()   → 在环形缓冲里找空闲区，返回地址
4. memcpy           → App 的 PCM 拷进共享内存
5. releaseBuffer()  → 推写指针，futex 唤醒睡着的 AudioFlinger
```

- `MODE_STREAM`：走 3→4→5 的环形写入循环，缓冲满了 App 睡、AF 读完叫醒
- `MODE_STATIC`：`memcpy` 整块拷完事，不阻塞（分块写只是因为数组大）

**关键**：这里只有**一次** memcpy，从 App 的 buffer 到共享内存。没有内核态、没有 Binder、没有第二次拷贝。

### 3.5 缓冲区该开多大

| 太小 | 太大 |
|-----|------|
| 延迟低，但极易 underrun（卡顿、爆音） | 稳定，但延迟不可接受、起播慢 |

- 下限：`AudioTrack.getMinBufferSize()`（**这是下限，不是推荐值**）
- 常规建议：2～4 倍 `getMinBufferSize()`
- 低延迟场景：`PERFORMANCE_MODE_LOW_LATENCY` + 小 buffer + 前台可见（否则 FAST 标志会被清掉，见 2.3）
- 省电场景：`PERFORMANCE_MODE_POWER_SAVING`（`DEEP_BUFFER`，>100ms）

## 四、阶段三：`start()` —— 挂进活跃列表

`play()` 就是 `start()`。整条路径：

```
Java:   AudioTrack.play()
  │
  ▼ JNI: android_media_AudioTrack_start()
  │
Native: AudioTrack::start()            [libaudioclient/AudioTrack.cpp]
  │  ① mState = STATE_ACTIVE
  │  ② 清位置计数：mPosition = 0, mFramesWritten = 0
  │  ③ 清 CBLK_STREAM_END_DONE / CBLK_DISABLED
  │  ④ mAudioTrack->start(&status)   ← Binder 跨进程
  │  ⑤ 线程优先级提到 ANDROID_PRIORITY_AUDIO
  │
  ▼ ═══════ Binder ═══════
  │
AudioFlinger: Track::start()           [audioflinger/Tracks.cpp]
  │  ① 取 PlaybackThread，加线程锁
  │  ② 状态迁移：FLUSHED→reset / PAUSED→RESUMING / 其它→ACTIVE
  │  ③ 重置位置与时间戳映射
  │  ④ addTrack_l(this)   ★★★
  │       AudioSystem::startOutput(portId)     ← 通知 AudioPolicy「我出声了」
  │       fillingStatus() = FS_FILLING         ← 标记「正在填充」
  │       mActiveTracks.add(track)
  │       onAddNewTrack_l()                    ← 唤醒 MixerThread
  │  ⑤ mAudioTrackServerProxy->start()         ← 启动读代理
```

### 4.1 为什么 `play()` 之后没有立刻出声

**这是本篇最值得记住的一条**：新 Track 进列表时状态是 `FS_FILLING`，MixerThread 每轮 `prepareTracks_l()` 会检查：

- `fillingStatus == FS_FILLING` 且 `framesReady < startThreshold` → **继续等待，不出声**
- `framesReady >= startThreshold` → 置 `FS_FILLED`，解除静音，**这时才开始真正出声**

所以「起播延迟」= 填到阈值的时间 + 混音周期，而不是纯的硬件延迟。几个直接推论：

1. **`play()` 之前必须先 `write()` 至少一个阈值量的数据**，否则 Track 要等 App 供数据，表现为「按了播放没反应」。
2. 阈值过大 → 起播慢；阈值过小 → 一开始就 underrun，噼啪响。
3. 短音效（`MODE_STATIC`）不受阈值影响，因为数据在 `play()` 前已经全在内存里。

### 4.2 Track 内部是 9 态，不是 Java 那 4 态

Java 层只暴露 `PLAYSTATE_STOPPED / PAUSED / PLAYING`，AudioFlinger 内部是 `TrackBase::track_state`：

```
IDLE → FLUSHED → ACTIVE → PAUSING → PAUSED → RESUMING → ACTIVE
                    │
                    └→ STOPPING_1 → STOPPING_2 → STOPPED
```

| 状态 | 含义 |
|-----|------|
| `IDLE` | 刚创建，未启动 |
| `FLUSHED` | 被 flush 过，位置清零，可重新起播 |
| `ACTIVE` | 在 `mActiveTracks` 中，正在被混音 |
| `PAUSING` / `PAUSED` | 正在停 / 已暂停（数据保留） |
| `RESUMING` | 从暂停恢复，等待重新对齐 |
| `STOPPING_1` / `STOPPING_2` | 两级停止：先等当前周期数据处理完，再清理 |
| `STOPPED` | 已停止（数据可能保留，取决于 stop 类型） |

**排查价值**：`dumpsys media.audio_flinger` 里看到 Track 停在 `PAUSING` 或 `STOPPING_1` 不走，通常是有别的线程持锁或混音周期卡住了。

## 五、阶段四：MixerThread 混音

`PlaybackThread::threadLoop()` 是一个常驻循环，每轮固定四件事：

```
for (;;) {
    ① prepareTracks_l(&tracksToRemove)
         遍历 mActiveTracks
           每个 Track 取 framesReady = track->framesReady()   // 读 cblk 算可读帧数
           FILLING 且够阈值 → FS_FILLED 解除静音
           有数据 → 设置 AudioMixer 参数（音量/采样率/格式/重采样器）
                  → mixerStatus = MIXER_TRACKS_READY
           无数据 → tracksToRemove.add(track)                  // 或计入 underrun

    ② threadLoop_mix()
         mAudioMixer->process()
         通过各 Track 的 ServerProxy 从共享内存读 PCM
         按各自音量缩放 → 采样率转换 → 声道映射 → 多路相加
         结果写入 mSinkBuffer（float 中间态在 mMixerBuffer，有音效链时过 mEffectBuffer）

    ③ threadLoop_write()
         mOutput->write((char*)mSinkBuffer + offset, bytes)   // Threads.cpp 约 3626 行

    ④ 没有活跃 Track → mWaitWorkCV.wait()   睡到 onAddNewTrack_l 唤醒
}
```

几个容易忽略的细节：

- **`mSinkBuffer` 是 AudioFlinger 进程内的普通堆内存**（`posix_memalign` 分配，32 字节对齐给 SIMD 用），不是共享内存。大小 ≈ `mNormalFrameCount * mFrameSize`。
- **混音是纯数学**：`out[i] = Σ (track_k[i] × gain_k)`。AudioMixer 用 NEON 做了 SIMD 优化，一个周期通常 1ms 内算完。
- **重采样在这里发生**：App 送 44.1kHz、设备跑 48kHz，就在 `AudioMixer` 里转。这是「为什么我的 44.1k 音乐听起来有点糊」的常见答案之一。
- **Track 启动/停止有短暂音量渐变**（ramp，约几毫秒），用来消 pop 音。所以「起播第一个采样」并不是满音量。
- **一个 App 一个 Track**：三个 App 同时播 = 三个 Track 挂在同一个 MixerThread 上，统一读、混、输出。

## 六、阶段五：到 HAL 与硬件

### 6.1 AIDL HAL 的三条 FMQ

Android 13 开始 Audio HAL 迁到 AIDL，Android 15 是主流形态。**PCM 数据不走 Binder**，走开流时建立的 Fast Message Queue。

`StreamHalAidl` 持有三条队列（`StreamHalAidl.h` 约 44-49 行）：

| 队列 | 方向 | 用途 |
|-----|------|------|
| `CommandMQ` | AudioFlinger → HAL | 命令：BURST（搬了 N 字节）/ DRAIN / FLUSH / PAUSE |
| `ReplyMQ` | HAL → AudioFlinger | 回执：状态码 + 新的可写量 |
| `DataMQ` | AudioFlinger → HAL | **PCM 数据本体**（共享内存环形队列） |

这三条队列的「钥匙」是 `StreamDescriptor`（`openOutputStream` 时随 Binder 句柄一起交付）：

```aidl
parcelable StreamDescriptor {
    MQDescriptor command;   // client 写，HAL 读
    MQDescriptor reply;     // HAL 写，client 读
    @nullable MQDescriptor fmq;   // PCM 数据
    MmapBufferDescriptor mmap;    // MMAP_NOIRQ 模式专用
    int frameSizeBytes;
    long bufferSizeFrames;
}
```

一次 `write` 在 `StreamHalAidl::write()` 里实际做的是：

```cpp
bytes = min(bytes, DataMQ->availableToWrite());
DataMQ->write(mSinkBuffer + offset, bytes);   // PCM 拷进共享 FMQ
sendCommand(burst{bytes}, &reply);            // CommandMQ 通知 HAL
// reply 从 ReplyMQ 读回新状态
```

**为什么不用 Binder 传数据**，算笔账：48kHz / 立体声 / 16bit ≈ 192 KB/s。若 5ms 一个周期，每秒 200 次 transaction，每次进内核 10~20µs，纯开销 2~4ms/秒；而 24bit / 192kHz / 6 声道 ≈ 3.4 MB/s，直接撞上 Binder 每进程 1MB 的 buffer 上限。

### 6.2 全链路：7 块内存，6 次搬运

这是理解「到底拷贝了几次」最清晰的一张图：

```
App buffer (Java byte[] / AAudio)
   │ ① memcpy（obtainBuffer / cblk）
AudioTrack 共享环形缓冲（audio_track_cblk_t + data，跨进程 ashmem）
   │ ② AudioMixer::process 混音写出
mSinkBuffer（AudioFlinger 进程堆内存）
   │ ③ memcpy 进 FMQ
AIDL DataMQ（AF 与 HAL 共享的环形队列）
   │ ④ memcpy（HAL worker 读 FMQ）
HAL 本地 buffer（mDataBuffer，worker 线程堆内存）
   │ ⑤ ioctl(WRITEI) → copy_from_user
Kernel ALSA dma_area（DMA 一致性内存 / ring buffer）
   │ ⑥ DMA 硬件搬运（零 CPU）
SAI FIFO → I2S → Codec DAC → 喇叭
```

| 编号 | 是否 CPU 拷贝 | 边界 |
|:---:|:---:|------|
| ① | ✅ memcpy | App 进程内 |
| ② | ❌ 混音计算，不是搬运 | AF 进程内 |
| ③ | ✅ memcpy | AF → FMQ（共享内存） |
| ④ | ✅ memcpy | HAL worker |
| ⑤ | ✅ copy_from_user | 用户态 → 内核态 |
| ⑥ | ❌ DMA | 纯硬件 |

**所以「零拷贝」要分范围说**：

- App → AudioFlinger：**真零拷贝**（同一块物理内存，只传 fd 和控制信令）
- AudioFlinger → HAL：**不是零拷贝**（memcpy 进 DataMQ）
- 全链路零 CPU 拷贝：只有 `MmapPlaybackThread` / AAudio MMAP 模式（App 直接写 `dma_area`，只同步指针）

### 6.3 内核侧：`snd_pcm_period_elapsed` 是整条链路的心跳

```
tinyalsa pcm_write()
  → ioctl(SNDRV_PCM_IOCTL_WRITEI_FRAMES)
  → 内核 __snd_pcm_lib_xfer() → do_transfer() → copy_from_user()
  → runtime->dma_area + hwoff
  → DMA 按 period 搬进 SAI FIFO
  → 每个 period 结束时 DMA 中断回调 snd_pcm_period_elapsed()
       ├─ 更新硬件读指针 hw_ptr
       ├─ 唤醒等待的写者（「可以再写了」）
       └─ 这个反压一路传回 AudioFlinger 的 threadLoop 节拍
```

**理解这一点，才能理解 underrun 的真正成因**：不是「App 写得太慢」这么简单，而是「从 DMA 中断反压上来的这一拍里，共享内存里没有数据」。所以 CPU 被抢占、App 线程调度延迟、解码太慢、网络卡顿，最后都表现为同一个 underrun。

## 七、传输模式与性能模式

### 7.1 五种 `transfer_type`

| 模式 | 谁驱动 | 典型使用者 |
|-----|-------|-----------|
| `TRANSFER_SYNC` | App 同步 `write()` 阻塞 | **Java `MODE_STREAM`**、OpenSL ES |
| `TRANSFER_CALLBACK` | AF 需要数据时回调客户端 | MediaPlayer / AudioSink、低延迟 Native |
| `TRANSFER_OBTAIN` | App 手动 `obtainBuffer()` / `releaseBuffer()` | 高级用法、需要精确控制时机 |
| `TRANSFER_SHARED` | 一次性共享整块数据 | **Java `MODE_STATIC`** |
| `TRANSFER_SYNC_NOTIF_CALLBACK` | 同步写 + 完成通知 | 混合场景（要写要通知） |

**关键差异**：`TRANSFER_SYNC` 里是「App 找 AF」，`TRANSFER_CALLBACK` 里是「AF 找 App」——后者不会因为 App 主线程被占用而 underrun，这也是 MediaPlayer 不用你手动喂数据的原因。

### 7.2 性能模式 → output flags

| Builder 设置 | 底层 flags | 线程 | 端到端延迟（经验值） |
|-------------|-----------|------|:---:|
| `PERFORMANCE_MODE_LOW_LATENCY` | `AUDIO_OUTPUT_FLAG_FAST` | FastTrack + FastMixer | ~5-10ms |
| 默认（`NONE`） | 无特殊标志 | MixerThread | ~40-50ms |
| `PERFORMANCE_MODE_POWER_SAVING` | `AUDIO_OUTPUT_FLAG_DEEP_BUFFER` | MixerThread（大 buffer） | >100ms |
| 压缩格式 / offload | `DIRECT \| COMPRESS_OFFLOAD` | OffloadThread | DSP 侧，较难估 |
| AAudio Exclusive | `DIRECT \| MMAP_NOIRQ` | MmapPlaybackThread | ~2-5ms |

> 延迟是**端到端**估计（App write → 喇叭出声），跟具体 HAL、buffer 周期、DSP 通路强相关，别拿它当保证值。

## 八、Android 15 的变化

对播放链路有实际影响的四条：

| 变更 | 影响 | 怎么应对 |
|-----|------|---------|
| **direct / offload 资源抢占**（所有应用） | 资源达到上限时请求 direct/offload 播放，系统会**主动废弃当前已打开的其他 direct/offload AudioTrack**。被废弃的一方 `write()` 开始返回错误 | 监听 `onMarkerReached` / write 返回负值，收到 `ERROR_DEAD_OBJECT` 要**重建 Track** |
| **音频焦点请求限制**（targetSdk 35+） | 必须是顶层应用或运行音频相关前台服务，否则 `requestAudioFocus()` 直接返回 `AUDIOFOCUS_REQUEST_FAILED` | 后台起播（如 `BOOT_COMPLETED`）必定失败，转前台服务；别再忽略返回值 |
| **bit-perfect 进入主流程** | `createTrack` 的响应里带 `isBitPerfect`，`dumpsys` 有 `BitPerfect` 列；走该通路时**跳过任何音量缩放和格式转换** | 做 Hi-Res 输出时检查该标志；它跟 `setVolume()` 是互斥的，设了音量就不再 bit-perfect |
| **`Virtualizer` 弃用 → 用 `Spatializer`** | 空间音频接口换血 | 旧音效代码要迁移，`setIsContentSpatialized` 走新通路 |

另外两个与播放相关的新 API：`LoudnessCodecController`（基于 CTA-2075 的响度一致性控制，切换内容不用手调音量）、`mediaProcessing` 前台服务类型（转码等场景，与 `dataSync` 共享 6h/24h 配额）。

## 九、时间戳链路（音视频同步必知）

`AudioTrack.getTimestamp()` 拿到的不是一个简单计数器，而是一条跨三层的链路：

```
App: getTimestamp()
  → IAudioTrack::getTimestamp()
  → Track 侧的 ServerProxy 位置（已消费帧数）
  + HAL 的 getPresentationPosition()（已到 DAC 的帧数 + 对应时刻）
  → TimeCheck 线程做漂移校正
       （HAL 时钟 ≠ kernel monotonic 时，按采样率估算偏移）
  → 返回 frames + nanoTime
```

排查音画不同步时的顺序：先确认 HAL 是否真的实现了 `getPresentationPosition`（很多参考 HAL 返回 0），再看 `TimeCheck` 有没有报「HAL and kernel time disagree」，最后才怀疑 App 侧。

## 十、调试速查

### 10.1 `dumpsys media.audio_flinger` 关键字段

```bash
adb shell dumpsys media.audio_flinger
```

```
Output thread 0x...  type 0 (MIXER)          ← 线程类型
  I/O handle: 13                              ← outputId
  Standby: no  Sample rate: 48000  HAL frame count: 192
  Normal frame count: 960                     ← 每周期混音帧数
  Tracks of which 1 are active
    Type  Id  Active  Client  Session  Port  S  Flags  Format  Chn  SRate
    ...
    Server FrmCnt  FrmRdy  Underruns  Flushed  BitPerfect
            7088      0       71188    false    false        ← ★ FrmRdy=0 且 Underruns 狂涨 = 卡顿
```

| 字段 | 含义 | 异常信号 |
|-----|------|---------|
| `FrmCnt` | 共享内存环形缓冲容量（帧） | 太小 → 容易 underrun |
| `FrmRdy` | 当前 App 已备好、未被消费的帧数 | 持续 0 → App 供数据不上 |
| `Underruns` | 累计欠载次数 | 单调上涨说明问题在持续 |
| `Latency` | 该 Track 的延迟估计 | 与预期差距大 → 检查是否被降级到深缓冲 |
| `BitPerfect` | 是否走无损通路 | 期望 true 却是 false → 被音量/重采样破坏 |
| `Standby: yes` | 线程已进待机 | 有 Track 活跃却 standby → 唤醒链路有问题 |

### 10.2 日志过滤

```bash
# 全链路关键 TAG
adb logcat -s AudioTrack:V AudioFlinger:V AudioMixer:V StreamHalAidl:V

# 只看 underrun / 卡顿
adb logcat | grep -iE "underrun|underflow|framesReady"

# Track 创建失败（BAD_VALUE / DEAD_OBJECT）
adb logcat | grep -iE "createTrack|ERROR_DEAD_OBJECT|BAD_VALUE"

# fast 标志被降级
adb logcat | grep -iE "fastAllowed|AUDIO_OUTPUT_FLAG_FAST"
```

### 10.3 四个高频问题的定位顺序

**问题 1：完全没声音**

```
① dumpsys 里有没有这个 Track？         → 没有：卡在 createTrack（看 2.4 哪步失败）
② Track 状态是 ACTIVE 还是 PAUSED？     → PAUSED：play() 没生效或被别处 pause
③ FrmRdy 是 0 吗？                      → 是：App 没写数据 / 写了但没 release
④ fillingStatus 一直 FS_FILLING？       → 是：没到 startThreshold，见 4.1
⑤ 线程在 standby 吗？                   → 是：唤醒失败，看 onAddNewTrack_l
```

**问题 2：卡顿 / 爆音**

```
① Underruns 在涨 + FrmRdy 频繁为 0  → 生产者跟不上
   常见原因：CPU 被抢占（top 看 stress-ng 之类）、解码慢、网络抖动、写线程优先级不够
   应对：写线程提到 THREAD_PRIORITY_URGENT_AUDIO，加大 buffer，改用回调模式
② FrmRdy 充足但仍 underrun        → 消费者侧问题（线程调度、HAL 阻塞）
③ 音频没 underrun 但听起来破       → 查重采样质量 / 音量越界削顶 / 格式不匹配
```

> 实测过的一个经典案例：`stress-ng --cpu 10 --cpu-load 100` 打满 CPU 后，dumpsys 显示 `Underruns 71188`、`FrmRdy 0`，`Latency 163.14k` —— 生产者被饿死，不是音频系统的问题。

**问题 3：起播延迟大**

```
① buffer 开太大                    → 降到 2~4 倍 minBufferSize
② startThreshold 太高              → 减小，或提前 write 预填
③ 被降级到 DEEP_BUFFER             → 查 flags 是否被清（见 2.3）
④ 没走 Fast 通道                   → PERFORMANCE_MODE_LOW_LATENCY + 前台可见 + 格式匹配设备原生
```

**问题 4：延迟忽大忽小 / 声音断续有规律**

```
① 查是否有别的 App 在抢音频焦点（dumpsys audio | grep -A 20 "Audio Focus"）
② 查路由是否在反复切换（插拔耳机、蓝牙连接抖动）→ dumpsys media.audio_policy
③ Perfetto 看 threadLoop 的周期是否稳定
```

### 10.4 Perfetto 抓播放链路

```bash
adb shell perfetto -c - --txt -o /data/misc/perfetto-traces/audio.pftrace <<EOF
buffers: { size_kb: 16384 fill_policy: RING_BUFFER }
data_sources: {
  config {
    name: "linux.ftrace"
    ftrace_config {
      ftrace_events: "snd/snd_pcm_period_elapsed"
      atrace_categories: "audio"
    }
  }
}
duration_ms: 5000
EOF
```

看三件事：`threadLoop` 周期的抖动、`getNextBuffer` 是否返回 0、DMA period 中断是否规律。三者对齐，卡顿原因就基本锁定了。

## 十一、源码路径索引

> 行号以 AOSP `android15-release` 分支为参考，不同 tag / 厂商分支会有偏移，按函数名检索更稳。

| 组件 | 路径 |
|-----|------|
| Java API | `frameworks/base/media/java/android/media/AudioTrack.java` |
| JNI 桥接 | `frameworks/base/core/jni/android_media_AudioTrack.cpp` |
| Native 客户端 | `frameworks/av/media/libaudioclient/AudioTrack.cpp`（`set()` / `start()` / `write()`） |
| 共享内存与代理 | `frameworks/av/media/libaudioclient/include/media/AudioTrackShared.h` |
| Proxy 实现 | `frameworks/av/media/libaudioclient/AudioTrackShared.cpp` |
| createTrack 服务端 | `frameworks/av/services/audioflinger/AudioFlinger.cpp` |
| Track / TrackBase | `frameworks/av/services/audioflinger/Tracks.cpp`（TrackBase 构造约 84 行） |
| PlaybackThread | `frameworks/av/services/audioflinger/Threads.cpp`（`threadLoop_write` 约 3626 行） |
| 混音器 | `frameworks/av/services/audioflinger/AudioMixer.cpp` |
| FastMixer | `frameworks/av/services/audioflinger/fastpath/FastMixer.cpp` |
| AIDL HAL 客户端 | `frameworks/av/media/libaudiohal/impl/StreamHalAidl.cpp` / `.h` |
| HAL 接口定义 | `hardware/interfaces/audio/aidl/android/hardware/audio/core/IStreamOut.aidl` |
| Socket / FMQ 定义 | `hardware/interfaces/audio/aidl/android/hardware/audio/core/StreamDescriptor.aidl` |
| tinyalsa | `external/tinyalsa/pcm.c` |
| ALSA 内核 | `sound/core/pcm_lib.c`（`__snd_pcm_lib_xfer` / `snd_pcm_period_elapsed`） |

## 十二、六个常见误区

| 误区 | 事实 |
|-----|------|
| 「App 到喇叭全程零拷贝」 | 只有 App → AF 是零拷贝；AF → HAL 至少还有 2 次 memcpy（③④）+ 1 次 copy_from_user（⑤） |
| 「`play()` 之后立刻出声」 | 要等 fill 到 startThreshold 才解除静音，起播延迟主要来自这里 |
| 「`getMinBufferSize()` 是推荐 buffer」 | 它是**下限**。按它开，稍微一忙就 underrun |
| 「`MODE_STATIC` 是把数据预加载进 ring buffer」 | 服务端读取逻辑完全不同，不统计 underrun，支持零 CPU 循环 |
| 「请求了 LOW_LATENCY 就一定低延迟」 | 非线性 PCM / offload / 非前台可见 / 格式不匹配设备原生，任何一种都会清掉 `FAST` 标志，且**不报错** |
| 「Java 的 PLAYSTATE 就是 Track 状态」 | Java 只有 3 态，AudioFlinger 内部 9 态，排障要看后者 |

## 十三、与其他技能的关系

| 技能 | 关系 |
|-----|------|
| `android-audioflinger-architecture` | 本篇是它在**播放方向**的纵向展开。线程类型、混音策略、延迟分级在那篇有全景 |
| `android-audio-call-chain` | 录音方向的对照篇。Track 结构与共享内存机制对称，但数据流向相反 |
| `android-car-audio-policy` | Track 创建第二步 `getOutputForAttr()` 的决策细节在那篇（策略 → 设备 → output） |
| `android-car-audio-focus` | 焦点决定了「谁该出声」，本篇讲「怎么出声」 |
| AAudio MMAP 路径（待写） | 本篇讲到 `IStreamOut::write` 为止；MMAP 模式绕开 ①~⑤，是另一条独立链路 |

## 十四、总结

1. **五个阶段**：创建（Binder + 路由选线程）→ 分配共享内存（`TrackBase`）→ 写（ClientProxy + futex）→ 起播（`addTrack_l` + startThreshold）→ 混音出硬件（MixerThread + FMQ + DMA）。
2. **一个核心设计**：控制流走 Binder，数据流走共享内存。Binder 只负责建链路、传 fd、改参数，一次都不搬运 PCM。
3. **一次关键分配**：服务端 `TrackBase` 把控制块和音频数据拼成**同一块**共享内存，两个进程 mmap 后看到同一份指针。
4. **一个隐藏延迟源**：`FS_FILLING → FS_FILLED` 的 startThreshold 判定，比硬件延迟更容易被忽略。
5. **一条真实链路**：7 块内存、6 次搬运，其中 4 次 CPU 拷贝、1 次 DMA 零 CPU。理解这条链，才知道 underrun 该往哪一层查。
6. **Android 15 要记住的**：direct/offload 会被系统抢占失效（必须处理 `ERROR_DEAD_OBJECT` 重建）、targetSdk 35 后台不能请求焦点、bit-perfect 标志已进主流程。

*本文基于 Android 15 (API Level 35) 源码分析，不同厂商的定制 ROM 可能存在差异。*
