# 🎵 Android Audio Skills

> 让 AI 像音频架构师一样理解 Android 音频系统 —— 从 App 到 HAL，从 Java 到 DSP。

一套专为 AI 编程助手打造的 Android 音频子系统知识库，帮你快速定位音频初始化失败、卡顿、路由异常等疑难问题。每篇技能都经过 AOSP 源码验证，提供完整的调用链、架构图和调试命令。

---

## 📦 技能列表

### 🏗️ [android-audioflinger-architecture](./android-audioflinger-architecture/SKILL.md)

**AudioFlinger 全景架构** —— 音频系统的"心脏"

- 四层音频栈：应用 → Framework → Native → HAL
- 四种 PlaybackThread：MixerThread / DirectOutputThread / OffloadThread / DuplicatingThread
- 双混音策略：普通 Mixer vs FastMixer（SCHED_FIFO + Lock-free）
- 完整数据流：播放路径 & 录音路径 & Ring Buffer 机制
- 延迟分级：~5ms 极低延迟 → >100ms 省电模式
- 调试速查：dumpsys / logcat / Perfetto 一键命令

### 🔗 [android-audio-call-chain](./android-audio-call-chain/SKILL.md)

**AudioRecord 调用链追踪** —— 从一行 `new AudioRecord()` 到 HAL 的每一步

- 10+ 层完整调用时序：Java → JNI → Native → AudioFlinger → AudioPolicy → HAL
- 错误码 -22（BAD_VALUE）产生与传递路径
- Engine 设备选择逻辑：蓝牙 → 有线 → USB → 内置 Mic 的优先级
- QCOM HAL 加载机制：`dlopen` → `adev_open` → `open_input_stream`
- 电源状态影响：WAIT_FOR_VHAL 如何导致录音初始化失败

### 🚗 [android-car-audio-focus](./android-car-audio-focus/SKILL.md)

**Android 15 车载音频焦点管理** —— 多音源协调的"交通指挥系统"

- 四种焦点类型 + 四种通知类型，车载场景最佳响应策略
- MediaFocusControl 仲裁机制：FocusStack LIFO 栈管理
- 延迟焦点机制：通话中媒体请求挂起，通话结束自动恢复
- Android 15 新特性：系统级自动 Duck（-12dBFS）、targetSdk 35+ 后台限制
- 车载特有：多音区管理、CarAudioService 协同、安全告警最高优先级
- 调试速查：dumpsys / logcat 一键诊断焦点冲突

---

## ✨ 为什么值得关注

| 特点 | 说明 |
|------|------|
| 📖 **源码级精度** | 每处引用都标注了文件路径和行号，可直接跳转验证 |
| 🔍 **问题驱动** | 从真实调试场景出发，而非教科书式罗列 |
| 🧩 **AI 原生** | 为 AI 编程助手量身定制的结构化知识，触发即用 |
| 🎯 **实战导向** | 每个技能都包含调试命令、常见问题诊断和排查思路 |

---

## 🚧 持续更新

本仓库会持续完善，计划覆盖的方向包括：

- [ ] AudioTrack 播放流程详解
- [ ] AudioPolicyService 路由策略分析
- [ ] AAudio MMAP 低延迟路径
- [ ] 蓝牙音频（A2DP / LE Audio）调用链
- [x] 音频焦点（AudioFocus）管理机制
- [ ] 车载音频场景（CarAudioService）专题

> 💡 欢迎提 Issue 建议你关心的音频话题，或直接 PR 贡献！

---

## 📂 使用方式

将这些技能目录放入项目的 `.trae/skills/` 下：

```
your-project/
└── .trae/
    └── skills/
        ├── android-audio-call-chain/
        │   └── SKILL.md
        ├── android-audioflinger-architecture/
        │   └── SKILL.md
        └── android-car-audio-focus/
            └── SKILL.md
```

AI 助手会在分析音频相关问题时自动调用对应技能。

---

## 📚 源码基础

基于 **Android 15（API Level 35）** AOSP 源码分析，关键路径：

```
frameworks/av/services/audioflinger/     # AudioFlinger 服务
frameworks/av/media/libaudioclient/      # 客户端 Native 库
frameworks/av/services/audiopolicy/      # 音频策略服务
hardware/interfaces/audio/               # HAL 接口定义
hardware/qcom/audio/hal/                 # QCOM HAL 实现
```

---

<p align="center">
  <b>Made with ❤️ for Android audio developers</b>
</p>
