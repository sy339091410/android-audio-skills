---
name: "android-car-audio-focus"
description: "Documents Android 15 car audio focus management: focus types, arbitration mechanism (MediaFocusControl), focus stack, delayed focus, Android 15 auto-duck enhancement, car multi-zone audio, and debugging. Invoke when analyzing car audio focus conflicts, routing issues, or multi-zone audio management."
---

# Android 15 车载音频焦点管理

本技能总结了 Android 15 中音频焦点（Audio Focus）管理机制，重点覆盖车载场景下的多音区管理、焦点仲裁策略、延迟焦点、Android 15 新特性及调试方法。

## 一、概述

音频焦点是 Android 音频系统的**协作式仲裁协议**。它不是硬性锁，而是通过回调通知应用"你应该暂停"或"你可以降低音量"。应用是否响应全凭开发者自觉。在车载场景中，音频焦点管理尤为关键：导航、电话、音乐、语音助手、系统告警等多种音源需要在同一设备上协调发声。

**核心区分**：

| 维度 | 音频焦点 | 音量控制 |
|-----|---------|---------|
| 控制对象 | 是否应该播放 | 播放多响 |
| 生效范围 | 逻辑层（应用自愿遵守） | 硬件/软件实际音量值 |
| API | `requestAudioFocus()` | `setStreamVolume()` |
| 实施方 | 应用自行响应回调 | AudioFlinger/HAL 强制生效 |

## 二、焦点类型体系

### 2.1 请求类型（申请时指定）

| 请求类型 | 常量 | 车载场景 |
|---------|------|---------|
| 长期持有 | `AUDIOFOCUS_GAIN` | 音乐播放、收音机 |
| 短暂使用 | `AUDIOFOCUS_GAIN_TRANSIENT` | 导航播报、倒车雷达警告 |
| 短暂可 Duck | `AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK` | 系统提示音、方向盘按键反馈 |
| 短暂独占 | `AUDIOFOCUS_GAIN_TRANSIENT_EXCLUSIVE` | 语音助手唤醒、录音 |

### 2.2 通知类型（收到回调时携带）

| 通知类型 | 含义 | 车载建议响应 |
|---------|------|------------|
| `AUDIOFOCUS_LOSS` | 永久失去焦点 | 暂停音乐，不自动恢复 |
| `AUDIOFOCUS_LOSS_TRANSIENT` | 暂时失去焦点 | 暂停播放，保持资源，GAIN 时恢复 |
| `AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK` | 可降音量 | 音乐降音至 30%，语音类内容暂停 |
| `AUDIOFOCUS_GAIN` | 重新获得焦点 | 恢复播放，恢复音量 |

### 2.3 车载场景最佳响应策略

| 通知类型 | 音乐/收音机 | 导航播报 | 语音助手 | 系统告警 |
|---------|-----------|---------|---------|---------|
| `LOSS` | 暂停，不自动恢复 | 放弃本次播报 | 退出唤醒 | 退出 |
| `LOSS_TRANSIENT` | 暂停，GAIN 时恢复 | 等待重新播报 | 暂停拾音 | 延迟播报 |
| `LOSS_TRANSIENT_CAN_DUCK` | 降音至 30% | 降低播报音量 | — | — |

## 三、焦点请求 API

### 3.1 推荐方式：AudioFocusRequest（Android 8+）

```kotlin
val focusRequest = AudioFocusRequest.Builder(AudioManager.AUDIOFOCUS_GAIN)
    .setAudioAttributes(
        AudioAttributes.Builder()
            .setUsage(AudioAttributes.USAGE_MEDIA)
            .setContentType(AudioAttributes.CONTENT_TYPE_MUSIC)
            .build()
    )
    .setAcceptsDelayedFocusGain(true)  // 支持延迟获取焦点
    .setWillPauseWhenDucked(false)      // Duck 时不暂停，手动降音量
    .setOnAudioFocusChangeListener(
        focusChangeListener,
        Handler(Looper.getMainLooper())
    )
    .build()

val result = audioManager.requestAudioFocus(focusRequest)
when (result) {
    AudioManager.AUDIOFOCUS_REQUEST_GRANTED -> { /* 立即获得焦点 */ }
    AudioManager.AUDIOFOCUS_REQUEST_DELAYED -> { /* 延迟获取，等待回调 */ }
    AudioManager.AUDIOFOCUS_REQUEST_FAILED -> { /* 失败，不要播放 */ }
}
```

**Builder 关键参数**：

| 参数 | 含义 | 默认值 | 车载建议 |
|-----|------|-------|---------|
| `setAcceptsDelayedFocusGain(true)` | 接受延迟获取焦点 | `false` | 建议设为 `true`，通话中音乐请求可延迟到通话结束 |
| `setWillPauseWhenDucked(true)` | Duck 时直接暂停 | `false` | 音乐类 `false`，语音类 `true` |
| `setFocusGain()` | 焦点类型 | 必填 | — |
| `setAudioAttributes()` | 音频用途描述 | 必填建议 | 车载场景务必正确设置 Usage，影响优先级 |

### 3.2 焦点变化监听器

```kotlin
private val focusChangeListener = AudioManager.OnAudioFocusChangeListener { focusChange ->
    when (focusChange) {
        AudioManager.AUDIOFOCUS_GAIN -> {
            // 重新获得焦点（通话结束、导航播完）
            if (wasPlayingBeforeLoss) {
                mediaPlayer.start()
                mediaPlayer.setVolume(1.0f, 1.0f)
            }
        }
        AudioManager.AUDIOFOCUS_LOSS -> {
            // 永久失去焦点
            mediaPlayer.pause()
            wasPlayingBeforeLoss = false
        }
        AudioManager.AUDIOFOCUS_LOSS_TRANSIENT -> {
            // 短暂失去焦点（来电、语音助手）
            mediaPlayer.pause()
            wasPlayingBeforeLoss = true
        }
        AudioManager.AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK -> {
            // 可降音量继续播放（导航、通知）
            mediaPlayer.setVolume(0.3f, 0.3f)
        }
    }
}
```

### 3.3 焦点释放 —— 车载场景常见坑

```kotlin
// 常见坑：onStop() 中只释放 MediaPlayer，忘记释放焦点
override fun onStop() {
    super.onStop()
    if (::focusRequest.isInitialized) {
        audioManager.abandonAudioFocusRequest(focusRequest)
    }
    mediaPlayer.stop()
}

override fun onDestroy() {
    super.onDestroy()
    audioManager.abandonAudioFocusRequest(focusRequest)
    mediaPlayer.release()
}
```

## 四、焦点仲裁机制：MediaFocusControl

### 4.1 架构概览

音频焦点的仲裁逻辑运行在 `system_server` 进程的 `AudioService` 中，核心类是 `MediaFocusControl`：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        App 进程                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│  AudioManager.requestAudioFocus()                                            │
│         │                                                                    │
│         ▼ Binder IPC                                                        │
└─────────────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     system_server 进程                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│  AudioService                                                                │
│         │                                                                    │
│         ▼                                                                    │
│  MediaFocusControl                                                           │
│         │                                                                    │
│         ├─► 1. 构造 FocusRequester 对象                                      │
│         ├─► 2. 检查是否已在栈中（重复请求去重）                                │
│         ├─► 3. canReceiveFocusLocked() 评估优先级                             │
│         ├─► 4. notifyTopOfStackToGrantFocus() 通知当前持有者                  │
│         └─► 5. mFocusStack.push() 新请求者入栈                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 MediaFocusControl 核心逻辑

源码位置：`frameworks/base/services/core/java/com/android/server/audio/MediaFocusControl.java`

```java
// requestAudioFocus 核心流程
int requestAudioFocus(AudioAttributes aa, int focusChangeHint, IBinder cb,
        IAudioFocusDispatcher fd, String clientId, String callingPackageName,
        String attributionTag, int flags, IAudioPolicyCallback apc, int sdk) {
    synchronized (mAudioFocusLock) {
        // 1. 构造 FocusRequester 对象
        FocusRequester nfr = new FocusRequester(aa, focusChangeHint, cb, fd,
            clientId, callingPackageName, attributionTag, this, ...);

        // 2. 如果请求者已在栈中（重复请求），先移除旧的
        if (mFocusOwners.contains(clientId)) {
            removeFocusStackEntry(clientId, false, false);
        }

        // 3. 评估是否可以立即获得焦点
        if (canReceiveFocusLocked(nfr)) {
            // 4. 通知当前所有焦点持有者
            notifyTopOfStackToGrantFocus(nfr);
            // 5. 新请求者入栈
            mFocusStack.push(nfr);
            return AudioManager.AUDIOFOCUS_REQUEST_GRANTED;
        } else if ((flags & AudioManager.AUDIOFOCUS_FLAG_DELAY_OK) != 0) {
            // 支持延迟：加入等待队列
            mFocusStack.push(nfr);
            return AudioManager.AUDIOFOCUS_REQUEST_DELAYED;
        } else {
            return AudioManager.AUDIOFOCUS_REQUEST_FAILED;
        }
    }
}
```

### 4.3 焦点损失通知映射

`getFocusLossForGainRequest()` 根据请求类型决定通知类型：

```java
// FocusRequester.java
int getFocusLossForGainRequest(int gainRequest) {
    switch (gainRequest) {
        case AudioManager.AUDIOFOCUS_GAIN:
            return AudioManager.AUDIOFOCUS_LOSS;                    // 长期 → 永久丢失
        case AudioManager.AUDIOFOCUS_GAIN_TRANSIENT:
            return AudioManager.AUDIOFOCUS_LOSS_TRANSIENT;          // 短暂 → 短暂丢失
        case AudioManager.AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK:
            return AudioManager.AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK; // 可 Duck → 可降音量
        case AudioManager.AUDIOFOCUS_GAIN_TRANSIENT_EXCLUSIVE:
            return AudioManager.AUDIOFOCUS_LOSS_TRANSIENT;          // 独占 → 短暂丢失
        default:
            return AudioManager.AUDIOFOCUS_NONE;
    }
}
```

## 五、焦点栈管理

### 5.1 LIFO 栈机制

焦点栈是 `LinkedList<FocusRequester>`，遵循后进先出原则：

```
Push（入栈）：新请求者获得焦点 → 成为栈顶（当前焦点持有者）
Pop（出栈） ：请求者释放焦点 → 触发下一个持有者恢复

例如车载场景：
    初始状态：  [音乐App]               ← 音乐正在播放
    导航播报：  [音乐App, 导航App]       ← 导航入栈，音乐收到 LOSS_TRANSIENT_CAN_DUCK
    导航结束：  [音乐App]               ← 导航出栈，音乐收到 GAIN 恢复
    来电：      [音乐App, 电话App]       ← 电话入栈，音乐收到 LOSS_TRANSIENT
    通话结束：  [音乐App]               ← 电话出栈，音乐收到 GAIN 恢复
```

### 5.2 优先级判断：canReceiveFocusLocked

```java
// MediaFocusControl.java
private boolean canReceiveFocusLocked(FocusRequester nfr) {
    // 规则1：通话进行中，非通话相关音频不得抢占焦点
    if (isInVoiceCallLocked() &&
        !nfr.hasSameAttributesAs(AudioAttributes.USAGE_VOICE_COMMUNICATION)) {
        return false;
    }

    // 规则2：检查是否有优先级更高的持有者阻止获取
    for (FocusRequester focusOwner : mFocusStack) {
        if (focusOwner.isBlockedBy(nfr)) {
            return false;
        }
    }

    return true;
}
```

**车载场景优先级参考**：

```
电话 (USAGE_VOICE_COMMUNICATION)     ← 最高优先级
    │
语音助手 (USAGE_ASSISTANT)           ← Android 15 新增专项处理
    │
导航播报 (USAGE_ASSISTANCE_NAVIGATION_GUIDANCE)
    │
系统告警 (USAGE_ALARM)
    │
媒体播放 (USAGE_MEDIA)               ← 最低优先级
```

### 5.3 同一应用多次请求的去重

当同一应用多次请求焦点时（如切换电台），`MediaFocusControl` 先移除旧请求再入栈——不会出现同一应用在栈中多次出现。

## 六、延迟焦点机制

### 6.1 延迟触发条件

当设备处于通话状态（`AUDIO_MODE_IN_CALL`）时，媒体类音频焦点请求若设置了 `setAcceptsDelayedFocusGain(true)`，会以 `AUDIOFOCUS_REQUEST_DELAYED` 状态挂起，通话结束后自动回调。

**车载典型场景**：用户在通话中点击音乐播放 → 音乐请求延迟 → 通话结束后自动开始播放。

### 6.2 延迟焦点的正确处理

```kotlin
private var isFocusDelayed = false
private var wasPlayingBeforeLoss = false

private val focusChangeListener = AudioManager.OnAudioFocusChangeListener { focusChange ->
    when (focusChange) {
        AudioManager.AUDIOFOCUS_GAIN -> {
            if (isFocusDelayed || wasPlayingBeforeLoss) {
                isFocusDelayed = false
                mediaPlayer.start()
                mediaPlayer.setVolume(1.0f, 1.0f)
            }
        }
        // ... 其他情况
    }
}

fun playMusic() {
    val result = audioManager.requestAudioFocus(focusRequest)
    when (result) {
        AudioManager.AUDIOFOCUS_REQUEST_GRANTED -> mediaPlayer.start()
        AudioManager.AUDIOFOCUS_REQUEST_DELAYED -> {
            isFocusDelayed = true
            showDelayedPlaybackIndicator() // 显示"通话结束后自动播放"提示
        }
        AudioManager.AUDIOFOCUS_REQUEST_FAILED -> showPlaybackFailedMessage()
    }
}
```

## 七、Android 15 车载关键变化

### 7.1 系统级自动 Duck（Auto Ducking）

Android 15 引入了**系统级自动 Duck**，不再依赖应用自行响应回调。系统直接通过 `VolumeShaper` 在 AudioFlinger 层面降低音量：

```java
// frameworks/base/services/core/java/com/android/server/audio/PlaybackActivityMonitor.java
void duckPlayers(FocusRequester winner, FocusRequester loser, boolean forceDuck) {
    for (AudioPlaybackConfiguration apc : mPlayers.values()) {
        if (apc.getClientUid() == loser.getClientUid()) {
            VolumeShaper.Configuration duckConfig = new VolumeShaper.Configuration.Builder()
                .setInterpolatorType(VolumeShaper.Configuration.INTERPOLATOR_TYPE_CUBIC)
                .setOptionFlags(VolumeShaper.Configuration.OPTION_FLAG_DELAY_EFFECT |
                    VolumeShaper.Configuration.OPTION_FLAG_VOLUME_IN_DBFS)
                .setDuration(DEFAULT_DUCK_DURATION_MS)
                .setCurve(new float[] {0.f, 1.f}, new float[]{0.f, -12.f}) // 降低 12dBFS
                .build();

            apc.getPlayerProxy().applyVolumeShaper(duckConfig,
                new VolumeShaper.Operation.Builder().setXOffset(0.f).build());
        }
    }
}
```

**系统级 Duck 对车载的好处**：
- 第三方应用即使忘记实现 `CAN_DUCK` 响应，系统也会自动降音量
- 使用 `VolumeShaper` 实现平滑淡入淡出（而非突变）
- Duck 幅度固定为 -12dBFS（约 30% 音量），体验统一

### 7.2 焦点优先级体系更新

Android 15 细化了基于 `AudioAttributes` 的优先级判断，新增对 `USAGE_ASSISTANT`（语音助手）和 `USAGE_NOTIFICATION_EVENT`（事件通知）的专项处理：

```java
// 语音助手可以抢占除了通话以外的几乎所有焦点
if (attributes.getUsage() == AudioAttributes.USAGE_ASSISTANT) {
    // 允许 duck 媒体播放，但不打断电话
    gainRequest = AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK;
}
```

### 7.3 焦点请求限制（targetSdk 35+）

Android 15 对 `targetSdkVersion` ≥ 35 的应用引入关键限制：

> **应用必须是顶层应用或正在运行前台服务，才能请求音频焦点。**

否则 `requestAudioFocus()` 返回 `AUDIOFOCUS_REQUEST_FAILED`。

**车载影响**：

| 场景 | 是否需要适配 |
|-----|------------|
| 应用在前台播放 | 无影响 |
| 应用有前台服务（FGS）播放 | 无影响 |
| 后台启动（如 `BOOT_COMPLETED`）时请求焦点 | **需要适配** |
| 后台预加载音频并获取焦点 | **会被拒绝** |

### 7.4 MediaSession 与焦点联动增强

Android 15 加强了 `MediaSession` 与 `AudioFocus` 的联动。使用 Media3/ExoPlayer 时，焦点管理可完全自动化：

```kotlin
val player = ExoPlayer.Builder(context)
    .setAudioAttributes(
        AudioAttributes.Builder()
            .setUsage(C.USAGE_MEDIA)
            .setContentType(C.AUDIO_CONTENT_TYPE_MUSIC)
            .build(),
        /* handleAudioFocus = */ true  // 自动处理焦点
    )
    .build()
```

开启 `handleAudioFocus = true` 后，ExoPlayer 自动完成：请求焦点 → 响应 LOSS → Duck 降音量 → GAIN 恢复。

## 八、车载特有场景分析

### 8.1 多音区管理

车载系统通常有多个音区（主驾、副驾、后排），不同音区可能需要独立的焦点管理：

```
┌──────────────────────────────────────────────────────────────────┐
│                      车载音频焦点架构                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │ 主驾音区     │  │ 副驾音区     │  │ 后排音区     │              │
│  │ (Zone 1)    │  │ (Zone 2)    │  │ (Zone 3)    │              │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤              │
│  │ 导航播报     │  │ 副驾娱乐     │  │ 后排娱乐     │              │
│  │ 电话         │  │ 音量控制     │  │ 音量控制     │              │
│  │ 系统告警     │  │             │  │             │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              AudioPolicyService（全局焦点仲裁）              │   │
│  │  根据 AudioAttributes + Zone ID 决定焦点归属               │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              AudioFlinger（执行层）                        │   │
│  │  各音区对应独立 PlaybackThread，物理隔离音频输出            │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

### 8.2 车载音频焦点优先级

车载场景中，音频焦点的优先级排序通常为：

```
1. 安全告警（倒车雷达、碰撞预警）     ← 最高，不可被抢占
2. 电话（蓝牙通话、VoLTE）            ← 抢占所有媒体
3. 语音助手（唤醒、交互）              ← Android 15 专项处理
4. 导航播报                           ← 抢占媒体，可 Duck
5. 系统提示音（按键、状态提示）         ← 短暂 Duck
6. 媒体播放（音乐、收音机、视频）       ← 最低，可被任何抢占
```

### 8.3 CarAudioService 与焦点管理

车载系统的音频焦点管理涉及 `CarAudioService`（`packages/services/Car/service/`），它作为车载音频的总管家，与标准 `AudioService` 协同工作：

```
CarAudioService
    │
    ├─► 管理多音区（Zone）配置
    ├─► 协调音区间的焦点策略
    ├─► 处理车载特有 AudioAttributes（如 USAGE_ASSISTANCE_NAVIGATION_GUIDANCE）
    └─► 与 AudioService/MediaFocusControl 协同仲裁
```

### 8.4 电源状态对焦点的影响

车载电源状态变化（如 WAIT_FOR_VHAL、S2R 恢复）会影响音频焦点系统：

```java
// packages/services/Car/service/src/com/android/car/power/CarPowerManagementService.java
// 第333行 - 初始状态
onApPowerStateChange(CpmsState.WAIT_FOR_VHAL, CarPowerStateListener.WAIT_FOR_VHAL);

// 第482行 - S2R 恢复时音频焦点可能需重建
// TODO(b/177478420): Restore audio if artificially modified for S2R.
```

## 九、调试速查

### 9.1 焦点状态快照

```bash
# 查看音频焦点完整状态
adb shell dumpsys audio | grep -A 30 "Audio Focus"

# 关键输出解读：
#  - Focus Stack: 当前焦点栈（从栈顶到栈底）
#  - Focus Owners: 当前持有焦点的客户端列表
#  - Notified focus losers: 被通知失去焦点的客户端
```

### 9.2 实时日志过滤

```bash
# 过滤焦点相关日志
adb logcat -s MediaFocusControl:V AudioService:V AudioManager:V

# 查看焦点请求和释放
adb logcat | grep -E "requestAudioFocus|abandonAudioFocus|focusChange|FocusRequester"

# 查看 CarAudioService 相关日志
adb logcat | grep -E "CarAudioService|CarAudioFocus"
```

### 9.3 常见问题诊断

**问题1：音乐在后台突然停止播放**

```bash
# 检查焦点是否被抢占
adb shell dumpsys audio | grep -A 20 "Audio Focus"
adb logcat | grep -E "focusChange|LOSS|AUDIOFOCUS"

# 可能原因：
# 1. 导航/语音助手抢占了焦点
# 2. Android 15 目标 SDK 35+ 后台请求焦点被拒绝
# 3. 应用未正确响应 LOSS 回调
```

**问题2：导航播报时音乐音量没有降低**

```bash
# 检查 Duck 是否生效
adb logcat | grep -i "duck\|VolumeShaper\|PlaybackActivityMonitor"

# 可能原因：
# 1. 导航请求了 GAIN_TRANSIENT 而非 GAIN_TRANSIENT_MAY_DUCK
# 2. 音乐 App 设置了 setWillPauseWhenDucked(true)
# 3. 音乐 App 未实现 CAN_DUCK 回调（Android 15 系统级 Duck 可兜底）
```

**问题3：通话结束后音乐没有自动恢复**

```bash
# 检查焦点栈是否正常出栈
adb shell dumpsys audio | grep -A 30 "Audio Focus"
adb logcat | grep -E "removeFocusStackEntry|notifyStackTopToGrantFocus"

# 可能原因：
# 1. 电话 App 未正确释放焦点
# 2. 音乐 App 未设置 setAcceptsDelayedFocusGain(true)
# 3. 音乐 App 在 LOSS 时将 wasPlayingBeforeLoss 设为 false
```

**问题4：车载多音区焦点冲突**

```bash
# 检查各音区状态
adb shell dumpsys media.audio_flinger | grep -E "Output thread|zone"
adb shell dumpsys audio | grep -A 50 "Audio Focus"

# 检查 CarAudioService 配置
adb shell dumpsys car_service | grep -A 30 "audio"
```

### 9.4 焦点相关 dumpsys 命令汇总

```bash
# 音频焦点详情
adb shell dumpsys audio | grep -A 30 "Audio Focus"

# AudioPolicy 路由决策
adb shell dumpsys media.audio_policy

# AudioFlinger 线程和 Track 状态
adb shell dumpsys media.audio_flinger

# 车载服务状态
adb shell dumpsys car_service | grep -A 50 "audio"

# 当前播放器列表
adb shell dumpsys media_session | grep -A 10 "player"
```

## 十、关键源码路径

| 组件 | 路径 |
|-----|------|
| MediaFocusControl（焦点仲裁核心） | `frameworks/base/services/core/java/com/android/server/audio/MediaFocusControl.java` |
| AudioService（音量和焦点服务） | `frameworks/base/services/core/java/com/android/server/audio/AudioService.java` |
| AudioManager（客户端 API） | `frameworks/base/media/java/android/media/AudioManager.java` |
| AudioFocusRequest（焦点请求封装） | `frameworks/base/media/java/android/media/AudioFocusRequest.java` |
| PlaybackActivityMonitor（Android 15 自动 Duck） | `frameworks/base/services/core/java/com/android/server/audio/PlaybackActivityMonitor.java` |
| FocusRequester（焦点请求者封装） | `frameworks/base/services/core/java/com/android/server/audio/FocusRequester.java` |
| CarAudioService（车载音频服务） | `packages/services/Car/service/src/com/android/car/audio/CarAudioService.java` |
| CarPowerManagementService（电源状态） | `packages/services/Car/service/src/com/android/car/power/CarPowerManagementService.java` |
| AudioPolicyService（策略决策） | `frameworks/av/services/audiopolicy/` |
| AudioFlinger（音频执行） | `frameworks/av/services/audioflinger/` |

## 十一、与现有技能的关系

| 维度 | 本技能 | android-audioflinger-architecture | android-audio-call-chain |
|-----|--------|-----------------------------------|--------------------------|
| 视角 | 焦点管理策略 | 宏观架构 | 微观调用链 |
| 重点 | 焦点仲裁、栈管理、延迟焦点、车载场景 | 四层架构、线程类型、混音机制 | AudioRecord 逐层调用 |
| 适用场景 | 多音源冲突、焦点抢占、Duck 不生效 | 理解整体设计、定位架构问题 | 追踪错误码、分析初始化失败 |

## 十二、总结

本技能涵盖了 Android 15 车载音频焦点管理机制，核心要点：

1. **焦点类型**：四种请求类型（GAIN/TRANSIENT/MAY_DUCK/EXCLUSIVE）和四种通知类型（LOSS/TRANSIENT/CAN_DUCK/GAIN），车载场景各有明确的响应策略
2. **仲裁机制**：`MediaFocusControl` 在 `system_server` 中运行，通过 `FocusStack`（LIFO）管理焦点持有者
3. **焦点栈**：Push/Pop 机制实现焦点自动归还，优先级判断确保通话不被媒体抢占
4. **延迟焦点**：通话中媒体请求可挂起，通话结束后自动回调
5. **Android 15 新特性**：系统级自动 Duck（-12dBFS）、`USAGE_ASSISTANT` 专项优先级、targetSdk 35+ 后台焦点限制、MediaSession 联动增强
6. **车载特有**：多音区管理、`CarAudioService` 协同、安全告警最高优先级、电源状态影响

理解音频焦点机制，是处理车载多音源协调、避免"声音打架"、提升用户体验的基础。

*本文基于 Android 15 (API Level 35) 源码分析，不同厂商的定制 ROM 可能存在差异。车载音频焦点管理的具体实现可能因 OEM 定制而异。*