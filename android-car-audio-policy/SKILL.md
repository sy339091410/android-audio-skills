---
name: "android-car-audio-policy"
description: "Documents Android 15 AudioPolicyService strategy management: architecture, policy XML config, strategy mapping, routing decision flow, device failover, ForceUse, audio modes, AIDL HAL migration, CAP engine, and car multi-zone policy. Invoke when analyzing audio routing issues, device selection, or car audio policy configuration."
---

# Android 15 车载 AudioPolicyService 策略管理

本技能总结了 Android 15 中 AudioPolicyService（APS）的策略管理机制，重点覆盖车载场景下的策略配置、路由决策流程、设备管理、ForceUse 强制路由、AIDL HAL 迁移及可配置音频策略引擎（CAP Engine）。

## 一、概述

AudioPolicyService 是 Android 音频系统的**策略决策中心**。它与 AudioFlinger 形成经典的"策略与执行分离"架构：

| 服务 | 角色 | 关注点 |
|-----|------|-------|
| **AudioPolicyService** | 决策者（销售总监） | 往哪里播放、用什么设备、音量多大 |
| **AudioFlinger** | 执行者（音频工厂） | 怎么播放、混音、重采样、写 HAL |

APS 不接触任何音频数据，只负责回答三个核心问题：
1. **该用哪个设备**？（扬声器、耳机、蓝牙、USB……）
2. **该走哪个流类型**？（音乐、通话、铃声、通知……）
3. **各流之间如何协调**？（音量组、焦点、Duck……）

**关键区分**：`AudioService`（Java 层，`system_server`）提供 `AudioManager` API 作为应用入口，最终会跨 Binder 调用到 APS（Native 层，`audioserver`）执行真正的策略决策。

## 二、AudioPolicyService 架构概览

### 2.1 启动与进程位置

APS 运行在 `audioserver` 进程中，与 AudioFlinger 是"邻居"，由 `main_audioserver.cpp` 启动：

```cpp
// frameworks/av/media/audioserver/main_audioserver.cpp
int main(int argc __unused, char **argv) {
    // ...
    AudioFlinger::instantiate();          // 音频混音引擎
    AudioPolicyService::instantiate();    // 音频策略服务
    // ...
}
```

### 2.2 核心组件层次

```
AudioPolicyService（Binder 服务端）
    │
    ├── AudioPolicyManager（APM）       ← 策略管理主体，持有全局状态
    │   ├── Engine（EngineInterface）    ← 路由决策引擎，可插拔
    │   │   └── EngineBase               ← 默认引擎实现，读取 XML 配置
    │   ├── AudioPolicyMix               ← 动态路由（Mix 策略管理）
    │   └── HwModule                     ← 硬件模块抽象
    │
    └── AudioCommandThread              ← 异步命令线程（设备连接/断开）
```

| 组件 | 职责 |
|------|------|
| **AudioPolicyManager** | 策略逻辑核心，持有已连接设备列表、已打开的输出/输入流、活跃策略等全局状态 |
| **Engine** | 路由决策大脑，从 APM 中抽离，通过 `EngineInterface` 隔离。默认实现读取 `audio_policy_engine_configuration.xml`，根据 Mode、ForceUse 等计算最终设备 |
| **AudioCommandThread** | 异步处理设备连接/断开等耗时操作，避免阻塞调用方 |
| **AudioPolicyMix** | 动态路由：允许运行时注册匹配规则，将特定音频流重定向到指定设备 |

### 2.3 与 AudioFlinger 的协作

```
App                      AudioPolicyService          AudioFlinger
 │                              │                         │
 │── getOutput() ──────────────>│                         │
 │                              │ 决策路由...              │
 │                              │── openOutput() ────────>│
 │                              │<─ output handle ────────│
 │<─ IoHandle ─────────────────│                         │
 │                              │                         │
 │── createTrack(IoHandle) ──────────────────────────────>│
```

APS 只负责决定用哪个输出流（`IoHandle`），AudioFlinger 负责实际创建 Track 并处理音频数据。**策略和执行完全解耦**。

## 三、策略配置文件：audio_policy_configuration.xml

### 3.1 文件位置与加载时机

```
# 主配置文件（设备商提供，优先级最高）
/vendor/etc/audio_policy_configuration.xml

# AOSP 默认配置（通用参考）
/system/etc/audio_policy_configuration.xml

# 引擎配置（策略规则定义）
/system/etc/audio_policy_engine_configuration.xml
```

AudioPolicyManager 初始化时解析这些文件，构建设备拓扑图。

### 3.2 配置文件结构：三个核心概念

```xml
<!-- audio_policy_configuration.xml 核心结构 -->
<audioPolicyConfiguration version="1.0">
    <globalConfiguration speaker_drc_enabled="true"/>

    <modules>
        <module name="primary" halVersion="2.0">

            <!-- mixPort: 软件端口，对应 AudioFlinger 中的一个 PlaybackThread -->
            <mixPorts>
                <mixPort name="primary output" role="source"
                         flags="AUDIO_OUTPUT_FLAG_PRIMARY">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                             samplingRates="48000"
                             channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                </mixPort>
                <mixPort name="deep_buffer" role="source"
                         flags="AUDIO_OUTPUT_FLAG_DEEP_BUFFER">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                             samplingRates="48000"
                             channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                </mixPort>
            </mixPorts>

            <!-- devicePort: 物理硬件设备 -->
            <devicePorts>
                <devicePort tagName="Speaker" type="AUDIO_DEVICE_OUT_SPEAKER" role="sink">
                    <gains>
                        <gain name="" mode="AUDIO_GAIN_MODE_JOINT"
                              minValueMB="-9600" maxValueMB="0"
                              defaultValueMB="0" stepValueMB="100"/>
                    </gains>
                </devicePort>
                <devicePort tagName="Earpiece" type="AUDIO_DEVICE_OUT_EARPIECE" role="sink"/>
                <devicePort tagName="Wired Headset" type="AUDIO_DEVICE_OUT_WIRED_HEADSET" role="sink"/>
                <devicePort tagName="BT A2DP" type="AUDIO_DEVICE_OUT_BLUETOOTH_A2DP" role="sink"/>
                <devicePort tagName="BT SCO" type="AUDIO_DEVICE_OUT_BLUETOOTH_SCO" role="sink"/>
            </devicePorts>

            <!-- routes: 定义 mixPort 到 devicePort 的连接关系 -->
            <routes>
                <route type="mix" sink="Speaker"
                       sources="primary output,deep_buffer"/>
                <route type="mix" sink="Wired Headset"
                       sources="primary output,deep_buffer"/>
                <route type="mix" sink="BT A2DP"
                       sources="primary output,deep_buffer"/>
            </routes>
        </module>
    </modules>
</audioPolicyConfiguration>
```

| 概念 | 含义 | 对应实体 |
|-----|------|---------|
| `mixPort` | 软件端口 | AudioFlinger 中的 `PlaybackThread` |
| `devicePort` | 物理硬件设备 | 扬声器、耳机插孔、蓝牙等 |
| `route` | 连接关系 | 定义哪些 mixPort 的音频可以路由到哪个 devicePort |

### 3.3 引擎配置：策略规则定义

`audio_policy_engine_configuration.xml` 定义的是**业务规则**（而非拓扑）：

```xml
<ProductStrategies>
    <!-- 媒体策略：音乐/视频/游戏 -->
    <ProductStrategy name="STRATEGY_MEDIA">
        <AttributesGroup streamType="AUDIO_STREAM_MUSIC" volumeGroup="music">
            <Attributes usage="AUDIO_USAGE_MEDIA"/>
            <Attributes usage="AUDIO_USAGE_GAME"/>
        </AttributesGroup>
        <DeviceConfiguration>
            <Device type="AUDIO_DEVICE_OUT_WIRED_HEADPHONE"/>
            <Device type="AUDIO_DEVICE_OUT_WIRED_HEADSET"/>
            <Device type="AUDIO_DEVICE_OUT_BLUETOOTH_A2DP"/>
            <Device type="AUDIO_DEVICE_OUT_USB_DEVICE"/>
            <Device type="AUDIO_DEVICE_OUT_SPEAKER"/>
        </DeviceConfiguration>
    </ProductStrategy>

    <!-- 通话策略：蓝牙免提 > 有线耳机 > 听筒 -->
    <ProductStrategy name="STRATEGY_PHONE">
        <AttributesGroup streamType="AUDIO_STREAM_VOICE_CALL">
            <Attributes usage="AUDIO_USAGE_VOICE_COMMUNICATION"/>
        </AttributesGroup>
        <DeviceConfiguration>
            <Device type="AUDIO_DEVICE_OUT_BLUETOOTH_SCO_HEADSET"/>
            <Device type="AUDIO_DEVICE_OUT_WIRED_HEADSET"/>
            <Device type="AUDIO_DEVICE_OUT_EARPIECE"/>
        </DeviceConfiguration>
    </ProductStrategy>
</ProductStrategies>
```

这就是"插上有线耳机音乐自动切到耳机"的原因——`STRATEGY_MEDIA` 的设备优先级列表中，有线耳机排在扬声器前面。

## 四、Stream 类型与策略映射

### 4.1 Legacy Stream 与 AudioAttributes 映射

| Stream Type | AudioAttributes Usage | 典型场景 |
|------------|----------------------|---------|
| `STREAM_MUSIC` | `USAGE_MEDIA` / `USAGE_GAME` | 音乐、游戏音效 |
| `STREAM_VOICE_CALL` | `USAGE_VOICE_COMMUNICATION` | 电话通话 |
| `STREAM_RING` | `USAGE_NOTIFICATION_RINGTONE` | 来电铃声 |
| `STREAM_ALARM` | `USAGE_ALARM` | 闹钟 |
| `STREAM_NOTIFICATION` | `USAGE_NOTIFICATION` | 消息通知 |
| `STREAM_SYSTEM` | `USAGE_ASSISTANCE_SONIFICATION` | 按键音、触控反馈 |
| `STREAM_DTMF` | `USAGE_VOICE_COMMUNICATION_SIGNALLING` | 拨号按键音 |

### 4.2 策略映射链路

```
AudioAttributes (usage=USAGE_MEDIA)
         │
         ▼ Engine::getProductStrategyForAttributes()
ProductStrategy (STRATEGY_MEDIA)
         │
         ▼ Engine::getDevicesForProductStrategy()
DevicePriorityList [HEADPHONE > HEADSET > BT_A2DP > USB > SPEAKER]
         │
         ▼ 过滤当前可用设备
FinalDevice (例如：有线耳机已插入 → DEVICE_OUT_WIRED_HEADSET)
```

关键源码：

```cpp
// frameworks/av/services/audiopolicy/enginedefault/src/EngineBase.cpp
DeviceVector EngineBase::getDevicesForProductStrategy(
        product_strategy_t strategy) const {
    const auto& strategyDevices =
        getProductStrategies().getDevicesForProductStrategy(strategy);

    // 从优先级列表中过滤出当前可用的设备
    DeviceVector availableDevices =
        getApmObserver()->getAvailableOutputDevices();

    for (const auto& device : strategyDevices) {
        if (availableDevices.contains(device)) {
            return DeviceVector(device); // 返回优先级最高的可用设备
        }
    }
    return DeviceVector();
}
```

## 五、音频路由决策全流程

### 5.1 getOutputForAttr()：路由决策入口

```cpp
// frameworks/av/services/audiopolicy/managerdefault/AudioPolicyManager.cpp
status_t AudioPolicyManager::getOutputForAttr(
        const audio_attributes_t *attr,
        audio_io_handle_t *output,
        audio_session_t session,
        audio_stream_type_t *stream,
        uid_t uid,
        const audio_config_t *config,
        audio_output_flags_t *flags,
        audio_port_handle_t *selectedDeviceId,
        audio_port_handle_t *portId,
        std::vector<audio_io_handle_t> *secondaryOutputs,
        output_type_t *outputType,
        bool *isSpatialized,
        bool *isBitPerfect) {
    // 1. AudioAttributes 转 ProductStrategy
    result_device_t resultDevice =
        getDeviceAndMixForAttributes(*attr, uid, ...);

    // 2. 通过 Engine 决策输出设备
    sp<DeviceDescriptor> outputDevice = resultDevice.device;

    // 3. 选择合适的输出流
    *output = getOutputForDevice(outputDevice, session, *stream,
                                 config, flags, ...);
    return NO_ERROR;
}
```

### 5.2 Engine 三因素决策（优先级从高到低）

```cpp
// frameworks/av/services/audiopolicy/enginedefault/src/Engine.cpp
DeviceVector Engine::getOutputDevicesForAttributes(
        const audio_attributes_t &attributes,
        const sp<DeviceDescriptor> preferredDevice,
        bool fromCache) const {
    // 因素1：优先使用应用指定设备（setPreferredDevice()）
    if (preferredDevice != nullptr) {
        return DeviceVector(preferredDevice);
    }

    // 因素2：获取产品策略
    auto strategy = getProductStrategyForAttributes(attributes);

    // 因素3：IN_CALL 模式强制走通话路由
    if (mPhoneState == AUDIO_MODE_IN_CALL) {
        strategy = getProductStrategyForStream(AUDIO_STREAM_VOICE_CALL);
    }

    return getDevicesForProductStrategy(strategy);
}
```

| 优先级 | 决策因素 | 说明 |
|:-----:|---------|------|
| 1 | 应用指定设备 | `setPreferredDevice()` 最高优先级 |
| 2 | 音频模式（Mode） | `IN_CALL` 强制走通话路由 |
| 3 | 策略优先级列表 | 正常情况下按设备优先级选择 |

### 5.3 openOutput：决策落地

```cpp
audio_io_handle_t AudioPolicyManager::getOutputForDevice(
        const sp<DeviceDescriptor>& device,
        audio_session_t session,
        audio_stream_type_t stream,
        const audio_config_t *config,
        audio_output_flags_t *flags) {
    // 在已有输出流中查找可复用的
    for (size_t i = 0; i < mOutputs.size(); i++) {
        sp<SwAudioOutputDescriptor> desc = mOutputs.valueAt(i);
        if (desc->supportedDevices().contains(device) &&
            desc->isCompatibleProfile(...)) {
            return desc->mIoHandle; // 复用已有输出流
        }
    }

    // 没有合适的，请求 AudioFlinger 打开新输出流
    audio_io_handle_t output = AUDIO_IO_HANDLE_NONE;
    status_t status = mpClientInterface->openOutput(
        profile->getModuleHandle(), &output, &outputConfig, device, ...);
    return output;
}
```

## 六、设备管理：连接、断开与 Failover

### 6.1 设备连接通知路径

```
内核 jack detection
    │
    ▼ AudioHAL (setParameters "headset_connect=true")
    │
    ▼ AudioFlinger
    │
    ▼ AudioPolicyClient::setAudioPortConfig()
    │
    ▼ AudioPolicyManager::setDeviceConnectionState()
```

APM 收到连接通知后做三件事：

```cpp
// frameworks/av/services/audiopolicy/managerdefault/AudioPolicyManager.cpp
status_t AudioPolicyManager::setDeviceConnectionState(
        const sp<DeviceDescriptor> &device,
        audio_policy_dev_state_t state, ...) {
    if (state == AUDIO_POLICY_DEVICE_STATE_AVAILABLE) {
        // 1. 更新可用设备列表
        mAvailableOutputDevices.add(device);

        // 2. 异步通知 AudioFlinger 更新路由
        checkOutputsForDevice(device, state, outputs);

        // 3. 触发路由重新计算
        updateCallAndOutputRouting();
    }
    // ...
}
```

### 6.2 设备断开 Failover 机制

设备断开时，APM 自动执行 Failover —— 找下一个优先级最高的可用设备：

```cpp
void AudioPolicyManager::checkOutputsForDevice(
        const sp<DeviceDescriptor>& device,
        audio_policy_dev_state_t state,
        SortedVector<audio_io_handle_t>& outputs) {
    if (state == AUDIO_POLICY_DEVICE_STATE_UNAVAILABLE) {
        for (auto output : outputs) {
            sp<SwAudioOutputDescriptor> desc = mOutputs.valueFor(output);
            if (desc->devices().contains(device)) {
                // 切换到下一个优先级设备
                DeviceVector newDevices = getNewOutputDevices(desc, false);
                if (newDevices != desc->devices()) {
                    setOutputDevices(desc, newDevices, true);
                }
            }
        }
    }
}
```

这就是"拔出耳机音乐自动切回扬声器"的原理——不是魔法，是 Failover。

### 6.3 蓝牙 A2DP 特殊处理

蓝牙 A2DP 连接/断开比有线设备复杂，涉及协议栈异步状态：

```cpp
status_t AudioPolicyManager::setDeviceConnectionStateInt(
        audio_devices_t deviceType,
        audio_policy_dev_state_t state, ...) {
    if (audio_is_a2dp_out_device(deviceType)) {
        if (state == AUDIO_POLICY_DEVICE_STATE_AVAILABLE) {
            // 等待 A2DP 编解码协商完成
            mA2dpSuspended = false;
        }
    }
    // ...
}
```

蓝牙设备有独立的 `A2dpSuspended` 状态，通话中会被挂起（切回听筒/有线），通话结束后自动恢复。

## 七、音频模式与 ForceUse

### 7.1 四种音频模式

| Mode | 值 | 触发条件 | 效果 |
|-----|:--:|---------|------|
| `AUDIO_MODE_NORMAL` | 0 | 默认状态 | 按各策略优先级路由 |
| `AUDIO_MODE_RINGTONE` | 1 | 来电铃声 | 铃声走 RINGTONE 策略 |
| `AUDIO_MODE_IN_CALL` | 2 | 接通电话 | 强制走 PHONE 策略，启用 EC/NS |
| `AUDIO_MODE_IN_COMMUNICATION` | 3 | VoIP 通话 | 类似 IN_CALL，但走软件路径 |

### 7.2 ForceUse 强制路由

ForceUse 可以**覆盖默认策略**，强制特定场景的设备路由。常见类型：

| 强制场景 | 用途 |
|---------|------|
| `AUDIO_POLICY_FORCE_FOR_COMMUNICATION` | 强制通话/VoIP 设备路由（如免提模式） |
| `AUDIO_POLICY_FORCE_FOR_MEDIA` | 强制媒体播放设备路由（如强制蓝牙输出） |
| `AUDIO_POLICY_FORCE_FOR_DOCK` | 强制扩展坞场景路由（车载模式） |
| `AUDIO_POLICY_FORCE_FOR_SYSTEM` | 强制系统音效路由（如静音模式下仍播放按键音） |
| `AUDIO_POLICY_FORCE_FOR_ENCODED_SURROUND` | 强制环绕声编码输出 |

**STRATEGY vs ForceUse 对比**：

| 维度 | STRATEGY | ForceUse |
|-----|---------|---------|
| 目的 | 逻辑分组，决定默认行为 | 强制覆盖默认行为 |
| 动态性 | 静态配置（XML） | 可运行时动态修改（`setForceUse`） |
| 优先级 | 低（可被强制覆盖） | 高（覆盖策略） |
| 影响范围 | 全局策略 | 特定场景 |

**典型用法**：

```java
// 强制通话走扬声器（免提）
AudioManager audioManager = (AudioManager) getSystemService(AUDIO_SERVICE);
audioManager.setMode(AudioManager.MODE_IN_CALL);
audioManager.setSpeakerphoneOn(true);
// 内部调用：setForceUse(FOR_COMMUNICATION, FORCE_SPEAKER)
```

```cpp
// Engine 层处理 ForceUse
status_t Engine::setForceUse(audio_policy_force_use_t usage,
                              audio_policy_forced_cfg_t config) {
    switch(usage) {
    case AUDIO_POLICY_FORCE_FOR_COMMUNICATION:
        if (config != AUDIO_POLICY_FORCE_SPEAKER &&
            config != AUDIO_POLICY_FORCE_BT_SCO &&
            config != AUDIO_POLICY_FORCE_NONE) {
            return BAD_VALUE;
        }
        break;
    // ...
    }
    return EngineBase::setForceUse(usage, config);
}
```

## 八、Android 15 车载关键变化

### 8.1 AIDL HAL 全面替代 HIDL

Android 15 完成了音频 HAL 从 HIDL 到 AIDL 的全面迁移：

| 影响维度 | 变化 |
|---------|------|
| IPC 效率 | AIDL 比 HIDL 更高效，降低策略决策到硬件执行的延迟 |
| 设备描述符 | 更精细的 `AudioPort`/`AudioProfile`，携带更多元数据 |
| 空间音频 | 支持编码格式（E_AC3_JOC）与设备物理声道布局匹配 |
| 策略决策 | 更丰富的数据基础，支持更智能的路由选择 |

### 8.2 音量组与路由深度绑定

Android 15 将音量组从简单的音量耦合单元升级为**路由策略的核心组织维度**：

| 版本 | 音量组角色 |
|-----|----------|
| Android 14 及以前 | 主要用于音量同步（将多个设备绑定到同一音量曲线） |
| Android 15 | 策略路由的标识单元，每个音量组可关联独立路由规则树 |

**车载场景应用**：为"主驾区"、"后排左区"、"后排右区"分别定义独立音量组，动态路由策略可配置为：导航音频强制路由到"主驾区"音量组对应的扬声器集合。

### 8.3 可配置音频策略引擎（CAP Engine）

Android 15 车载引入了可配置音频策略引擎（Configurable Audio Policy Engine），基于 Intel Parameter Framework：

```
传统 Engine（默认）              CAP Engine（车载新增）
    │                                │
    ├─ 读取 XML 配置                  ├─ 基于插件的规则引擎
    ├─ 静态策略规则                   ├─ 支持运行时条件匹配
    ├─ 固定优先级列表                  ├─ 多条件组合（Mode + Zone + Usage）
    └─ 不区分音区                     └─ 原生支持多音区路由
```

CAP Engine 通过 `car_audio_configuration.xml` 中的 `audioPolicyEngine` 配置启用，允许 OEM 定义更复杂的车载音频路由规则。

### 8.4 车载音频配置（car_audio_configuration.xml）

车载系统还有独立的音频配置文件：

```
# 车载音频配置（CarAudioService 读取）
/vendor/etc/car_audio_configuration.xml

# 引用关系
car_audio_configuration.xml 中引用的所有设备
必须在 audio_policy_configuration.xml 中定义
```

车载音频上下文（CarAudioContexts）将音频用途归入标准化类别，用于路由、音量组、焦点和冲突管理。

## 九、车载特有场景分析

### 9.1 多音区策略路由

车载系统通常有多个音区（主驾、副驾、后排），每个音区需要独立的路由策略：

```
┌──────────────────────────────────────────────────────────────────┐
│                    AudioPolicyService                             │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                  Engine（CAP Engine）                       │  │
│  │                                                             │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                 │  │
│  │  │ 主驾音区  │  │ 副驾音区  │  │ 后排音区  │                 │  │
│  │  │ Zone 0   │  │ Zone 1   │  │ Zone 2   │                 │  │
│  │  ├──────────┤  ├──────────┤  ├──────────┤                 │  │
│  │  │ 导航      │  │ 媒体      │  │ 媒体      │                 │  │
│  │  │ 电话      │  │ 音量控制  │  │ 音量控制  │                 │  │
│  │  │ 告警      │  │           │  │           │                 │  │
│  │  └──────────┘  └──────────┘  └──────────┘                 │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              │                                    │
│                              ▼                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                    AudioFlinger                             │  │
│  │  各音区对应独立 PlaybackThread，物理隔离音频输出             │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### 9.2 CarAudioService 与 APS 协同

```
CarAudioService（Java 层，CarService 进程）
    │
    ├─► 管理多音区（Zone）配置
    ├─► 读取 car_audio_configuration.xml
    ├─► 协调音区间的焦点策略
    ├─► 处理车载特有 AudioAttributes
    └─► 与 AudioPolicyService 协同路由

AudioPolicyService（Native 层，audioserver 进程）
    │
    ├─► 执行实际路由决策
    ├─► 管理设备连接/断开
    ├─► 与 AudioFlinger 交互
    └─► 处理 ForceUse 和 Mode 切换
```

### 9.3 车载策略优先级参考

```
安全告警（倒车雷达、碰撞预警）     ← 不可被抢占
    │
电话（蓝牙通话、VoLTE）            ← 强制走 PHONE 策略
    │
语音助手                           ← Android 15 专项处理
    │
导航播报                           ← 主驾音区优先
    │
系统提示音                         ← 短暂 Duck
    │
媒体播放（音乐、收音机、视频）       ← 最低，可被任何抢占
```

## 十、调试速查

### 10.1 策略和路由状态

```bash
# 查看 AudioPolicy 路由决策
adb shell dumpsys media.audio_policy

# 关键输出解读：
#  - Output devices: 当前输出设备
#  - Input devices: 当前输入设备
#  - Available output devices: 可用输出设备列表
#  - Available input devices: 可用输入设备列表
#  - HW Modules: 已加载的硬件模块
```

### 10.2 实时日志过滤

```bash
# 过滤策略相关日志
adb logcat -s AudioPolicyService:V AudioPolicyManager:V Engine:V

# 查看设备连接/断开
adb logcat | grep -E "setDeviceConnectionState|deviceConnectionState"

# 查看路由切换
adb logcat | grep -E "getOutputForAttr|getOutputForDevice|openOutput"

# 查看车载策略
adb logcat | grep -E "CarAudioService|CarAudioManager|audioPolicyEngine"
```

### 10.3 常见问题诊断

**问题1：插入耳机后声音没有切换**

```bash
# 检查设备是否被识别
adb shell dumpsys media.audio_policy | grep -A 10 "Available output devices"

# 检查路由是否触发
adb logcat | grep -E "setDeviceConnectionState|checkOutputsForDevice"

# 可能原因：
# 1. audio_policy_configuration.xml 中 route 配置缺失
# 2. 策略优先级列表中耳机不在扬声器前面
# 3. HAL 未正确上报设备连接事件
```

**问题2：通话时声音从扬声器出来而非听筒**

```bash
# 检查当前音频模式
adb shell dumpsys media.audio_policy | grep -i "mode\|phoneState"

# 检查 ForceUse 配置
adb shell dumpsys media.audio_policy | grep -A 5 "Force use"

# 可能原因：
# 1. setSpeakerphoneOn(true) 已被调用
# 2. ForceUse 被设置为 FORCE_SPEAKER
# 3. 蓝牙 SCO 未连接，回退到扬声器
```

**问题3：车载多音区路由异常**

```bash
# 检查车载音频配置
adb shell dumpsys car_service | grep -A 30 "audio"

# 检查各音区状态
adb shell dumpsys media.audio_policy | grep -E "zone|bus|device"

# 检查 CAP Engine 状态
adb shell dumpsys media.audio_policy | grep -A 10 "policy engine"
```

**问题4：蓝牙 A2DP 连接后音乐没有切换**

```bash
# 检查 A2DP 状态
adb shell dumpsys media.audio_policy | grep -A 5 "A2DP\|a2dp"

# 检查蓝牙设备是否在可用设备列表中
adb shell dumpsys media.audio_policy | grep "BLUETOOTH_A2DP"

# 可能原因：
# 1. A2DP 编解码协商未完成（mA2dpSuspended = true）
# 2. 通话中 A2DP 被挂起
# 3. 策略优先级配置问题
```

### 10.4 配置文件检查

```bash
# 查看当前生效的配置文件
adb shell cat /vendor/etc/audio_policy_configuration.xml
adb shell cat /vendor/etc/car_audio_configuration.xml

# 查看策略引擎配置
adb shell cat /system/etc/audio_policy_engine_configuration.xml
```

## 十一、关键源码路径

| 组件 | 路径 |
|-----|------|
| AudioPolicyService 服务 | `frameworks/av/services/audiopolicy/service/AudioPolicyService.cpp` |
| AudioPolicyManager 策略管理 | `frameworks/av/services/audiopolicy/managerdefault/AudioPolicyManager.cpp` |
| Engine 路由决策引擎 | `frameworks/av/services/audiopolicy/enginedefault/src/Engine.cpp` |
| EngineBase 基础引擎实现 | `frameworks/av/services/audiopolicy/enginedefault/src/EngineBase.cpp` |
| AudioPolicyMix 动态路由 | `frameworks/av/services/audiopolicy/managerdefault/AudioPolicyManager.cpp` |
| AudioCommandThread 异步命令 | `frameworks/av/services/audiopolicy/service/AudioPolicyService.cpp` |
| audioserver 启动入口 | `frameworks/av/media/audioserver/main_audioserver.cpp` |
| AudioFlinger 执行层 | `frameworks/av/services/audioflinger/AudioFlinger.cpp` |
| AudioManager 客户端 API | `frameworks/base/media/java/android/media/AudioManager.java` |
| AudioService Java 层服务 | `frameworks/base/services/core/java/com/android/server/audio/AudioService.java` |
| CarAudioService 车载音频服务 | `packages/services/Car/service/src/com/android/car/audio/CarAudioService.java` |
| 策略主配置文件 | `/vendor/etc/audio_policy_configuration.xml` |
| 引擎策略配置 | `/system/etc/audio_policy_engine_configuration.xml` |
| 车载音频配置 | `/vendor/etc/car_audio_configuration.xml` |

## 十二、与现有技能的关系

| 维度 | 本技能 | android-car-audio-focus | android-audioflinger-architecture |
|-----|--------|------------------------|-----------------------------------|
| 视角 | 策略路由决策 | 焦点仲裁管理 | 宏观架构 |
| 重点 | 设备选择、路由策略、配置、模式切换 | 焦点栈、延迟焦点、Duck | 四层架构、线程类型、混音 |
| 适用场景 | 路由异常、设备选择错误、配置问题 | 多音源冲突、焦点抢占 | 理解整体设计、定位架构问题 |

## 十三、总结

本技能涵盖了 Android 15 车载 AudioPolicyService 策略管理机制，核心要点：

1. **策略与执行分离**：AudioPolicyService（决策）与 AudioFlinger（执行）通过 `IoHandle` 解耦，各司其职
2. **三层配置体系**：`audio_policy_configuration.xml`（拓扑）+ `audio_policy_engine_configuration.xml`（规则）+ `car_audio_configuration.xml`（车载）
3. **策略映射链路**：AudioAttributes → ProductStrategy → DevicePriorityList → FinalDevice
4. **三因素决策**：应用指定设备 > 音频模式（Mode）> 策略优先级列表
5. **设备 Failover**：断开时自动切换到下一个优先级最高的可用设备
6. **ForceUse 强制路由**：可覆盖默认策略，适合免提、车载模式等场景
7. **Android 15 新特性**：AIDL HAL 全面替代 HIDL、音量组与路由深度绑定、CAP Engine 可配置策略引擎
8. **车载特有**：多音区策略路由、`CarAudioService` 协同、`car_audio_configuration.xml` 独立配置

理解 AudioPolicyService 的策略管理，是处理车载音频路由异常、设备选择错误、多音区配置问题的核心基础。

*本文基于 Android 15 (API Level 35) 源码分析，不同厂商的定制 ROM 可能存在差异。车载音频策略的具体实现可能因 OEM 定制而异。*