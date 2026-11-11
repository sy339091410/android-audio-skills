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

Builder 不显式设置的默认值（`AudioTrack.Builder.build()` 实测行为，很容易踩坑）：

| 属性 | 缺省值 | 在哪生效 |
|-----|--------|---------|
| AudioAttributes | `USAGE_MEDIA` | `build()` 里补 |
| AudioFormat | `CHANNEL_OUT_STEREO` + `ENCODING_DEFAULT`（即 16bit PCM），采样率 `SAMPLE_RATE_UNSPECIFIED` 由设备决定 | `build()` 里补 |
| bufferSizeInBytes | **先填「1 帧的字节数」**，再由 native 换算成最小缓冲 | `build()` 里补 |
| transferMode | `MODE_STREAM` | 字段声明时初始化 |
| sessionId | `AUDIO_SESSION_ID_GENERATE` | 字段声明时初始化 |
| performanceMode | `PERFORMANCE_MODE_NONE` | 字段声明时初始化 |

> 注意 buffer 那一行：源码注释写的是 "use a single frame for the buffer size and let the native code figure out the minimum buffer size"。它不是「自动取 `getMinBufferSize()`」，而是先塞 1 帧，让 `PlaybackThread::createTrack_l()` 用 `calculateMinFrameCount()` 算最小值。所以不设 buffer 时，你拿到的大小由系统决定，不是你能预期的值。

**旧构造器（已废弃但仍兼容）会做一次隐式翻译**：`AudioTrack(int streamType, ...)` → `setLegacyStreamType(streamType)` → `usageForStreamType()` 的 switch 映射（`STREAM_MUSIC → USAGE_MEDIA`、`STREAM_VOICE_CALL → USAGE_VOICE_COMMUNICATION`、`STREAM_ALARM → USAGE_ALARM`…）。

⚠️ **Android 15 的一个行为变化**：从 streamType 推导 AudioAttributes 时，**不再从策略里读 contentType**，且 `STREAM_MUSIC` 的 contentType 保持 `CONTENT_TYPE_UNKNOWN`。源码里这两处注释很直白：

```java
// on purpose ignoring the content type: stream types are deprecated for
// playback, making assumptions about the content type is prone to
// interpretation errors for ambiguous types such as STREAM_TTS and STREAM_MUSIC
//mContentType = attributes.mContentType;

case AudioSystem.STREAM_MUSIC:
    // leaving as CONTENT_TYPE_UNKNOWN
    break;
```

所以以前写死 `CONTENT_TYPE_MUSIC` 的代码，在 dumpsys 里会看到 `content=CONTENT_TYPE_UNKNOWN`，**这不是 bug**。其余分支（`STREAM_VOICE_CALL → SPEECH`、`STREAM_RING/ALARM/NOTIFICATION → SONIFICATION`、`STREAM_TTS` 额外加 `FLAG_BEACON`）仍按老规则填。

### 2.2 JNI 层：参数过河

`frameworks/base/core/jni/android_media_AudioTrack.cpp` 里的 `android_media_AudioTrack_setup()`：

- `AudioAttributes` → `audio_attributes_t*`（身份：usage / contentType / flags / tags）
- `int[]` → 逐项拆成 sampleRate / channelMask / format / sessionId
- `Parcel` → `AttributionSourceState`（调用者身份，用于权限与归因）
- 按 `MODE_STREAM` / `MODE_STATIC` 二选一（源码实测）：
  - `MODE_STREAM`：`sharedBuffer = 0`（交给服务端分配）、`transferType = TRANSFER_SYNC`（offload 时是 `TRANSFER_SYNC_NOTIF_CALLBACK`）、`threadCanCallJava = true`
  - `MODE_STATIC`：先 `allocSharedMem(buffSizeInBytes)` 自己分配、`transferType = TRANSFER_SHARED`

> `threadCanCallJava = true` 这个参数不是随手写的——它是后面 `TRANSFER_SYNC` 能否申请 FAST 通道的必要条件（见 2.3 第 ② 关）。Native 侧自己建 AudioTrack 时如果传 false，FAST 会被静默清掉。

### 2.3 Native 层：`AudioTrack::set()` 里的那处硬改写

`frameworks/av/media/libaudioclient/AudioTrack.cpp`，`AudioTrack::set()`：

```cpp
if ((flags & AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) || !audio_is_linear_pcm(format)) {
    flags = (flags | AUDIO_OUTPUT_FLAG_DIRECT) & ~AUDIO_OUTPUT_FLAG_FAST;
}
```

**这一句是「为什么我的 Track 没走 Fast 通道」的第一现场**：只要格式不是线性 PCM，或者带 offload 标志，系统会强制加上 `DIRECT` 并且**清掉 `FAST`**。你在 App 里请求低延迟，底层直接被降级，而且不会报错。

**FAST 一共要过三道关**，全部通过才真正跑在 FastTrack 上。任何一道不过，标志被静默清掉：

| 关卡 | 位置 | 条件 |
|:---:|------|------|
| ① 格式 | `AudioTrack::set()` | 必须是线性 PCM 且没请求 offload，否则强制 `DIRECT` 并清 `FAST` |
| ② 客户端 | `AudioTrack::createTrack_l()` | `fastAllowed = sharedBuffer \|\| transferAllowed`；`transferAllowed` = 回调模式 / obtain 模式 / 同步写且 `mThreadCanCallJava`。不满足打 `AUDIO_OUTPUT_FLAG_FAST denied by client` |
| ③ 服务端 | `PlaybackThread::createTrack_l()` | 五个条件全要满足：线性 PCM、声道无需昂贵下混、`sampleRate == mSampleRate`（**App 采样率必须等于设备原生采样率**）、`hasFastMixer()`、`mFastTrackAvailMask != 0`（还有空槽位）。不满足打 `AUDIO_OUTPUT_FLAG_FAST denied: ...` |

```cpp
// 第 ③ 关的拒绝日志格式（字段齐了就知道卡在哪），按 format string 还原：
// AUDIO_OUTPUT_FLAG_FAST denied: sharedBuffer=0x0 frameCount=960 mFrameCount=192
//   format=0x1 mFormat=0x1 isLinear=1 channelMask=0x3 sampleRate=44100 mSampleRate=48000
//   hasFastMixer=1 tid=1234 fastTrackAvailMask=0x0
//   ↑ 采样率不匹配 + 槽位用尽，两个原因同时命中
```

另外 `sharedBuffer != 0`（即 `MODE_STATIC`）会让第 ② 关直接通过——静态 Track 天生满足条件。

还有一处容易忽略：**效果链也会撤销 FAST**。`createTrack_l()` 里会遍历 `AUDIO_SESSION_DEVICE / OUTPUT_STAGE / OUTPUT_MIX / 本 session` 的 EffectChain 调 `checkOutputFlagCompatibility()`，挂了下混或重采样类效果，标志就没了。

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

### 2.4 AudioFlinger 侧：`createTrack()` 的 15 步

`frameworks/av/services/audioflinger/AudioFlinger.cpp`，`AudioFlinger::createTrack()`：

```
 1. 解包 CreateTrackRequest（AIDL → CreateTrackInput）
 2. 归属校验 checkAttributionSourcePackage / validateAttributionFromContextOrTrustedCaller
 3. sessionId == AUDIO_SESSION_ALLOCATE 时 newAudioUniqueId() 分配
 4. AudioSystem::getOutputForAttr()
      → 路由决策，拿到 output.outputId / selectedDeviceId / portId / secondaryOutputs
      → 同时返回 isSpatialized / isBitPerfect（Android 15 已进入主流程）
 5. 校验 streamType / channelMask / format
 6. 加 AF 主锁，checkPlaybackThread_l(output.outputId) → 拿到已存在的播放线程
 7. registerClient(pid, uid) → Client（按进程复用，wp 弱引用防环）
 8. 扫 mPlaybackThreads 找同 sessionId 的 EFFECT_SESSION，命中记下 effectThread
 9. 没找到则 getOrphanEffectChain_l(sessionId) → 孤儿效果链
10. thread->createTrack_l(...)  ★ 真正创建 Track
11. 回填线程参数：afFrameCount / afSampleRate / afChannelMask / afFormat / afLatencyMs
12. 加线程锁，updateSecondaryOutputsForTrack_l() → 连接辅助输出（多音区/并发投递）
13. moveEffectChain_ll() → 把效果链搬到本线程（有条件，最多调两次）
14. 处理 mPendingSyncEvents；带 AUDIO_OUTPUT_FLAG_HW_AV_SYNC 时 setAudioHwSyncForSession_l
15. output.audioTrack = IAfTrack::createIAudioTrackAdapter(track) → 返回给客户端
```

**失败路径容易被忽略**：以上任何一步 `goto Exit`，最终都会执行 `AudioSystem::releaseOutput(portId)` 把端口还回去。如果日志里看到 createTrack 失败后 port 没释放、或者焦点状态卡住，先确认是不是走岔了清理逻辑。

**第 6 步是关键认知**：AudioFlinger 并不会为每个 Track 新开线程。线程是 `openOutput_l()` 时就按 output 建好的（`MixerThread` / `DirectOutputThread` / `OffloadThread` / `MmapPlaybackThread`），createTrack 只是把 Track 挂到一个**已经在跑的线程**上。

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

前面这些步骤都只是把 `sharedBuffer` 这个参数往下传。**真正分配共享内存的是 `TrackBase` 构造函数**（`frameworks/av/services/audioflinger/Tracks.cpp`，约 84 行起）。

```cpp
// 第 1 步：算大小
size_t size = sizeof(audio_track_cblk_t);        // 控制块
if (buffer == NULL && alloc == ALLOC_CBLK) {     // STREAM + 普通混音
    size += bufferSize;                          // 控制块和数据区拼成一块
}

// 第 2 步：从 Client 的内存分配器分配（按进程管理）
// Android 15 的分配请求带名字，dumpsys meminfo / libmeminfo 里能认出是谁
mCblkMemory = client->allocator().allocate(
        mediautils::NamedAllocRequest{{size},
                std::string("Track ID: ").append(std::to_string(mId))});
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

定义在 `frameworks/av/include/private/media/AudioTrackShared.h`（`private/media/`，不是 `libaudioclient/include/media/`，找文件时容易走错）。

| 字段 | 类型 | 作用 |
|-----|------|------|
| `mServer` | `uint32_t` | 服务端已消费的帧数。异步更新，源码注释直言「**for entertainment purposes only**」——不能拿它做重要决策 |
| `mFutex` | `volatile int32_t` | 等待/唤醒事件标志；`CBLK_FUTEX_WAKE` 位表示有延迟唤醒待处理 |
| `mMinimum` | `uint32_t` | 服务端唤醒客户端的门限（可用帧数 ≥ `mMinimum` 就通知） |
| `mVolumeLR` | `gain_minifloat_packed_t` | 打包的左右声道增益，**minifloat 格式，不是定点数** |
| `mSampleRate` | `uint32_t` | 客户端请求的采样率，0 表示默认 |
| `mPlaybackRateQueue` | `SingleStateQueue::Shared` | 变速播放（`AudioPlaybackRate`）的异步通道 |
| `mSendLevel` | `uint16_t` | 发送电平（U4.12 定点，仅 remote submix 等少数场景） |
| `mExtendedTimestampQueue` | `SingleStateQueue::Shared` | 服务端 → 客户端的时间戳队列，`getTimestamp()` 的数据源 |
| `mBufferSizeInFrames` | `volatile uint32_t` | **有效**缓冲大小，可被 `setBufferSizeInFrames()` 缩小 |
| `mStartThresholdInFrames` | `volatile uint32_t` | 起播门限：缓冲里至少多少帧才开始流（见 4.1） |
| `mFlags` | `volatile int32_t` | `CBLK_*` 组合 |
| `mState` | `std::atomic<int32_t>` | `TrackBase` 状态的镜像，源码用 `static_assert` 保证与 `IDLE / ACTIVE / PAUSING` 三个值一致 |
| `u` | `union { mStreaming, mStatic, mAlign[8] }` | 流式/静态模式的专属区，**`mFront` / `mRear` 读写指针就在 `u.mStreaming` 里** |

**三个最常被写错的点**（很多二手资料在这儿失真）：

1. `mFront` / `mRear` **不是 `audio_track_cblk_t` 的顶层成员**，在 `u.mStreaming` 联合体里。
2. `mFrameCount` / `mFrameSize` **不在控制块里**，是 `Proxy` 与 `TrackBase` 的成员。控制块里管容量的是 `mBufferSizeInFrames`。
3. 进度用 `mServer` 表示，但它是「仅供参考」的——精确位置要走时间戳队列。

`mFlags` 的位定义（源码 `CBLK_*`，排障时会直接看到这些名字）：

| 标志 | 值 | 含义 |
|-----|:--:|------|
| `CBLK_UNDERRUN` | `0x01` | 服务端一发生欠载就置位，客户端清除 |
| `CBLK_FORCEREADY` | `0x02` | 置位＝立即认为 Track 就绪（不等缓冲填满）；清零＝填满才就绪 |
| `CBLK_INVALID` | `0x04` | 缓冲被 AudioFlinger 判定无效，**需要重建 Track** |
| `CBLK_DISABLED` | `0x08` | 连续大量欠载后被禁用，需要 restart（不是立刻置位） |
| `CBLK_LOOP_CYCLE` | `0x20` | 静态 Track 每完成一个非最后一次循环 |
| `CBLK_LOOP_FINAL` | `0x40` | 最后一次循环完成 |
| `CBLK_BUFFER_END` | `0x80` | 非循环模式播到缓冲末尾 |
| `CBLK_OVERRUN` | `0x100` | 录音方向的上溢 |
| `CBLK_INTERRUPT` | `0x200` | 客户端调用 `interrupt()` |
| `CBLK_STREAM_END_DONE` | `0x400` | 服务端渲染完成 |

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
    ALOGV("%s(%d): underrun,  framesReady(%zu) < framesDesired(%zd), state: %d",
          __func__, mId, buf.mFrameCount, desiredFrames, (int)mState);
    mAudioTrackServerProxy->tallyUnderrunFrames(desiredFrames);
    // 同时打一条 ATRACE 事件：EVENT=UNDERRUN, FRAMECOUNT=desiredFrames
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

**服务端会二次校正你传的 frameCount**（`PlaybackThread::createTrack_l()`），这点很少有人提：

```cpp
if (sharedBuffer == 0) {                 // 流式模式
    pthread_once(&sFastTrackMultiplierOnce, sFastTrackMultiplierInit);
    frameCount = max(frameCount, mFrameCount * sFastTrackMultiplier);
}
...
// 非 FAST：按延迟和正常混音帧数算出下限
minFrameCount = AudioSystem::calculateMinFrameCount(latencyMs, mNormalFrameCount,
                                                    mSampleRate, sampleRate, speed);
if (frameCount < minFrameCount) frameCount = minFrameCount;
```

源码注释把规则说得很直白：

> Buffer depth is forced to be at least 2 x the normal mixer frame count and cover audio hardware latency. This is probably too conservative, but legacy application code may depend on it. **If you change this calculation, also review the start threshold which is related.**

也就是说：

- **普通 Track**：缓冲下限由 HAL 上报的 latency 和 `mNormalFrameCount` 共同决定，**至少是正常混音周期帧数的 2 倍**。你传更小的值会被顶上去。
- **FAST Track**：走 `mFrameCount * sFastTrackMultiplier`（`sFastTrackMultiplier` 由系统属性配置），路径不同。
- 这条计算和**起播门限**是联动的（见 4.1）——调 buffer 时两者要一起看。

实践建议：

- 下限参考 `AudioTrack.getMinBufferSize()`（**这是下限，不是推荐值**）
- 常规取 2～4 倍 `getMinBufferSize()`
- 低延迟：`PERFORMANCE_MODE_LOW_LATENCY` + 小 buffer + **采样率与设备原生一致**（否则第 ③ 关直接拒绝，见 2.3）
- 省电：`PERFORMANCE_MODE_POWER_SAVING`（`DEEP_BUFFER`，>100ms）

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

**这是本篇最值得记住的一条，而且实际机制和很多文章的写法不一样。**

第一半在 `PlaybackThread::addTrack_l()`，进列表时按「有没有共享缓冲」决定填充状态：

```cpp
track->fillingStatus() = track->sharedBuffer() != 0 ? FS_FILLED : FS_FILLING;
```

- `MODE_STATIC`（`sharedBuffer != 0`）→ 直接 `FS_FILLED`，**立刻可播**
- `MODE_STREAM` → `FS_FILLING`，要等填够

第二半在 `Track::isReady()`（注意是 `Track` 的重写，不是 `TrackBase`）：

```cpp
if (mFillingStatus != FS_FILLING || isStopped() || isPausing()) return true;

size_t bufferSizeInFrames = mServerProxy->getBufferSizeInFrames();
const size_t startThresholdInFrames = mServerProxy->getStartThresholdInFrames();
const size_t framesToBeReady = std::clamp(
        std::min(startThresholdInFrames, bufferSizeInFrames), size_t(1), mFrameCount);

if (framesReady() >= framesToBeReady || (mCblk->mFlags & CBLK_FORCEREADY)) {
    mFillingStatus = FS_FILLED;
    android_atomic_and(~CBLK_FORCEREADY, &mCblk->mFlags);
    return true;
}
return false;
```

关键在 `framesToBeReady` 的三层夹取：**取 `startThreshold` 和当前 `bufferSizeInFrames` 的较小值，再夹到 `[1, mFrameCount]`**。而 `startThreshold` 的来源是创建时传入的 `frameCountToBeReady`——普通播放 Track 传的是 `SIZE_MAX`，`IAfTrack::create()` 的注释写明了默认语义：

> default behaviour is to start when there are as many frames ready as possible (aka. **Buffer is full**).

**所以默认行为是「缓冲区填满才起播」**，不是填一点就播。这一条能解释掉很多现象：

1. **起播延迟 ≈ 一个 buffer 的时长 + 混音周期**。buffer 开大，延迟必然大——不是硬件慢。
2. **`play()` 之前必须先 `write()`**，否则 Track 永远等不到 `framesReady >= 阈值`，表现为「按了播放没反应」。
3. 想压低起播延迟，正确做法是调小起播门限（Java 侧 `AudioTrack.Builder.setStartThresholdInFrames()`，服务端对应 `mCblk->mStartThresholdInFrames`），而不是一味缩 buffer。
4. 短音效用 `MODE_STATIC` 完全绕过这个机制——`FS_FILLED` 且数据已在内存。
5. 源码注释提醒 `isReady()` **不要在 fast track 上调用**，因为 `framesReady()` 可能造成优先级反转。

### 4.1.1 `start()` 也可能被拒

`addTrack_l()` 的返回值不是永远 OK，`INVALID_OPERATION / PERMISSION_DENIED / DEAD_OBJECT` 都会导致起播失败，且 `Track::start()` 会把状态**回滚**：

```cpp
status = playbackThread->addTrack_l(this);
if (status == INVALID_OPERATION || status == PERMISSION_DENIED || status == DEAD_OBJECT) {
    triggerEvents(AudioSystem::SYNC_EVENT_PRESENTATION_COMPLETE);
    if (status == PERMISSION_DENIED || status == DEAD_OBJECT) {
        mState = state;   // 恢复成调用前的状态
    }
}
```

`PERMISSION_DENIED` 常见于焦点/静音策略拦截或 AppOps 限制；`DEAD_OBJECT` 通常是 audioserver 重启过、Track 已失效。**这两种情况下 App 的 `play()` 不会抛异常**，只能靠 `getPlayState()` 或 write 返回负值发现。

### 4.2 Track 内部是 10 态，不是 Java 那 3 态

Java 层只暴露 `PLAYSTATE_STOPPED / PAUSED / PLAYING` 三态；AudioFlinger 的 `track_state` 有 10 个值（录音 Track 另有 `STARTING_1 / STARTING_2`，播放用不到）。

```
IDLE
 ├─→ ACTIVE ─→ PAUSING ─→ PAUSED ─→ RESUMING ─→ ACTIVE
 │      │
 │      └─→ STOPPING_1 ─→ STOPPING_2 ─→ STOPPED
 │
 ├─→ FLUSHED          （flush 后位置清零，可重新起播）
 └─→ TERMINATED       （终态，资源回收）
```

| 状态 | 含义 | 判定函数 |
|-----|------|---------|
| `IDLE` | 刚创建，未启动 | — |
| `FLUSHED` | 被 flush 过，位置清零 | `isStopped()` **返回 true** |
| `STOPPED` | 已停止 | `isStopped()` **返回 true** |
| `ACTIVE` | 在 `mActiveTracks` 中，正在被混音 | — |
| `PAUSING` / `PAUSED` | 正在停 / 已暂停（数据保留） | `isPausing()` |
| `RESUMING` | 从暂停恢复，等待重新对齐 | — |
| `STOPPING_1` / `STOPPING_2` | 两级停止：先等当前周期数据处理完，再清理（Fast/Offload 专用） | `isStopping()` |
| `TERMINATED` | 终态 | `isTerminated()` |

两个和排障强相关的细节：

1. **`isStopped()` 把 `FLUSHED` 也算作「停止」**，`isStopping()` 只认 `STOPPING_1/2`。所以看到 FLUSHED 不要以为是异常状态。
2. **`dumpsys` 里显示的是字母缩写**（`getTrackStateAsCodedString()`）：`I`=IDLE、`A`=ACTIVE、`p`=PAUSING、`P`=PAUSED、`r`=RESUMING、`S`=STOPPED、`F`=FLUSHED、`s1`/`s2`=STOPPING、`T`=TERMINATED。大写是稳态、小写是瞬态——**看到 `p`/`r`/`s1` 长时间不变，就是卡在迁移过程里**。
3. 控制块 `mState` 只镜像其中三个（`IDLE=0`、`ACTIVE=6`、`PAUSING=7`），源码用 `static_assert` 锁死了对应关系。想从共享内存侧读状态，只有这三个值可信。

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

| 模式 | 谁驱动 | 典型使用者 | 能否拿 FAST |
|-----|-------|-----------|:---:|
| `TRANSFER_SYNC` | App 同步 `write()` 阻塞 | **Java `MODE_STREAM`**、OpenSL ES | 需 `threadCanCallJava` |
| `TRANSFER_CALLBACK` | AF 需要数据时回调客户端 | MediaPlayer / AudioSink、低延迟 Native | ✅ |
| `TRANSFER_OBTAIN` | App 手动 `obtainBuffer()` / `releaseBuffer()` | 高级用法、需要精确控制时机 | ✅ |
| `TRANSFER_SHARED` | 一次性共享整块数据 | **Java `MODE_STATIC`** | ✅（有共享缓冲） |
| `TRANSFER_SYNC_NOTIF_CALLBACK` | 同步写 + 完成通知 | **Java offload**、压缩流播放 | 需 `threadCanCallJava` |

**Java 层的实际映射**（`android_media_AudioTrack.cpp`，这段是判断「我的 Track 到底什么模式」的唯一依据）：

| Java `MODE_*` | transfer_type | sharedBuffer |
|--------------|---------------|-------------|
| `MODE_STREAM`（普通） | `TRANSFER_SYNC` | `0`（服务端分配） |
| `MODE_STREAM`（offload） | `TRANSFER_SYNC_NOTIF_CALLBACK` | `0` |
| `MODE_STATIC` | `TRANSFER_SHARED` | `allocSharedMem()` 的结果 |

**关键差异**：`TRANSFER_SYNC` 里是「App 找 AF」，`TRANSFER_CALLBACK` 里是「AF 找 App」——后者不会因为 App 主线程被占用而 underrun，这也是 MediaPlayer 不用你手动喂数据的原因。

注意 `MODE_STREAM` 默认是 `TRANSFER_SYNC`，而 `TRANSFER_SYNC` 想拿 FAST 通道必须满足 `mThreadCanCallJava`（JNI 里固定传 `true`，所以 Java Track 不受影响；**Native 侧自己建 AudioTrack 时如果传 false，FAST 会被静默清掉**）。

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
| **bit-perfect 进入主流程** | `createTrack` 的响应里带 `isBitPerfect`，`dumpsys` 有 `BitPerfect` 列；走该通路时跳过音量缩放和格式转换 | 做 Hi-Res 输出时检查该标志；**该 session 上不能挂数据处理类 effect** |

bit-perfect 有个硬性副作用值得单独记一笔：`createTrack_l()` 会拿 `AUDIO_OUTPUT_FLAG_BIT_PERFECT` 去过一遍 EffectChain 兼容性检查，不通过就**直接创建失败**，不是降级：

```cpp
if (isBitPerfect) {
    audio_output_flags_t flagsToCheck = *flags & AUDIO_OUTPUT_FLAG_BIT_PERFECT;
    chain->checkOutputFlagCompatibility(&flagsToCheck);
    if ((flagsToCheck & AUDIO_OUTPUT_FLAG_BIT_PERFECT) == AUDIO_OUTPUT_FLAG_NONE) {
        ALOGE("%s cannot create track as there is data-processing effect attached to "
              "given session id(%d)", __func__, sessionId);
        lStatus = BAD_VALUE;
        goto Exit;    // ← Track 建不出来
    }
    *flags = flagsToCheck;
}
```

所以「开了 Hi-Res 反而没声音 / createTrack 返回 -22」，先查这个 session 上是不是还挂着均衡器或音效。
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
④ 没走 Fast 通道                   → 过 2.3 的三道关：格式线性 PCM、采样率等于设备原生、有空闲 fast 槽位
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
| 共享内存与代理 | `frameworks/av/include/private/media/AudioTrackShared.h` |
| Proxy 实现 | `frameworks/av/media/libaudioclient/AudioTrackShared.cpp` |
| Track 状态机与基类 | `frameworks/av/services/audioflinger/TrackBase.h` |
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

## 十二、七个常见误区

| 误区 | 事实 |
|-----|------|
| 「App 到喇叭全程零拷贝」 | 只有 App → AF 是零拷贝；AF → HAL 至少还有 2 次 memcpy（③④）+ 1 次 copy_from_user（⑤） |
| 「`getMinBufferSize()` 是推荐 buffer」 | 它是**下限**。按它开，稍微一忙就 underrun；而且服务端还会按 `calculateMinFrameCount()` 再顶高 |
| 「`MODE_STATIC` 是把数据预加载进 ring buffer」 | 服务端读取逻辑完全不同，不统计 underrun，支持零 CPU 循环 |
| 「请求了 LOW_LATENCY 就一定低延迟」 | 要过三道关：非线性 PCM/offload 强制 `DIRECT`、采样率必须等于设备原生、fast 槽位得有剩余。任何一道不过都会清掉 `FAST`，而且**不报错** |
| 「`play()` 之后很快就出声」 | 默认起播门限是「缓冲区填满」——创建时传 `SIZE_MAX`，被夹取成 `mFrameCount`，起播延迟 ≈ 一个 buffer 时长 |
| 「`mFrameCount` 在控制块里」 | 控制块里管容量的是 `mBufferSizeInFrames`；`mFrameCount` / `mFrameSize` 是 `Proxy` 与 `TrackBase` 的成员 |
| 「Java 的 PLAYSTATE 就是 Track 状态」 | Java 只有 3 态，AudioFlinger 内部 10 态（含 `TERMINATED`），排障要看后者 |

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

## 附录：核对记录

本版本对以下结论做了逐条源码校验，基准是 `frameworks/av` / `frameworks/base` / `hardware/interfaces` 的 Android 15 分支，并与 Android 15 官方行为变更文档交叉比对。

**已逐字核对（引用的代码、字段、枚举与源码一致）**

| 结论 | 所在位置 |
|-----|---------|
| `set()` 的 `DIRECT/FAST` 强制改写、`createTrack_l()` 的 `fastAllowed` 三段条件 | 2.3 |
| `size = sizeof(cblk) + bufferSize`、`mBuffer = mCblk + sizeof(cblk)`、`NamedAllocRequest` | 2.5 |
| `getNextBuffer()` 的 underrun 判定、`tallyUnderrunFrames` | 3.3 |
| `isReady()` 的 `framesToBeReady` 三层夹取、`addTrack_l()` 的 `FS_FILLING/FS_FILLED` 赋值、`start()` 的失败回滚 | 4.1 / 4.1.1 |
| `minFrameCount` 计算与「2× normal frame count」注释 | 3.5 |
| cblk 全部字段、`CBLK_*` 位定义、`track_state` 枚举与 dumpsys 字母码 | 3.1 / 4.2 |
| `createTrack()` 的执行顺序、失败回滚 `releaseOutput` | 2.4 |
| JNI：`MODE_STREAM → TRANSFER_SYNC`、offload → `TRANSFER_SYNC_NOTIF_CALLBACK`、`MODE_STATIC → TRANSFER_SHARED` | 2.2 / 7.1 |
| `Builder` 默认值、`AudioAttributes` 的 `CONTENT_TYPE_UNKNOWN` 处理 | 2.1 |
| 官方行为变更：direct/offload 失效、targetSdk 35 焦点限制、`mediaProcessing`、`Virtualizer` 弃用 | 8 |

**属归纳模型或经验值，不能当源码结论用**

- 「7 块内存 6 次搬运」（6.2）是对 AIDL HAL + tinyalsa 常规路径的归纳。真实拷贝次数随厂商 HAL 实现变化——走虚拟 DMA、DSP 中转或 MMAP 模式，块数就不一样。
- 延迟档位（~5-10ms / 40-50ms / >100ms）是量级参考，不是官方保证值。
- `outputId` 对照表来自具体机型 dump，ID 由 `openOutput_l()` 分配，每台机器都不同。

*本文基于 Android 15 (API Level 35) 源码分析，不同厂商的定制 ROM 可能存在差异。*
